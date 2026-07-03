# Chapter 24 — RBAC

## What it is

**RBAC (Role-Based Access Control)** governs *who* (a user, group, or
**ServiceAccount** — a Pod's own identity) can perform *which actions*
(verbs like `get`, `list`, `create`, `delete`) on *which objects*
(Pods, Secrets, Deployments, etc.), optionally scoped to one namespace or
cluster-wide. Every single API request you've made since Chapter 4 has
passed through an RBAC check — you just haven't hit a denial yet, because
Kind's default kubeconfig grants you full admin access.

## Why it exists

Chapter 2 established the API server as the single front door to
everything in the cluster. A front door needs a lock. RBAC is that lock —
without it, anyone who can reach the API server (any authenticated user,
any Pod with API access) could read every Secret, delete any Deployment,
or grant themselves more permissions. Beyond humans, RBAC is equally
critical for **workload identity**: your own application Pods (and
cluster add-ons like the Ingress Controller you installed in Chapter 16,
or the metrics-server from Chapter 23) run *as* a ServiceAccount, and
should only be able to do exactly what they need — nothing more. This is
the direct, practical application of the **principle of least privilege**
to a Kubernetes cluster.

## When to use it

Always — RBAC is enabled by default on essentially every real cluster.
The actual work is designing *which* Roles you grant to *which*
identities: developers who should manage Deployments but not read
Secrets, a CI/CD pipeline (Chapter 29) that can deploy to `staging` but
not `prod`, your application's own ServiceAccount that should only read
one specific ConfigMap it actually needs.

## Internal architecture

- Four object types work together:
  - **Role** — a set of permission rules (verbs on resources),
    **namespace-scoped**.
  - **ClusterRole** — the same idea, but **cluster-scoped** (can grant
    access across all namespaces, or to cluster-scoped resources like
    Nodes/Namespaces themselves, which have no namespace to scope a plain
    Role to at all).
  - **RoleBinding** — grants a Role (or, notably, a ClusterRole too) to a
    specific subject (user/group/ServiceAccount), **within one
    namespace**.
  - **ClusterRoleBinding** — grants a ClusterRole to a subject
    **cluster-wide**.
- A **ServiceAccount** is an identity Pods run as — every Pod has one
  (defaulting to a namespace's `default` ServiceAccount if you don't
  specify one) — and the kubelet automatically mounts a token for it into
  the Pod's filesystem, letting *your application code itself* call the
  Kubernetes API as that identity, if it needs to (many apps never need
  this; some, like the Ingress Controller, fundamentally do).
- **Authorization is purely additive and deny-by-default**: RBAC has no
  explicit "deny" rule type — access is granted only by the union of
  every Role/ClusterRole bound to a subject; anything not explicitly
  granted is implicitly denied. This is a deliberately simple model
  (compared to, say, systems with both allow *and* deny rules) — it
  trades some expressiveness for being much easier to reason about
  correctly.
- Every RBAC check happens **inside the API server itself**, before a
  request is ever processed further — this is why a `Forbidden` error
  (Chapter 4) is a distinctly different failure mode from a connectivity
  problem: the request *reached* the API server fine, and was explicitly
  rejected there.

## YAML Definition

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: whoami-reader
  namespace: default
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: default
rules:
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: whoami-reader-binding
  namespace: default
subjects:
  - kind: ServiceAccount
    name: whoami-reader
    namespace: default
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```
- `ServiceAccount` — a namespaced identity object; on its own it grants
  nothing — permissions only exist once bound.
- `Role.rules[].apiGroups` — `""` means the **core** API group (Pods,
  Services, ConfigMaps, Secrets — Chapter 6's core-vs-grouped
  distinction, now relevant to security config too); use `"apps"` for
  Deployments/ReplicaSets, `"batch"` for Jobs/CronJobs, etc.
- `resources` — the object type(s) this rule covers.
- `verbs` — the specific actions allowed: `get`/`list`/`watch` (read-only,
  as here), or `create`/`update`/`patch`/`delete` for write access —
  granted individually, not as an all-or-nothing bundle.
- `RoleBinding.subjects` — who receives the permission; here, the
  `whoami-reader` ServiceAccount specifically.
- `roleRef` — which Role is being granted; note a RoleBinding can also
  reference a **ClusterRole** here while still only granting it within
  this one namespace — a genuinely useful pattern for reusing one
  ClusterRole definition (e.g., a common "view" role) across many
  namespace-scoped bindings without redefining the rules each time.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: whoami-test
spec:
  serviceAccountName: whoami-reader
  containers:
    - name: kubectl
      image: bitnami/kubectl
      command: ["sleep", "3600"]
```
`spec.serviceAccountName` — explicitly runs this Pod as the
`whoami-reader` identity rather than the namespace's `default` one.

## Hands-on Example

```bash
kubectl apply -f whoami-sa.yaml -f pod-reader-role.yaml -f whoami-rolebinding.yaml
kubectl apply -f whoami-test-pod.yaml
```

**Prove the ServiceAccount's token grants exactly the Role's
permissions — no more, no less:**
```bash
kubectl exec -it whoami-test -- sh
# inside the pod:
kubectl get pods                 # succeeds — the Role grants this
kubectl get secrets              # Forbidden — the Role never granted this
kubectl delete pod whoami-test   # Forbidden — no "delete" verb was granted, even on pods
```
Notice `get pods` succeeds using nothing but the automatically-mounted
ServiceAccount token — you never ran `kubectl config` inside the Pod;
`kubectl` running in-cluster auto-discovers its own Pod's ServiceAccount
credentials.

**Use `kubectl auth can-i` to check permissions without actually trying
the action** — the single most useful RBAC debugging tool:
```bash
kubectl auth can-i get pods --as=system:serviceaccount:default:whoami-reader
kubectl auth can-i delete pods --as=system:serviceaccount:default:whoami-reader
kubectl auth can-i get secrets --as=system:serviceaccount:default:whoami-reader
```

**Grant cluster-wide read access instead**, to see the Cluster* variants:
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: pod-reader-cluster
rules:
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: whoami-reader-cluster-binding
subjects:
  - kind: ServiceAccount
    name: whoami-reader
    namespace: default
roleRef:
  kind: ClusterRole
  name: pod-reader-cluster
  apiGroup: rbac.authorization.k8s.io
```
```bash
kubectl apply -f pod-reader-clusterrole.yaml -f whoami-clusterrolebinding.yaml
kubectl auth can-i get pods --as=system:serviceaccount:default:whoami-reader -n kube-system
```
Now `True` — the same ServiceAccount can read Pods in *any* namespace,
not just `default`, because a ClusterRoleBinding (unlike a RoleBinding)
isn't scoped to one namespace.

Cleanup:
```bash
kubectl delete pod whoami-test
kubectl delete -f whoami-sa.yaml -f pod-reader-role.yaml -f whoami-rolebinding.yaml
kubectl delete -f pod-reader-clusterrole.yaml -f whoami-clusterrolebinding.yaml
```

## Debugging Techniques

- **`Error from server (Forbidden): pods is forbidden: User "system:
  serviceaccount:..." cannot ... resource "pods"`** — read this error
  literally, it names the exact identity, verb, and resource denied;
  cross-check against `kubectl auth can-i <verb> <resource> --as=<identity>`
  before touching any YAML.
- **`kubectl auth can-i --list --as=<identity>`** — dumps every permission
  a given identity actually has, the fastest way to audit "what can this
  ServiceAccount actually do" without reading through multiple
  Role/RoleBinding files by hand.
- **A cluster add-on (Ingress Controller, metrics-server) fails with
  Forbidden errors in its own logs** — its installation manifest almost
  certainly ships its own ServiceAccount/ClusterRole/ClusterRoleBinding;
  check those weren't accidentally omitted or edited during install.
- **Permissions "not taking effect" after applying a new Role** — check
  the RoleBinding's `subjects` and `roleRef` reference the *exact* names
  (and, for `subjects`, the exact namespace) of the Role/ServiceAccount —
  a mismatched name silently grants nothing, with no error at apply time
  (the objects themselves are all valid independently).

## Best Practices

- Never bind the built-in `cluster-admin` ClusterRole to anything other
  than genuine cluster administrators — it grants unrestricted access to
  everything, and is a common, dangerous shortcut under deadline pressure.
- Give every non-trivial workload its **own** dedicated ServiceAccount
  with only the permissions it actually needs — never rely on a
  namespace's `default` ServiceAccount for anything beyond truly
  permission-less Pods.
- Prefer namespaced `Role`/`RoleBinding` over cluster-wide grants
  whenever the access genuinely only needs to be namespace-scoped —
  minimize blast radius by default.
- Use `kubectl auth can-i --list` as part of routine security review, not
  just ad-hoc debugging — treat RBAC configuration itself as something
  worth periodically auditing, since permissions tend to accumulate and
  rarely get revoked once granted.

## Interview Questions

1. **What four objects make up Kubernetes RBAC, and how do they relate?**
   Role/ClusterRole define permission rules (namespaced vs. cluster
   -scoped); RoleBinding/ClusterRoleBinding grant a Role/ClusterRole to a
   subject (namespaced vs. cluster-wide binding) — a Role or ClusterRole
   grants nothing on its own until bound.
2. **What's the difference between a Role and a ClusterRole?**
   A Role's permissions apply only within its own namespace; a
   ClusterRole's permissions can apply cluster-wide (via
   ClusterRoleBinding) or be reused within a single namespace (via an
   ordinary RoleBinding referencing it) — also required for granting
   access to cluster-scoped resources like Nodes.
