# Chapter 11 — Namespaces

## What it is

A **Namespace** is a virtual partition within one physical cluster —
a scoping mechanism for names, RBAC permissions (Ch. 24), resource quotas,
and Network Policies (Ch. 25). Objects in different namespaces can share
the same `name` without conflict; most object types are "namespaced,"
while a few (Nodes, PersistentVolumes, Namespaces themselves) are
cluster-scoped and exist outside any namespace.

## Why it exists

One physical Kubernetes cluster is expensive to stand up and operate
(Chapter 2's whole control plane), so in practice many teams/environments
share one cluster rather than each getting a dedicated one. Namespaces
give you the isolation you'd otherwise need separate clusters for — team
A's `staging` and team B's `staging` can both have a Deployment named
`api` without collision — without paying for a second control plane.
Analogy to something you know: this is conceptually closer to separate
Docker Compose *projects* (each with its own project name/network
namespace) sharing one Docker Engine, than to separate machines.

## When to use it

To separate environments (`dev`/`staging`/`prod` — though many production
setups actually prefer *separate clusters* for prod vs. non-prod, for
blast-radius reasons, and use namespaces mainly *within* an environment),
separate teams sharing a cluster, or to scope third-party
tooling (Prometheus, cert-manager, ingress-nginx conventionally each get
their own namespace, which is why you saw `kube-system` constantly in
earlier chapters — it's simply the namespace Kubernetes' own control-plane
Pods live in).

## Internal architecture

- A Namespace is itself just another API object (`kind: Namespace`,
  cluster-scoped). Every namespaced object's full identity to the API
  server is really `(namespace, name)`, not `name` alone — this is why two
  Pods named `api` in different namespaces are entirely unrelated objects.
- **DNS is namespace-aware**: recall Chapter 9's `<service>.<namespace>.
  svc.cluster.local` — within the same namespace, the short name works;
  across namespaces, you must qualify it. This is a frequent source of
  "works in this namespace, breaks when I move it" bugs.
- **Namespaces do NOT provide network isolation by default** — a Pod in
  `namespace-a` can reach a Pod in `namespace-b` over the network exactly
  as freely as any other Pod in the cluster unless a NetworkPolicy
  (Chapter 25) explicitly restricts it. This is a very common
  misconception worth correcting now: namespace boundary ≠ security
  boundary, on its own.
- **ResourceQuota** and **LimitRange** objects (both namespace-scoped) let
  you cap total CPU/memory/object-count consumption per namespace — the
  practical mechanism for "team A can't starve team B" on a shared
  cluster.

## YAML Definition

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: staging
  labels:
    environment: staging
---
apiVersion: v1
kind: ResourceQuota
metadata:
  name: staging-quota
  namespace: staging
spec:
  hard:
    requests.cpu: "4"
    requests.memory: 8Gi
    limits.cpu: "8"
    limits.memory: 16Gi
    pods: "20"
```
- `kind: Namespace` — no `spec` of real substance for a plain namespace;
  it exists purely to be referenced as a scope by other objects.
- `ResourceQuota.metadata.namespace: staging` — quotas themselves live
  *inside* the namespace they constrain (this is the general rule: any
  namespaced object's YAML must specify, or inherit via `kubectl -n`,
  which namespace it belongs to).
- `spec.hard` — the actual caps: total requested/limited CPU and memory
  summed across every Pod in the namespace, and a hard cap on Pod count.
  Once hit, new Pod creations are rejected by the API server at admission
  time, with a clear quota-exceeded error.

## Hands-on Example

```bash
kubectl apply -f staging-ns.yaml
kubectl get ns
kubectl get ns staging -o yaml
```

**Same name, two namespaces, no conflict** — prove the `(namespace,
name)` identity claim directly:
```bash
kubectl create namespace team-a
kubectl create namespace team-b
kubectl run api --image=traefik/whoami -n team-a
kubectl run api --image=traefik/whoami -n team-b
kubectl get pods -A | grep api      # both exist, same name, different namespaces
```

**Cross-namespace DNS**, proving the short-name convenience is
namespace-scoped:
```bash
kubectl expose pod api -n team-a --port=80 --name=api-svc
kubectl run tmp -n team-a --rm -it --image=busybox -- wget -qO- api-svc               # works — same namespace
kubectl run tmp -n team-b --rm -it --image=busybox -- wget -qO- api-svc               # fails — wrong namespace
kubectl run tmp -n team-b --rm -it --image=busybox -- wget -qO- api-svc.team-a        # works — fully qualified
```

**Prove namespaces don't isolate network traffic by default:**
```bash
kubectl run tmp -n team-b --rm -it --image=busybox -- wget -qO- api-svc.team-a.svc.cluster.local
```
This succeeds with no NetworkPolicy in place — a fully separate namespace
is not, by itself, a network security boundary; that's Chapter 25's job.

**Set a default namespace for your kubectl context**, so you stop typing
`-n` constantly:
```bash
kubectl config set-context --current --namespace=staging
kubectl config view --minify | grep namespace:
```

**Apply and hit the quota you defined:**
```bash
kubectl apply -f staging-quota.yaml -n staging
kubectl create deployment quota-test --image=nginx --replicas=50 -n staging
kubectl get events -n staging | grep -i quota
```

Cleanup:
```bash
kubectl delete namespace staging team-a team-b
```
(Deleting a namespace cascades — every namespaced object inside it is
deleted too; a genuinely destructive, hard-to-reverse action in a real
cluster — always double-check which namespace you're targeting.)

## Debugging Techniques

- **"Object not found" that actually exists** — the single most common
  namespace bug: you're querying the wrong namespace (or the default one,
  by omission). Always try `kubectl get <type> -A` (all namespaces) before
  concluding something doesn't exist.
- **Cross-namespace Service calls failing** — check whether the caller
  used the short name (only valid within the same namespace) versus the
  fully-qualified `<svc>.<namespace>` form.
- **Deployment stuck, `Error creating: pods "x" is forbidden: exceeded
  quota`** — a ResourceQuota is capping you; `kubectl describe quota
  <name> -n <ns>` shows current usage against the hard limits.
- **`kubectl delete namespace` hangs in `Terminating`** — usually a
  "finalizer" on some object inside it blocking clean deletion (common
  with certain custom resources); a real operational headache worth
  knowing exists, not something to force through carelessly.

## Best Practices

- Set your `kubectl` context's default namespace per environment/project
  to avoid accidentally operating on the wrong one — but always sanity
  check `kubectl config view --minify` before anything destructive.
- Apply a `ResourceQuota` + `LimitRange` to every namespace on a shared
  cluster from day one — an ungoverned namespace can starve every other
  tenant on the same cluster.
- Don't rely on namespace boundaries for security — pair them with
  NetworkPolicies (Ch. 25) and RBAC (Ch. 24) for genuine isolation.
- Reserve fully separate clusters (not just namespaces) for hard
  boundaries like production vs. non-production, where you want faults,
  quota exhaustion, or control-plane issues on one side to be physically
  incapable of affecting the other.

## Interview Questions

1. **What problem do Namespaces solve?**
   Scoping — for names, RBAC, quotas, and network policy — allowing
   multiple teams/environments to share one physical cluster without
   naming collisions or (with the right add-ons) unbounded resource
   contention.
2. **Do Namespaces provide network isolation by default?**
   No — Pods in different namespaces can reach each other freely unless a
   NetworkPolicy explicitly restricts it; namespace boundary and network
   security boundary are separate concerns.
3. **How does a Pod's real identity to the API server differ from just
   its `name`?** It's the pair `(namespace, name)` — two Pods with the
   same name in different namespaces are unrelated distinct objects.
4. **What's the difference between a namespaced and a cluster-scoped
   object?** Namespaced objects (Pods, Deployments, Services, etc.) live
   inside exactly one namespace; cluster-scoped objects (Nodes,
   PersistentVolumes, Namespaces themselves) exist outside any namespace,
   visible cluster-wide.
5. **How would you prevent one team from consuming all cluster
   resources on a shared cluster?** A `ResourceQuota` on their namespace
   capping total requested/limited CPU, memory, and object counts,
   enforced by the API server at admission time.

## Mini Assignment

Create three namespaces (`dev`, `staging`, `prod`), apply a different
`ResourceQuota` to each (prod generous, dev tight), then try to deploy the
same Deployment manifest unmodified into all three using `kubectl apply -f
file.yaml -n <ns>`. Observe and document exactly what error you get in
whichever namespace's quota you deliberately size too small to fit it.

## Lesson Summary

- Namespaces virtually partition one physical cluster for naming, RBAC,
  and quota scoping — sharing one expensive control plane across
  teams/environments, similar in spirit to separate Compose projects
  sharing one Docker Engine.
- An object's true identity is `(namespace, name)`; DNS short names only
  resolve within the same namespace, and cross-namespace calls need the
  fully-qualified form.
- Namespaces are **not** a network security boundary by default — that
  requires NetworkPolicies (Ch. 25) layered on top.
- ResourceQuota/LimitRange are the practical multi-tenancy tools that make
  sharing a cluster across teams safe.

---

### Before Chapter 12 (ConfigMaps) — tell me:

1. Why did `wget api-svc` succeed from `team-a` but fail from `team-b` in
   the hands-on lab, and what fixed it?
2. Is it true or false that putting two teams in separate namespaces
   protects them from each other's network traffic? Why?
3. What exact error did you get in the Mini Assignment when a Deployment
   exceeded its namespace's quota?
