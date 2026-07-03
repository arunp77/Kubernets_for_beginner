# Chapter 12 — ConfigMaps

## What it is

A **ConfigMap** is a Kubernetes object holding non-sensitive configuration
data as key/value pairs, which can be injected into Pods as environment
variables, individual files, or a whole mounted directory of files —
decoupled entirely from the container image itself.

## Why it exists

You already do this with Docker via `.env` files and Compose's
`environment:` block, but that config lives *with the host running
Compose* — it's not a first-class, cluster-managed, independently
versionable object. A ConfigMap makes configuration a real Kubernetes
object: you can update it without rebuilding an image, reference the same
ConfigMap from multiple Deployments, and (critically) change config
without necessarily needing a full image-level rollout — though, as
you'll see below, actually *picking up* a change still has real nuance.

## When to use it

Any non-secret configuration that should be separate from your image:
feature flags, external service URLs, log levels, `.ini`/`.conf`/`.json`
config files an application reads at startup. For anything sensitive
(passwords, API keys, tokens), use a **Secret** instead (Chapter 13) —
mechanically almost identical, but with different at-rest handling and
intent.

## Internal architecture

- A ConfigMap's data is stored in etcd exactly like any other object — in
  the unencrypted-by-default case, in plain text (this is the core reason
  Secrets, Chapter 13, exist as a *distinct* type with different
  handling, even though the mechanics of consuming them are nearly
  identical).
- When mounted as **environment variables**, the values are injected at
  **container start time only** — the kubelet resolves them once when
  creating the container process's environment. If you update the
  ConfigMap afterward, already-running Pods' environment variables do
  **not** change; you must trigger a new Pod (typically via a rollout,
  Chapter 8) to pick up the new values.
- When mounted as a **volume** (a directory of files, one file per key),
  the kubelet **does** periodically sync updates to the mounted files on
  a live Pod (usually within about a minute, subject to kubelet sync
  period) — but your application still needs to *notice* the file changed
  and reload it; Kubernetes doesn't restart your process or signal it for
  you. This env-var-vs-volume distinction is one of the most
  interview-tested and practically important facts in this chapter.

## YAML Definition

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: whoami-config
data:
  LOG_LEVEL: "debug"
  GREETING: "hello from ConfigMap"
  app.conf: |
    server {
      listen 80;
      log_level debug;
    }
```
- `data` — a flat map of string keys to string values. Simple values
  (`LOG_LEVEL`) are meant for env-var injection; multi-line values like
  `app.conf` (note the YAML block scalar `|`) are meant for file-mount
  injection, one key becoming one filename.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: whoami
spec:
  replicas: 2
  selector:
    matchLabels: { app: whoami }
  template:
    metadata:
      labels: { app: whoami }
    spec:
      containers:
        - name: whoami
          image: traefik/whoami
          envFrom:
            - configMapRef:
                name: whoami-config
          volumeMounts:
            - name: config-volume
              mountPath: /etc/whoami-config
      volumes:
        - name: config-volume
          configMap:
            name: whoami-config
```
- `envFrom.configMapRef` — injects **every** key in the ConfigMap as an
  environment variable in one shot (alternative: `env[].valueFrom.
  configMapKeyRef` to cherry-pick individual keys under custom variable
  names).
- `volumes[].configMap.name` — declares a Pod-level volume backed by the
  ConfigMap's contents.
- `volumeMounts` (per-container, same pattern as Chapter 5's `emptyDir`
  sidecar example) — mounts that volume at `/etc/whoami-config`; each key
  in `data` becomes a file there (`/etc/whoami-config/app.conf`,
  `/etc/whoami-config/LOG_LEVEL`, etc.).

## Hands-on Example

```bash
kubectl apply -f whoami-config.yaml
kubectl apply -f whoami-deploy.yaml
kubectl exec deploy/whoami -- env | grep -E 'LOG_LEVEL|GREETING'
kubectl exec deploy/whoami -- cat /etc/whoami-config/app.conf
```

**Prove the env-var vs. volume update behavior directly — the core lesson
here:**
```bash
kubectl patch configmap whoami-config --type merge -p '{"data":{"LOG_LEVEL":"trace"}}'

# env var: unchanged on the ALREADY-RUNNING pod
kubectl exec deploy/whoami -- env | grep LOG_LEVEL     # still "debug"

# mounted file: eventually updates on the SAME running pod, no restart
sleep 60
kubectl exec deploy/whoami -- cat /etc/whoami-config/LOG_LEVEL   # now "trace" (if you'd mounted it as a file)
```
(To actually see this for `LOG_LEVEL` as a file, add it to the
`volumeMounts` path too — try adding it yourself as an exercise before
Chapter 13, since Secrets share the exact same mechanic.)