3. **How does a Pod authenticate to the Kubernetes API as itself?**
   Via its ServiceAccount — the kubelet automatically mounts a token for
   the Pod's ServiceAccount into its filesystem, which `kubectl` or any
   API client running inside the Pod uses automatically.
4. **Is there an explicit "deny" rule in Kubernetes RBAC?**
   No — RBAC is purely additive and deny-by-default; access is the union
   of everything explicitly granted, and anything not granted is
   implicitly forbidden.
5. **How would you check what permissions a given ServiceAccount
   actually has, without trial and error?** `kubectl auth can-i <verb>
   <resource> --as=system:serviceaccount:<namespace>:<name>` for a
   specific check, or `--list` for a full dump of its granted
   permissions.

## Mini Assignment

Create a ServiceAccount granted only `get`/`list` on ConfigMaps (nothing
else, not even Pods) in the `default` namespace. Run a Pod as that
ServiceAccount and, from inside it, attempt: reading a ConfigMap (should
succeed), listing Pods (should fail), and reading a Secret (should fail).
Use `kubectl auth can-i --list --as=<identity>` beforehand to predict all
three outcomes, then verify by actually trying each command inside the
Pod.

## Lesson Summary

- RBAC governs who can do what to which objects, via Role/ClusterRole
  (permission rules) bound to subjects via RoleBinding/ClusterRoleBinding
  — purely additive, deny-by-default.
- Every API request since Chapter 4 has passed through this exact check;
  Kind's default kubeconfig just happens to grant full admin, masking it
  until this chapter.
- ServiceAccounts give Pods their own API identity, automatically
  authenticated via a mounted token — critical both for your own
  workloads and for cluster add-ons like Ingress controllers.
- `kubectl auth can-i` (and `--list`) is the core debugging/auditing tool
  for this entire chapter.

---

### Before Chapter 25 (Network Policies) — tell me:

1. Why does a Role or ClusterRole grant nothing at all on its own,
   without a binding?
2. How does a Pod's own process authenticate to the API server as a
   specific identity, without you configuring a kubeconfig inside it?
3. From the Mini Assignment — did your predictions from `kubectl auth
   can-i --list` match what actually happened when you tried each command
   inside the Pod?
