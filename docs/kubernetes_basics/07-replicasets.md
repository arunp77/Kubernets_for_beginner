# Chapter 7 — ReplicaSets

## What it is

A **ReplicaSet** is a controller whose entire job is: "ensure exactly N
Pods matching this label selector exist, at all times." That's the whole
scope — it does not know about versions, rollouts, or history; it only
counts and corrects.

## Why it exists

This is the **first concrete controller** you'll operate yourself, and it
exists to solve exactly the Chapter 1 self-healing problem in its purest
form. Recall `docker run --restart=always` only restarts a *specific*
container on a *specific* host. A ReplicaSet instead watches "how many
Pods matching this selector exist right now?" against "how many should
exist" and creates or deletes Pods — on *any* healthy node in the cluster
— to close the gap. Delete a Pod it manages, by hand, right now, and it
comes back within moments as a brand-new Pod (new name, new IP) — not the
same Pod resurrected, a *replacement*.

## When to use it

Almost never directly — this is important to say plainly. You will use a
**Deployment** (Chapter 8) for nearly everything, and a Deployment
*creates and manages ReplicaSets for you*, adding rollout/rollback/version
history on top. Learning ReplicaSets in isolation first is purely
pedagogical: Deployments are a thin, valuable layer on top of exactly this
mechanism, and that layering only makes sense once you've seen the layer
underneath work on its own.

## Internal architecture

- The ReplicaSet controller runs inside `kube-controller-manager`
  (Chapter 2). It watches the API server for ReplicaSet objects and for
  Pods matching each one's `selector`.
- On every reconciliation pass: count matching Pods. If count < desired
  `replicas`, create new Pods from the ReplicaSet's `template`. If count >
  desired, delete the excess (oldest-first by default, though the exact
  tie-breaking logic considers Pod readiness/restart count too).
- Critically, the **selector is independent of the template** — a
  ReplicaSet manages *any* Pod matching its selector, even ones it didn't
  create itself, as long as the labels match. This is why Kubernetes
  requires `spec.selector` to be a subset of `spec.template.metadata.labels`
  — otherwise a ReplicaSet could create Pods that don't match its own
  selector, which would immediately be "under-replicated" again, an
  infinite loop the API server rejects at validation time.
- Pods are linked back to their owning ReplicaSet via an
  **ownerReference** in the Pod's metadata — this is how `kubectl delete`
  cascading works, and how you can trace any Pod back to what's managing
  it.

## YAML Definition

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: whoami-rs
  labels:
    app: whoami
spec:
  replicas: 3
  selector:
    matchLabels:
      app: whoami
  template:
    metadata:
      labels:
        app: whoami
    spec:
      containers:
        - name: whoami
          image: traefik/whoami
          ports:
            - containerPort: 80
