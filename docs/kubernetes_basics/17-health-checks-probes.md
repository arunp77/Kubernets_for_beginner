# Chapter 17 — Health Checks (Probes)

## What it is

A **probe** is a periodic check the kubelet performs against a
container to answer one of three distinct questions: is it **alive**
(liveness), is it **ready for traffic** (readiness), and has it
**finished starting up** (startup). These are genuinely three different
questions with three different consequences when they fail — conflating
them is the single most common probe mistake.

## Why it exists

You already met `readinessProbe` in passing in Chapter 8 — this chapter
gives it, and its two siblings, full treatment, because Kubernetes'
entire self-healing and rollout-safety story (Chapters 1, 7, 8) depends
on being able to answer "is this container actually working" more
precisely than "is the process still running." A container's *process*
can be alive while the *application* is deadlocked, stuck waiting on a
dependency forever, or still loading a large dataset at startup — plain
process-alive checking (the closest Docker equivalent, `HEALTHCHECK` in a
Dockerfile) can't distinguish these cases from each other, and each one
needs a different response.

## When to use it

Every production container should define at least a **readiness probe**
(so Services/rollouts know when it's truly safe to send traffic) and,
where meaningful, a **liveness probe** (so a genuinely stuck process gets
restarted automatically). Use a **startup probe** for anything with slow,
variable startup time (large JVMs, apps doing schema migrations on boot)
to avoid the other two probes misfiring during that window.

## Internal architecture — three probes, three consequences

- **`livenessProbe`** — "should this container be restarted?" On repeated
  failure, the kubelet **kills and restarts the container** (same Pod,
  same node — not a reschedule, just a container restart, tracked in the
  Pod's restart count). Use this only for "this process is definitely
  wedged and a restart is the correct fix" — an overly aggressive
  liveness probe (e.g., checking a slow downstream dependency) causes
  **restart storms**: you kill a container that was fine, the new one
  takes time to warm up, still can't reach the slow dependency, gets
  killed again — a self-inflicted outage. This is the single most common
  production probe misconfiguration.
- **`readinessProbe`** — "should this Pod receive traffic right now?" On
  failure, the Pod is **removed from Service Endpoints** (Chapter 9) —
  the container keeps running, nothing is restarted, it's simply taken
  out of rotation until it passes again. This is exactly the mechanism
  Chapter 8's rolling updates gate on, and the correct tool for "briefly
  overloaded, will recover" conditions that should reduce traffic, not
  trigger a restart.
- **`startupProbe`** — "has this container finished its (possibly slow,
  variable-length) startup sequence?" While a startup probe is defined and
  failing, **liveness and readiness probes are not executed at all** —
  this exists specifically so a slow-starting app doesn't get killed by
  an impatient liveness probe before it's even finished booting. Once the
  startup probe succeeds once, it stops being checked, and liveness/
  readiness take over normally for the rest of the container's life.
- Every probe supports three check mechanisms: **`httpGet`** (expects
  2xx/3xx), **`tcpSocket`** (just checks the port accepts a connection),
  and **`exec`** (runs a command inside the container, 0 exit code =
  success) — pick whichever matches what "healthy" actually means for
  your app; a TCP check only proves the port is open, not that the
  application behind it is functioning correctly.

## YAML Definition

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: whoami
  labels: { app: whoami }
spec:
  containers:
    - name: whoami
      image: traefik/whoami
      ports:
        - containerPort: 80
      startupProbe:
        httpGet:
          path: /
          port: 80
        failureThreshold: 30
        periodSeconds: 2
      livenessProbe:
        httpGet:
          path: /
          port: 80
        initialDelaySeconds: 0
        periodSeconds: 10
        failureThreshold: 3
      readinessProbe:
        httpGet:
          path: /
          port: 80
        periodSeconds: 5
        failureThreshold: 2
```
- `httpGet.path`/`port` — the endpoint checked; a real app should expose a
  dedicated health endpoint here (e.g., `/healthz`) rather than reusing a
  business endpoint, so probe traffic and probe semantics stay clearly
  separated from application logic.
- `initialDelaySeconds` — grace period after container start before the
  first check — a crude alternative to a proper `startupProbe`; prefer a
  real `startupProbe` for genuinely variable startup times, since a fixed
  delay is either too short (flaky) or too long (slows detection of real
  failures) by definition.
- `periodSeconds` — how often the check repeats.
- `failureThreshold` — consecutive failures needed before the probe is
  considered failed (and its consequence — restart or Endpoint removal —
  triggers). `startupProbe`'s `failureThreshold: 30` × `periodSeconds: 2`
  = up to 60 seconds allowed for startup before it's considered failed.
- Note liveness and readiness can (and often should) check the *same*
  endpoint with *different* thresholds/periods — they're independent
  configurations even when pointed at the same URL, because their
  consequences differ.

## Hands-on Example

```bash
kubectl apply -f whoami-probes.yaml
kubectl describe pod whoami | grep -A3 -E "Liveness|Readiness|Startup"
```

**Watch the startup-probe gate hold off the others** — restart the Pod
and immediately check its readiness gate:
```bash
kubectl get pod whoami -w
```
Notice `READY` stays `0/1` until the startup probe passes at least once,
even though liveness/readiness would each individually already be
passing — confirming they're genuinely gated behind it during startup.

**Force a readiness failure without killing the container** — point the
probe at a path that 404s, to see the "removed from Endpoints, not
restarted" behavior directly:
```bash
kubectl patch pod whoami --type='json' \
  -p='[{"op":"replace","path":"/spec/containers/0/readinessProbe/httpGet/path","value":"/does-not-exist"}]'
# (in practice, edit the YAML and re-apply, since most Pod fields including probes are immutable in-place —
#  use a Deployment + `kubectl edit deployment` for a realistic live version of this test)
kubectl get pod whoami -o wide     # READY becomes 0/1
kubectl get endpoints whoami-svc   # (if a Service selects it) — this Pod's IP disappears from the list
```
The container is still `Running` — `kubectl exec` into it still works —
only its traffic eligibility changed.

**Cause a genuine liveness failure and watch a real restart:**
```bash
kubectl exec whoami -- sh -c "kill 1"   # kill the main process directly
kubectl get pod whoami -w
```
Watch `RESTARTS` increment — this is the liveness consequence, distinct
from the readiness case above where nothing restarted.

**See a restart-storm risk directly** — set an unreasonably strict
liveness probe (very short timeout against a naturally slower response)
and watch the Pod enter `CrashLoopBackOff` purely from probe
misconfiguration, with the application itself never actually broken —
worth doing once so the failure mode is unmistakable before you see it in
a real incident.

Cleanup:
```bash
kubectl delete pod whoami
```

## Debugging Techniques

- **Pod `Running` but `0/1` Ready, never receiving traffic** — check
  readiness probe config and hit the exact same endpoint yourself via
  `kubectl exec ... -- curl` to see what it actually returns; almost
  always a probe pointed at the wrong path/port, or an app that hasn't
  actually finished initializing internal state despite the process being
  up.
- **High `RESTARTS` count, app "should be fine"** — check `kubectl
  describe pod` events for `Liveness probe failed` messages with the
  specific reason (timeout vs. non-2xx vs. connection refused); this is
  the classic restart-storm signature and usually means the liveness
  probe itself is miscalibrated (too strict timeout, checking a slow
  dependency), not that the app is actually broken.
- **Rollout (Chapter 8) never completes** — new Pods are `Running` but
  never pass readiness, so the Deployment can't consider them available;
  debug the readiness probe on the *new* ReplicaSet's Pods specifically.
- **`exec` probe with a slow command** — each probe execution consumes
  real resources on the node; an expensive `exec` probe running every few
  seconds across hundreds of Pods is a genuine, easy-to-overlook resource
  cost.

## Best Practices

- Give liveness and readiness **genuinely different semantics**, not the
  same check reused blindly: liveness should only fail for "this process
  is definitely stuck, restart is the fix" conditions; readiness should
  fail for "temporarily can't serve traffic, will likely recover" —
  including downstream dependency issues, which belong in readiness, not
  liveness.
- Never point a liveness probe at a check that depends on external
  systems (a database, another microservice) — a downstream outage would
  then cause *your* container to restart-loop pointlessly, adding load to
  an already-struggling dependency instead of just backing off.
- Use a `startupProbe` (not just a generous `initialDelaySeconds`) for any
  workload with meaningfully variable startup time — it correctly avoids
  the tradeoff between "too-short delay causes flakiness" and
  "too-long delay slows real-failure detection."
- Give the readiness probe a dedicated, lightweight endpoint separate from
  business logic — don't reuse a heavy or side-effect-having endpoint
  purely because it happened to exist.

## Interview Questions

1. **What's the difference between a liveness and a readiness probe, in
   terms of consequence?** Liveness failure restarts the container
   (same Pod); readiness failure removes the Pod from Service Endpoints
   without restarting anything — traffic stops, the container keeps
   running.
2. **What is a restart storm, and how do liveness probes cause one?**
   A liveness probe too strict or dependent on a flaky/slow external
   system fails repeatedly, causing continuous unnecessary restarts of an
   otherwise-healthy container — each restart resets warm-up state,
   making the next check likely to fail too, in a self-sustaining loop.
3. **Why does a startupProbe exist when initialDelaySeconds already
   provides a grace period?** A fixed delay is a guess that's either too
   short (flaky failures during genuinely variable startup) or too long
   (masks real failures for longer than necessary); a startupProbe
   actively checks and only hands off to liveness/readiness once startup
   is genuinely confirmed complete.
4. **Should a liveness probe check a downstream dependency (e.g., a
   database connection)?** No — that belongs in a readiness probe. A
   liveness failure restarts your container, which does nothing to fix a
   downstream outage and can add load to an already-struggling
   dependency; readiness correctly just stops sending it traffic instead.
5. **What are the three probe check mechanisms, and when would `exec`
   be preferred over `httpGet`?** `httpGet`, `tcpSocket`, `exec` — use
   `exec` when health depends on something not exposable over HTTP (e.g.,
   checking a local file, a CLI health tool, a non-HTTP protocol's own
   health command).

## Mini Assignment

Configure one Deployment with a liveness probe that checks a downstream
dependency it doesn't actually have (point it at an endpoint that will
sometimes fail) and observe a restart storm develop over a few minutes of
`kubectl get pods -w`. Then fix it by moving that exact check to a
readiness probe instead, leaving liveness pointed at a simple, dependency
-free check, and confirm the restart count stops climbing while readiness
correctly flaps instead.

## Lesson Summary

- Liveness, readiness, and startup probes answer three distinct
  questions with three distinct consequences: restart, remove-from
  -traffic, and gate-the-other-two-probes, respectively.
- Conflating liveness with readiness — especially checking downstream
  dependencies in a liveness probe — is the single most common
  production probe mistake, and it causes self-inflicted restart storms.
- `startupProbe` correctly handles variable/slow startup without the
  flakiness-vs-slow-detection tradeoff a fixed `initialDelaySeconds`
  forces you into.
- Probes are what let Deployments' rolling updates (Chapter 8) and
  Services' Endpoints (Chapter 9) actually know what "healthy" means for
  your specific application.

---

### Before Chapter 18 (Jobs) — tell me:

1. If a Pod fails its readiness probe, does Kubernetes restart the
   container? What actually happens instead?
2. Why is checking a downstream database connection in a liveness probe
   specifically dangerous?
3. Did you observe the restart storm in the Mini Assignment, and did
   moving the check to readiness actually stop it?