**Force env vars to pick up the change, the way it's actually done in
practice — via a rollout, not a restart:**
```bash
kubectl rollout restart deployment/whoami
kubectl rollout status deployment/whoami
kubectl exec deploy/whoami -- env | grep LOG_LEVEL      # now "trace"
```
`rollout restart` is a genuinely useful command you haven't seen yet: it
triggers a normal Chapter-8 rolling update with no spec change at all
(just a timestamp annotation bump), purely to force fresh Pods that
re-read ConfigMap/Secret env vars at startup.

Cleanup:
```bash
kubectl delete -f whoami-deploy.yaml -f whoami-config.yaml
```

## Debugging Techniques

- **Changed a ConfigMap, app still behaves with old config** — check
  *how* it's consumed: env vars never live-update (expected — trigger a
  rollout), mounted files update but only if your app re-reads the file
  from disk rather than caching it in memory at startup.
- **`CreateContainerConfigError` on a Pod** — almost always means a
  Deployment references a ConfigMap (or key within it, via
  `configMapKeyRef`) that doesn't exist; `kubectl describe pod` names the
  missing ConfigMap/key exactly.
- **`kubectl exec ... -- cat <path>` shows nothing / wrong content** —
  double check `mountPath` isn't accidentally shadowing an existing
  directory the image itself populated (a mounted ConfigMap volume
  replaces the *entire* directory content at that path, not merges with
  it).

## Best Practices

- Use `envFrom`/volume mounts to keep application code free of any
  Kubernetes-awareness — your app should just read env vars or files, the
  same way it would locally or under Compose; don't couple app code to
  the Kubernetes API to fetch its own config.
- Prefer mounting config **files** for anything more complex than a
  handful of flat values (nginx configs, JSON/YAML config blobs) — env
  vars don't handle structured data or live updates gracefully.
- If you need config changes to reliably reach running Pods, don't rely
  on the ~60s volume sync timing for anything correctness-sensitive —
  design your rollout process (Chapter 29's CI/CD) to bump a
  Deployment annotation (or literally change the image tag) whenever
  config changes, forcing a clean, observable rollout.
- Never put secrets in a ConfigMap "just this once" — even though the
  mechanics look identical, ConfigMap data is stored and often displayed
  in plaintext; that's precisely the boundary Chapter 13 exists to draw.

## Interview Questions

1. **What's the difference between injecting a ConfigMap as environment
   variables versus as a mounted volume, in terms of update behavior?**
   Env vars are resolved once at container start and never update on a
   running Pod; mounted-volume files are periodically synced to reflect
   ConfigMap changes on already-running Pods (subject to kubelet sync
   timing), though the application must itself notice and reload the
   file.
2. **If you update a ConfigMap consumed via `envFrom`, how do you
   actually get running Pods to pick up the change?** Trigger a new
   rollout (e.g., `kubectl rollout restart deployment/<name>`) so fresh
   Pods are created and re-resolve the environment at startup.
3. **Why not just put everything, including secrets, in a ConfigMap?**
   ConfigMap data is stored and displayed as plain text with no special
   at-rest handling; Secrets exist as a distinct type specifically so
   sensitive data gets separate handling/conventions (Chapter 13),
   even though the consumption mechanics are nearly identical.
4. **What happens if a Deployment references a ConfigMap that doesn't
   exist?** The Pod fails to start with `CreateContainerConfigError`;
   `kubectl describe pod` names the missing ConfigMap/key.
5. **How is this different from Docker Compose's `.env`/`environment:`
   approach?** Compose's config lives with whatever host runs `docker
   compose up`; a ConfigMap is a first-class, independently versioned
   cluster object that multiple Deployments across the cluster can share
   and reference identically.

## Mini Assignment

Create a ConfigMap with at least one key meant for env-var injection and
one multi-line key meant for file-mount injection, wire both into a
Deployment, and — without looking anything up — predict in writing
whether each will update on a running Pod after you `kubectl patch` the
ConfigMap, before actually testing both. Compare your predictions against
what you observe.

## Lesson Summary

- ConfigMaps decouple non-secret configuration from container images,
  injectable as env vars (resolved once, at container start) or mounted
  files (synced live, but the app must notice and reload).
- This is a stricter, more Kubernetes-native version of Compose's `.env`/
  `environment:` pattern — a real, independently-versioned cluster object
  rather than host-local config.
- `kubectl rollout restart` is the standard way to force running Pods to
  pick up a ConfigMap change consumed via env vars.
- Never store secrets here — Chapter 13 covers the (mechanically similar,
  intent-different) object built for that.

---

### Before Chapter 13 (Secrets) — tell me:

1. Why doesn't updating a ConfigMap automatically update env vars on Pods
   that are already running?
2. What command would you run to force a Deployment's Pods to pick up a
   ConfigMap change, without changing anything else about the Deployment?
3. Did your predictions in the Mini Assignment match what you actually
   observed for the env-var key versus the file-mounted key?