```

- `apiVersion: apps/v1` — ReplicaSet lives in the `apps` API group
  (Chapter 6's group-vs-core distinction), alongside Deployment/
  StatefulSet/DaemonSet — all the "manage a set of Pods" controllers.
- `spec.replicas` — the desired count; this is the single number the
  reconciliation loop enforces.
- `spec.selector.matchLabels` — which existing Pods this ReplicaSet
  considers "mine." Must match (be a subset of) the labels in
  `template.metadata.labels`, enforced by the API server.
- `spec.template` — a **full Pod spec**, nested — everything from
  Chapter 5 (`metadata.labels`, `spec.containers`, etc.) appears here
  unchanged, just indented one level deeper because it's a *template* for
  Pods the ReplicaSet will stamp out, not a live Pod itself.

## Hands-on Example

```bash
kubectl apply -f whoami-rs.yaml
kubectl get rs
kubectl get pods -l app=whoami
```
Three Pods appear, all owned by the ReplicaSet — confirm ownership:
```bash
kubectl get pod -l app=whoami -o jsonpath='{.items[0].metadata.ownerReferences}'
```

**Prove self-healing, live:**
```bash
kubectl delete pod -l app=whoami --field-selector=status.phase=Running --limit=1
# (or simply: kubectl delete pod <one-specific-pod-name>)
kubectl get pods -l app=whoami -w
```
Watch (`-w`) a replacement Pod appear within moments — new name, and if
you check, a new IP — never the same Pod "coming back," always a fresh
one, exactly per Chapter 5's ephemeral-IP lesson.

**Scale imperatively, then feel why that's not enough on its own:**
```bash
kubectl scale rs whoami-rs --replicas=5
kubectl get pods -l app=whoami
```
Now try to change the image via `kubectl edit rs whoami-rs` (change
`image: traefik/whoami` to something else) and watch:
```bash
kubectl get pods -l app=whoami -o wide
```
**Nothing happens to the existing Pods.** The ReplicaSet controller only
enforces *count*, not *content* — it never compares a running Pod's spec
against the template except when it needs to create a *new* one. This is
the single most important thing to internalize before Chapter 8: a
ReplicaSet cannot roll out a change to already-running Pods, on purpose —
that missing piece is exactly what a Deployment adds.

Cleanup:
```bash
kubectl delete -f whoami-rs.yaml
```

## Debugging Techniques

- `kubectl get rs` shows `DESIRED`, `CURRENT`, `READY` columns side by
  side — a mismatch between them (e.g., `DESIRED=3 CURRENT=1`) tells you
  immediately the reconciliation loop can't keep up, usually because new
  Pods are stuck `Pending` (check scheduler-level Chapter 2 causes) or
  `CrashLoopBackOff` (check Chapter 5 causes).
- `kubectl describe rs <name>` shows recent Events like
  `SuccessfulCreate`/`SuccessfulDelete` — a fast way to see the
  reconciliation loop's recent actions without cross-referencing Pod
  timestamps yourself.
- **Two ReplicaSets fighting over the same Pods** — if two ReplicaSets'
  selectors accidentally overlap, they'll both try to "correct" the Pod
  count to their own desired number, causing Pods to be created and
  deleted in a churning loop. Always give selectors a unique, deliberate
  label (this is why Chapter 10 on Labels/Selectors matters so much).
- **Deleted a Pod but a new one appeared instead of it just staying
  gone** — not a bug, this is the controller working as designed; to
  actually remove Pods permanently, scale down or delete the ReplicaSet
  itself.

## Best Practices

- Never hand-author bare ReplicaSets in production manifests — always let
  a Deployment own them (Chapter 8 shows exactly how).
- Never give two different ReplicaSets (or a ReplicaSet and anything else)
  overlapping selectors — this is a real, easy-to-make production
  incident class.
- Remember `replicas` is a floor-and-ceiling, not a "launch and forget"
  count — treat any persistent `DESIRED != CURRENT` gap as an active
  incident, not eventual consistency that'll sort itself out.

## Interview Questions

1. **What does a ReplicaSet actually guarantee?**
   That exactly N Pods matching its label selector exist at any time — it
   reacts to count mismatches, nothing about content/version.
2. **If you change a ReplicaSet's Pod template image, what happens to
   already-running Pods?**
   Nothing — the ReplicaSet controller only compares desired vs. actual
   *count*; it doesn't retroactively update existing Pods' specs. New
   Pods created afterward (e.g. after scaling up, or replacing a deleted
   one) use the new template.
3. **Why must `spec.selector` be a subset of
   `spec.template.metadata.labels`?**
   So Pods the ReplicaSet creates are guaranteed to match its own
   selector — otherwise it would perpetually consider itself
   under-replicated and create Pods forever.
4. **How does Kubernetes know which ReplicaSet owns a given Pod?**
   Via an `ownerReference` in the Pod's metadata, set when the ReplicaSet
   creates it.
5. **Why do Deployments exist if ReplicaSets already provide
   self-healing?**
   Because ReplicaSets deliberately don't handle rolling out template
   changes to existing Pods — Deployments add exactly that layer
   (versioned rollout/rollback) on top of ReplicaSet's proven
   count-enforcement mechanism.

## Mini Assignment

Create two ReplicaSets with selectors that overlap by one shared label
(on purpose) and watch what happens to `kubectl get pods` and each
ReplicaSet's `DESIRED`/`CURRENT` counts over a minute or two of `kubectl
get rs -w`. Then fix it by giving them properly disjoint selectors.
Document what you observed during the conflict — no solution given here.

## Lesson Summary

- A ReplicaSet enforces one thing: N Pods matching a selector, always,
  recreating on any deletion/crash onto any healthy node.
- It deliberately does **not** roll out template changes to already-running
  Pods — that gap is intentional and is exactly what Deployments add next.
- Pods are linked to their ReplicaSet via `ownerReference`; selectors must
  be a subset of the Pod template's labels, enforced by the API server.
- You almost never write a bare ReplicaSet in real manifests — but
  understanding it fully is what makes Deployments make sense.

---

### Before Chapter 8 (Deployments) — tell me:

1. What exactly does "DESIRED=3, CURRENT=1" tell you is broken, and where
   would you look first?
2. Why doesn't editing a ReplicaSet's image field affect Pods that are
   already running?
3. What did you observe when two ReplicaSets' selectors overlapped in the
   Mini Assignment?
