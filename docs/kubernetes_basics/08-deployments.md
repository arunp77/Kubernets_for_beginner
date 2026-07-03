# Chapter 8 — Deployments

## What it is

A **Deployment** is a controller that manages ReplicaSets on your behalf,
adding the one thing Chapter 7 proved ReplicaSets deliberately lack:
**safe, versioned rollout of template changes** to already-running Pods —
plus rollback history. In practice, a Deployment is the object you will
create for almost every stateless workload in this entire course.

## Why it exists

Remember Lesson 1's hands-on lab: hand-rolling a zero-downtime rollout
with plain Docker required you to manually bring up a new container,
verify it was healthy, and only then remove the old one — for one
container. A Deployment automates exactly that choreography, correctly,
for any number of replicas, with a strategy you configure once. It does
this by **creating a brand-new ReplicaSet** for the new Pod template,
scaling it up while scaling the old ReplicaSet down, at a pace you
control — and it remembers every ReplicaSet it's ever created, giving you
instant rollback to any previous version.

## When to use it

Any stateless, horizontally-scalable workload where Pods are
interchangeable and don't need stable identity or ordered
startup/shutdown (your FastAPI backend, a worker pool, a frontend). If
Pods need stable network identity or ordered scaling (databases), you
want a **StatefulSet** instead (Chapter 21).

## Internal architecture

- A Deployment doesn't manage Pods directly — it manages **ReplicaSets**,
  which manage Pods. This is the layering you were promised in Chapter 7:
  Deployment → owns → ReplicaSet → owns → Pods, each link tracked via
  `ownerReference`.
- When you change `spec.template` (e.g. a new image tag), the Deployment
  controller creates a **new ReplicaSet** with the new template, and
  begins the configured rollout **strategy** — by default `RollingUpdate`:
  gradually scale the new ReplicaSet up and the old one down, respecting
  `maxSurge` (how many extra Pods above `replicas` are allowed
  temporarily) and `maxUnavailable` (how many Pods below `replicas` are
  tolerated temporarily) — both configurable, both defaulting to 25%.
- Crucially, the new ReplicaSet only scales up Pods that pass their
  **readiness probe** (Chapter 17) — a Pod that's merely "started" isn't
  counted as available for cutover purposes. This is the automated
  version of the manual health check you did by hand in Lesson 1's lab.
- The **old ReplicaSet is not deleted** after a successful rollout — it's
  scaled to 0 and kept (up to `revisionHistoryLimit`, default 10), which
  is exactly what makes `kubectl rollout undo` instant: rolling back is
  just re-running the same scale-up/scale-down dance against a ReplicaSet
  that already exists, rather than recreating anything from scratch.

## YAML Definition

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: whoami
  labels:
    app: whoami
spec:
  replicas: 3
  selector:
    matchLabels:
      app: whoami
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    metadata:
      labels:
        app: whoami
    spec:
      containers:
        - name: whoami
          image: traefik/whoami:v1.10
          ports:
            - containerPort: 80
          readinessProbe:
            httpGet:
              path: /
              port: 80
            initialDelaySeconds: 2
            periodSeconds: 5
