# Chapter 5 — Pods

## What it is

A **Pod** is the smallest deployable unit in Kubernetes — not a
container, a **Pod**. A Pod wraps one or more containers that are
guaranteed to be scheduled onto the same node, share the same network
namespace (same IP address and port space), and optionally share storage
volumes. Most Pods hold exactly one container; multi-container Pods are a
deliberate, specific pattern (sidecars), not the common case.

## Why it exists

You might reasonably ask: why not just schedule containers directly, the
way Docker does? Because some things genuinely need to be co-located and
share fate: a "sidecar" log-shipper that must read the same log file the
main container writes, or a service-mesh proxy that must intercept the
main container's network traffic. Kubernetes needed *some* unit that
groups containers with shared network/storage but keeps them separately
defined (separate images, separate resource limits, separate restart
behavior) — that unit is the Pod. It also gives Kubernetes one stable
"thing" to attach an IP address, a set of labels, and a lifecycle to,
regardless of how many containers are inside.

## When to use it

You'll almost never create bare Pods directly in production — you'll use
a ReplicaSet/Deployment (Chapters 7-8) that manages Pods for you. But
every higher-level object is *ultimately* just a template stamping out
Pods, so understanding the Pod spec itself is foundational to everything
after this chapter. Bare Pods (no controller) are appropriate only for
true one-off debugging containers you'll delete by hand.

## Internal architecture

- Every container in a Pod shares one **network namespace**: same IP,
  `localhost` reaches every other container in the Pod on whatever port
  it's listening on. This is implemented via a hidden **pause container**
  (technically the "infra container") that every Pod actually starts
  first — it does nothing but hold the network namespace open so your
  real containers can join it, even if your containers crash and restart.
