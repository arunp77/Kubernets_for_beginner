# Chapter 10 — Labels and Selectors

## What it is

**Labels** are arbitrary key/value strings attached to any object's
`metadata.labels`. **Selectors** are queries against those labels —
`matchLabels` (exact key/value match) or `matchExpressions` (richer
operators: `In`, `NotIn`, `Exists`, `DoesNotExist`). By this chapter you've
already used both, three times, without a dedicated explanation: a
ReplicaSet's `selector` (Ch. 7), a Deployment's `selector` (Ch. 8), and a
Service's `selector` (Ch. 9). This chapter makes that pattern explicit.

## Why it exists

Kubernetes deliberately does **not** let objects reference each other by
name for grouping purposes (e.g., a Service does not say "route to
Deployment named X"). It uses loose, label-based coupling instead. This is
a genuine architectural choice: it means a Service doesn't care *how* its
target Pods came to exist — a Deployment, a bare ReplicaSet, a hand-run
Pod, even Pods from two different Deployments (as you proved in Chapter
9's Mini Assignment) — as long as the labels match. This decoupling is
what lets you, e.g., point a Service at either a "blue" or "green"
Deployment during a blue/green release just by changing one label
selector, without touching either Deployment.

## When to use it

Constantly, and mostly implicitly — every object with a `selector` field
(ReplicaSet, Deployment, Service, NetworkPolicy in Ch. 25, HPA in Ch. 23)
relies on this mechanism. The deliberate design decision you make is
**what labeling scheme to standardize on** for your cluster — this
chapter is really about that discipline.

## Internal architecture

- Labels live in `metadata.labels` on *any* object — Pods, Nodes,
  Namespaces, everything. They have no meaning to the API server itself
  beyond being indexed strings available for selector queries — nothing
  is "special" about `app` or `tier` as label keys; those are just
  strong community conventions (part of the recommended labels
  documented by Kubernetes itself: `app.kubernetes.io/name`,
  `app.kubernetes.io/instance`, etc.).
- `matchLabels` is sugar for an `In` `matchExpressions` with one value —
  both compile down to the same underlying query the API server runs
  against its label index when listing/watching objects.
- Selectors are **evaluated live, continuously** — not once at creation
  time. This is precisely why Chapter 9's Mini Assignment worked: the
  moment any Pod's labels satisfy a Service's selector, it becomes a valid
  Endpoint on the *next* reconciliation tick, with no re-creation of the
  Service needed.
- **Selector immutability**: a Deployment's `spec.selector` is immutable
  after creation (you cannot repoint an existing Deployment to select
  different Pods) — this is a deliberate safety rail, since changing it
  could orphan or "steal" running Pods; Service selectors, by contrast,
  *are* mutable (you did this yourself with `kubectl patch` in Chapter 9),
  which is exactly what makes blue/green cutover-by-relabeling possible.

## YAML Definition — labels vs. selector expressions

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: demo
  labels:
    app: whoami
    tier: backend
    env: staging
spec:
  containers:
    - name: whoami
      image: traefik/whoami
```
```yaml
# a selector using richer matchExpressions instead of matchLabels
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: whoami-rs
spec:
  replicas: 2
  selector:
    matchExpressions:
      - key: tier
        operator: In
        values: ["backend"]
      - key: env
        operator: NotIn
        values: ["prod"]
  template:
    metadata:
      labels:
        app: whoami
        tier: backend
        env: staging
    spec:
      containers:
        - name: whoami
          image: traefik/whoami
```
- `metadata.labels` on the Pod — free-form key/values; `app`/`tier`/`env`
  here are just a convention, not built-in fields.
- `spec.selector.matchExpressions` — a list of conditions, all **ANDed**
  together: this selector matches Pods with `tier=backend` **and**
  `env` not in `["prod"]`. `matchLabels` and `matchExpressions` can be
  combined in the same selector; combining them is itself an AND.
- Note the template's labels must still satisfy the selector (Chapter 7's
  subset rule) — `tier: backend, env: staging` satisfies both conditions
  above.

## Hands-on Example

```bash
kubectl apply -f demo-pod.yaml
kubectl get pods --show-labels
```

**Query by label directly — you'll use this constantly for debugging:**
```bash
kubectl get pods -l app=whoami
kubectl get pods -l 'tier in (backend,frontend)'
kubectl get pods -l 'env!=prod'
kubectl get pods -l app=whoami,tier=backend   # comma = AND
```

**Add/remove labels on a live object without recreating it:**
```bash
kubectl label pod demo release=canary
kubectl get pods -l release=canary
kubectl label pod demo release-              # trailing dash removes it
```

**Blue/green cutover by relabeling a Service selector, not the
Deployments:**
```bash
kubectl run blue --image=traefik/whoami --labels="app=demo,version=blue"
kubectl run green --image=traefik/whoami --labels="app=demo,version=green"
kubectl expose pod blue --port=80 --selector="app=demo,version=blue" --name=demo-svc
kubectl run tmp --rm -it --image=busybox -- wget -qO- demo-svc   # hits "blue"

kubectl patch svc demo-svc -p '{"spec":{"selector":{"app":"demo","version":"green"}}}'
kubectl run tmp --rm -it --image=busybox -- wget -qO- demo-svc   # now hits "green"
```
Nothing about either Pod changed — only the Service's selector — this is
the entire mechanism behind blue/green and canary release patterns you'll
hear about in interviews.

Cleanup:
```bash
kubectl delete pod demo blue green
kubectl delete svc demo-svc
```

## Debugging Techniques

- **Empty results from a selector you expected to match something** — run
  `kubectl get pods --show-labels` and eyeball actual label values;
  whitespace, case-sensitivity, and singular/plural naming mismatches
  (`env`/`environment`) are the overwhelming majority of these bugs.
- **`kubectl get <type> -l <query> -o name`** — a fast way to check
  exactly which objects a selector resolves to, independent of whatever
  higher-level object (Service, ReplicaSet) uses that same selector
  internally.
- **"Immutable field" error when editing a Deployment's selector** — by
  design (see above); if you truly need to repoint it, you must delete and
  recreate the Deployment (understanding this will orphan/adopt Pods based
  on the new selector, so do this deliberately, never by accident).

## Best Practices

- Adopt the Kubernetes-recommended label set early:
  `app.kubernetes.io/name`, `app.kubernetes.io/instance`,
  `app.kubernetes.io/version`, `app.kubernetes.io/component` — Helm
  (Chapter 26) and most ecosystem tooling assume these exist.
- Keep selectors as narrow/specific as actually needed — an
  over-broad selector (like Chapter 9's Mini Assignment, intentionally)
  is a common accidental-production-incident shape when done by mistake
  instead of on purpose.
- Never rely on label ordering or use labels for large/non-queryable
  data — that's what `annotations` are for (Chapter 6); labels are meant
  to be short, indexed, and selector-friendly.

## Interview Questions

1. **Why does Kubernetes couple objects via label selectors instead of
   direct name references?** Decoupling — a Service (or ReplicaSet) can
   target any Pods matching a query, regardless of what created them,
   enabling patterns like blue/green cutover by just changing a selector.
2. **What's the difference between `matchLabels` and
   `matchExpressions`?** `matchLabels` is exact-match sugar for one `In`
   expression per key; `matchExpressions` supports richer operators
   (`In`, `NotIn`, `Exists`, `DoesNotExist`) and multiple conditions,
   always ANDed together.
3. **Why is a Deployment's selector immutable but a Service's isn't?**
   Changing a Deployment's selector could silently orphan or steal
   running Pods it doesn't fully control the lifecycle of elsewhere;
   Services are deliberately designed to be repointed live, which is what
   enables blue/green releases.
4. **How would you implement a blue/green deployment using just labels
   and selectors?** Run both versions as separate Deployments with a
   shared `app` label but different `version` labels, then flip a
   Service's selector's `version` value to cut traffic over instantly,
   with an instant revert available the same way.
5. **What are labels not meant to be used for?**
   Large or non-queryable metadata — use `annotations` for that; labels
   should stay short, structured, and selector-friendly.

## Mini Assignment

Implement a full blue/green cutover exactly like the hands-on lab, but
this time using two real Deployments (not bare Pods) with `replicas: 2`
each, and confirm via repeated `wget` through the Service that 100% of
traffic shifts the moment you patch the selector — no restart, no
downtime, no change to either Deployment's YAML.

## Lesson Summary

- Labels are free-form key/value tags on any object; selectors query
  them, live and continuously, using `matchLabels` (exact) or
  `matchExpressions` (richer operators), always ANDed together.
- This is the mechanism every selector-based object you've met so far
  (ReplicaSet, Deployment, Service) relies on — deliberately decoupling
  "who provides Pods" from "who consumes them."
- Deployment selectors are immutable (safety); Service selectors are
  mutable (this is precisely what makes blue/green cutovers possible with
  zero downtime).

---

### Before Chapter 11 (Namespaces) — tell me:

1. Why couldn't you implement blue/green cutover this cleanly if Services
   referenced Deployments by name instead of by label selector?
2. What's the practical difference between `matchLabels` and
   `matchExpressions` — when would you actually need the latter?
3. Did your Mini Assignment cutover show any dropped requests during the
   `kubectl patch` moment?