```

- `spec.replicas` / `spec.selector` / `spec.template` — identical in shape
  and meaning to a ReplicaSet (Chapter 7) — because underneath, a
  Deployment literally generates a ReplicaSet spec from these exact
  fields.
- `spec.strategy.type` — `RollingUpdate` (default, gradual/zero-downtime)
  or `Recreate` (kill all old Pods, then create all new ones — simpler,
  but causes downtime; appropriate only when old and new versions truly
  cannot coexist, e.g. incompatible shared-volume schema).
- `maxSurge: 1` — during rollout, allow up to 1 Pod *above* the desired
  `replicas` count temporarily (so with `replicas: 3`, up to 4 Pods may
  exist mid-rollout).
- `maxUnavailable: 0` — during rollout, never let the ready Pod count drop
  below `replicas` — combined with `maxSurge: 1`, this guarantees true
  zero-downtime: new Pods must become ready *before* any old ones are
  removed.
- `readinessProbe` — previewed here, full depth in Chapter 17 — this is
  the mechanism the rollout uses to decide "is this new Pod actually safe
  to receive traffic," replacing the manual `curl` health check you did by
  hand in Lesson 1.

## Hands-on Example — redo Lesson 1's manual rollout, automated

```bash
kubectl apply -f whoami-deploy.yaml
kubectl get deploy,rs,pods -l app=whoami
```
Notice a ReplicaSet was created *for* you, named `whoami-<hash>` — the
hash is derived from the Pod template's content, so a genuinely new
template always produces a genuinely new ReplicaSet name.

**Trigger a rollout** by changing the image — this is the moment that took
you five manual, error-prone steps in Lesson 1:
```bash
kubectl set image deployment/whoami whoami=traefik/whoami:v1.10.4
kubectl rollout status deployment/whoami
```
`rollout status` blocks and streams progress until the rollout completes
or fails — watch it report Pods becoming ready one at a time, respecting
`maxSurge`/`maxUnavailable`.

```bash
kubectl get rs -l app=whoami
```
Two ReplicaSets now exist: the new one at `replicas: 3`, the old one
scaled to `replicas: 0` — not deleted, kept for rollback.

**Watch it happen live** in a second terminal while triggering another
change:
```bash
kubectl get pods -l app=whoami -w
```

**Roll back instantly:**
```bash
kubectl rollout history deployment/whoami
kubectl rollout undo deployment/whoami
kubectl rollout status deployment/whoami
```
`rollout undo` re-activates the previous ReplicaSet — no image pull, no
rebuild, because that ReplicaSet (and the Pods it can recreate) already
existed the whole time, merely scaled to zero.

**Deliberately break a rollout** to see the safety net work:
```bash
kubectl set image deployment/whoami whoami=traefik/whoami:this-tag-does-not-exist
kubectl rollout status deployment/whoami --timeout=30s
```
The new ReplicaSet's Pods sit `ImagePullBackOff` and never become ready —
`maxUnavailable: 0` means the old, working ReplicaSet is *never* scaled
down, since the new Pods never satisfy the readiness gate. Your
application stays fully available on the old version the entire time.
Fix it:
```bash
kubectl rollout undo deployment/whoami
```

Cleanup:
```bash
kubectl delete -f whoami-deploy.yaml
```

## Debugging Techniques

- `kubectl rollout status deployment/<name>` — the single best command for
  "is my rollout stuck, and why," blocking until success/failure/timeout.
- `kubectl describe deployment <name>` — shows the `Conditions` section
  (`Progressing`, `Available`) with human-readable reasons when a rollout
  stalls.
- `kubectl rollout history deployment/<name> --revision=N` — inspect the
  exact Pod template of any prior revision, useful when you're not sure
  what changed between two rollouts.
- **Stuck rollout with `maxUnavailable: 0`** — check the new ReplicaSet's
  Pods directly (`kubectl describe pod`) for the actual failure
  (`ImagePullBackOff`, failing readiness probe, `CrashLoopBackOff`) — the
  Deployment object itself will only tell you *that* it's stuck, not
  *why* at the Pod level.
- **Rollout "succeeded" but the app is actually broken** — means your
  readiness probe (Chapter 17) is too permissive (e.g., checking a `/`
  endpoint that returns 200 even when a deeper dependency is broken) —
  the Deployment can only be as good a gatekeeper as the probe you give
  it.

## Best Practices

- Always set an explicit `readinessProbe` — without one, Kubernetes
  considers a Pod "ready" the instant its container process starts, which
  defeats the entire purpose of a gated rolling update.
- Prefer `maxUnavailable: 0` for user-facing services where any capacity
  dip is unacceptable; accept `maxUnavailable: 1` (allowing brief reduced
  capacity) for internal/batch-tolerant workloads where faster rollouts
  matter more.
- Keep `revisionHistoryLimit` reasonable (default 10 is usually fine) —
  it's your rollback safety net, but each retained ReplicaSet has a real
  (small) cost.
- Never edit a running Deployment's Pods directly — always change
  `spec.template` and let the Deployment orchestrate the rollout; direct
  Pod edits get silently overwritten on the next reconciliation.

## Interview Questions

1. **What's the relationship between a Deployment, a ReplicaSet, and a
   Pod?** Deployment manages ReplicaSets (creating a new one per unique
   Pod template); ReplicaSets manage Pods (enforcing count). Deployment
   adds versioned rollout/rollback on top of ReplicaSet's proven
   count-enforcement.
2. **How does a rolling update achieve zero downtime?**
   It scales up a new ReplicaSet and scales down the old one gradually,
   gated by `maxSurge`/`maxUnavailable` and by each new Pod passing its
   readiness probe before being counted as available — old capacity is
   only removed after equivalent new capacity is confirmed healthy.
3. **How does `kubectl rollout undo` work so fast?**
   The previous ReplicaSet was never deleted, only scaled to zero — undo
   just re-triggers the same scale-up/scale-down dance in reverse, with no
   need to recreate or re-pull anything.
4. **What's the difference between `RollingUpdate` and `Recreate`
   strategies?** RollingUpdate gradually replaces Pods with zero (or
   minimal) downtime; Recreate kills all old Pods before creating any new
   ones, causing a full outage during the switch — used only when old and
   new versions genuinely cannot coexist.
5. **If a rollout is stuck, where do you look first?**
   `kubectl rollout status` / `describe deployment` for the high-level
   signal, then `describe`/`logs` on the *new* ReplicaSet's Pods
   specifically, since that's almost always where the actual failure
   (image, crash, failing probe) is visible.

## Mini Assignment

Configure a Deployment with `replicas: 4`, `maxSurge: 2`,
`maxUnavailable: 1`, and a readiness probe with a deliberately long
`initialDelaySeconds` (say, 15s). Trigger a rollout and, in a second
terminal running `kubectl get pods -l app=<x> -w`, count exactly how many
Pods of each version exist at each moment during the rollout. Reconcile
what you observe against the `maxSurge`/`maxUnavailable` math yourself
before reading any explanation elsewhere.

## Lesson Summary

- A Deployment adds versioned, gated rollout/rollback on top of
  ReplicaSet's count-enforcement — creating a new ReplicaSet per unique
  Pod template and shifting traffic between old and new gradually.
- `maxSurge`/`maxUnavailable` control the pace and safety margin of a
  rollout; a readiness probe is the gate that decides when a new Pod
  actually counts as available.
- Old ReplicaSets are kept, scaled to zero, purely to make `rollout undo`
  instant.
- This is the automated version of the manual, error-prone process you
  did by hand with plain Docker in Lesson 1 — and it's the object type
  you'll use for nearly every workload in the rest of this course.

---

### Before Chapter 9 (Services) — tell me:

1. Why is a readiness probe mandatory, in practice, for a rolling update
   to actually protect you?
2. Walk through, in your own words, why `rollout undo` doesn't need to
   pull any images or recreate anything from scratch.
3. From the Mini Assignment: did the Pod counts you observed mid-rollout
   match what `maxSurge`/`maxUnavailable` predicted?