- Containers in a Pod can share **volumes** (Chapter 14 goes deep on
  persistent ones; Chapter 5 only needs `emptyDir`, a simple shared scratch
  directory that exists for the Pod's lifetime).
- A Pod is scheduled **as a whole, atomically**, onto exactly one node —
  you cannot have container A on Node 1 and container B (of the same Pod)
  on Node 2. This is the direct consequence of "shares a network
  namespace" — that's only possible on one machine.
- A Pod's IP address is **ephemeral** — if the Pod is deleted and
  recreated (even by a controller replacing a "crashed" one), it gets a
  *new* IP. This is precisely the problem Services (Chapter 9) exist to
  paper over.

## YAML Definition

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: whoami
  labels:
    app: whoami
spec:
  containers:
    - name: whoami
      image: traefik/whoami
      ports:
        - containerPort: 80
      resources:
        requests:
          cpu: "100m"
          memory: "64Mi"
        limits:
          cpu: "200m"
          memory: "128Mi"
```

- `apiVersion: v1` — Pod is one of the oldest, most stable object types,
  so it lives in the "core" API group, versioned simply `v1` (no group
  prefix, unlike e.g. `apps/v1` for Deployments).
- `kind: Pod` — tells the API server which schema/controller this object
  belongs to.
- `metadata.name` — the Pod's unique name within its namespace.
- `metadata.labels` — arbitrary key/value tags used for selection
  (Chapter 10) — critical even for a lone Pod, since Services find Pods
  purely by label matching, never by name.
- `spec.containers` — a **list**, because a Pod can hold more than one.
  Each entry needs its own `name` (for `kubectl logs -c <name>` and
  `exec -c <name>` when there's more than one) and `image`.
- `ports.containerPort` — purely documentation/introspection at the Pod
  level (unlike Docker's `-p`, this does **not** publish anything to your
  host — it's informational metadata other tools/humans read); actual
  exposure to the outside world is a Service's job (Chapter 9).
- `resources.requests` — the minimum CPU/memory the scheduler guarantees
  this container when picking a node (full depth in Chapter 22).
- `resources.limits` — the hard ceiling the container runtime enforces at
  runtime (exceeding memory limit gets the container OOM-killed;
  exceeding CPU limit gets it throttled, not killed).

## Hands-on Example

```bash
kubectl apply -f whoami-pod.yaml
kubectl get pods -o wide
kubectl describe pod whoami
kubectl logs whoami
kubectl exec -it whoami -- sh
```
Inside the shell, `hostname` will print `whoami` (the Pod name) —
confirming, from the inside, that a Pod behaves like a tiny standalone
host with its own hostname and IP, exactly as the "shared network
namespace" description promised.

Now watch a Pod's IP change on recreation — the core lesson of this
chapter:
```bash
kubectl get pod whoami -o jsonpath='{.status.podIP}'
kubectl delete pod whoami
kubectl apply -f whoami-pod.yaml
kubectl get pod whoami -o jsonpath='{.status.podIP}'
```
Different IP. Nothing in Kubernetes updated any client that had cached the
old one — this is precisely why nothing production-grade ever talks to a
Pod IP directly.

**Multi-container Pod — a genuine sidecar, sharing `emptyDir`:**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: log-sidecar-demo
spec:
  containers:
    - name: writer
      image: busybox
      command: ["sh", "-c", "while true; do echo $(date) >> /var/log/app.log; sleep 2; done"]
      volumeMounts:
        - name: shared-logs
          mountPath: /var/log
    - name: reader
      image: busybox
      command: ["sh", "-c", "tail -f /var/log/app.log"]
      volumeMounts:
        - name: shared-logs
          mountPath: /var/log
  volumes:
    - name: shared-logs
      emptyDir: {}
```
`volumes` (Pod-level) declares the shared scratch space; `volumeMounts`
(per-container) says where each container sees it in its own filesystem.
Two entirely separate images/processes, sharing files through a common
directory, scheduled together as one atomic unit:
```bash
kubectl apply -f log-sidecar-demo.yaml
kubectl logs log-sidecar-demo -c reader -f
```
You'll see the `writer` container's output appear in the `reader`
container's tail — proof the volume is genuinely shared.

Cleanup:
```bash
kubectl delete -f whoami-pod.yaml -f log-sidecar-demo.yaml
```

## Debugging Techniques

- `kubectl describe pod <name>` — always your first move. The **Events**
  section at the bottom explains scheduling failures, image pull errors,
  probe failures — in plain English, in chronological order.
- `kubectl logs <name>` — stdout/stderr of the main process. Add
  `--previous` to see the logs of a container that already crashed and
  was restarted (critical — by the time you notice a `CrashLoopBackOff`,
  the *current* container instance may have no logs yet).
- `kubectl get pod <name> -o yaml` — shows `status.conditions` and
  `status.containerStatuses`, including the exact `reason` for the current
  state (`ImagePullBackOff`, `CrashLoopBackOff`, `OOMKilled`, etc.) — more
  precise than `describe`'s summarized events for exact reason codes.
- **`Pending`** → scheduling problem, check `describe` events (Chapter 2's
  scheduler at work).
- **`ImagePullBackOff` / `ErrImagePull`** → wrong image name/tag, private
  registry auth missing (image pull secrets — related to Chapter 13), or
  no network egress from the node.
- **`CrashLoopBackOff`** → the container starts and exits repeatedly;
  check `logs --previous` for why the process itself is dying — this is
  almost never a Kubernetes problem, it's your application/image.
- **`OOMKilled`** (seen in `describe`'s last-state reason) → the container
  exceeded its memory `limit` (Chapter 22) and the kernel's OOM killer
  ended it — raise the limit or fix a memory leak, don't just restart it.

## Best Practices

- Never deploy a bare Pod in production — always go through a Deployment
  (or Job/StatefulSet/DaemonSet, depending on the workload shape) so a
  controller replaces it automatically on failure.
- Always set `resources.requests` and `resources.limits` (Chapter 22
  covers the reasoning fully) — an unbounded Pod can starve its
  neighbors on the same node.
- Keep one container = one responsibility. Reach for multi-container Pods
  only for genuine sidecars (log shippers, service-mesh proxies,
  config-reloaders) — not as a substitute for separate Deployments.
- Never write application code that assumes a Pod's IP is stable across
  restarts — always talk to a Service (Chapter 9) or DNS name instead.

## Interview Questions

1. **Why is a Pod, not a container, the smallest unit in Kubernetes?**
   Because Kubernetes needed a unit that can group containers sharing
   network/storage while still scheduling, networking, and labeling them
   as one atomic thing — necessary for sidecar patterns and for having a
   stable scheduling/IP/label anchor regardless of container count.
2. **How do containers within the same Pod communicate?**
   Via `localhost` — they share one network namespace, held open by the
   hidden pause/infra container, so ports are just localhost ports to each
   other.
3. **Why can't a Pod's containers be scheduled onto different nodes?**
   Because sharing a network namespace is a single-host-kernel feature;
   there's no way to share one IP/port space across a network boundary the
   way the Pod's networking guarantee requires.
4. **What happens to a Pod's IP address when it's deleted and recreated?**
   It changes — Pod IPs are ephemeral and are never guaranteed stable,
   which is exactly why Services (stable virtual IPs) exist.
5. **What's the difference between `resources.requests` and
   `resources.limits`?**
   Requests are what the scheduler guarantees is available on the chosen
   node (used for placement decisions); limits are the hard ceiling the
   runtime enforces at runtime (memory over-limit = OOM kill, CPU
   over-limit = throttling, not killing).

## Mini Assignment

Create a two-container Pod where one container is a simple web server
(`traefik/whoami` again works) and the second container periodically
`curl`s `localhost` (against the first container's port) and logs the
response. This proves, hands-on, that Pod-mates truly share a network
namespace — no Service, no DNS, needed between them. Debug it using only
`describe`, `logs -c <container>`, and `exec -c <container>` — no
solution given here.

## Lesson Summary

- A Pod, not a container, is Kubernetes' smallest deployable unit — one or
  more containers guaranteed to share a node, a network namespace, and
  optionally volumes.
- Containers in a Pod talk to each other over `localhost`; the Pod itself
  gets one IP, which is ephemeral and changes on every recreation.
- Bare Pods are for debugging only — real workloads always go through a
  controller (starting Chapter 7) that recreates Pods automatically.
- `describe`, `logs [--previous] [-c container]`, and `get -o yaml` are
  your three core debugging tools, and you'll use them in every remaining
  chapter.

---

### Before Chapter 6 (YAML for Kubernetes) — tell me:

1. Why does a multi-container Pod's `writer`/`reader` sidecar example
   need an `emptyDir` volume specifically, rather than just two separate
   Pods?
2. What's the very first thing you check when a Pod is stuck `Pending`?
   What about `CrashLoopBackOff`?
3. Confirm: did your Pod's IP actually change when you deleted and
   recreated it in the hands-on lab?
