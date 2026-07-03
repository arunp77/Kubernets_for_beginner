# Chapter 6 — YAML for Kubernetes

## What it is

Kubernetes YAML is a serialization of a **REST API resource** — every
manifest you write is literally the JSON body of an HTTP request to the
API server, just written in YAML for human readability (the API server
converts it to JSON internally; you can `kubectl apply` valid JSON too,
it's just less pleasant to write by hand). You already know YAML syntax
from Compose — this chapter is about the four fields every Kubernetes
object shares, and why they're structured the way they are.

## Why it exists

Docker Compose YAML describes "a fixed set of services to run on this
host, now." Kubernetes YAML describes **an object's desired state, as a
durable record** — it's not "run this," it's "this should exist, forever,
looking exactly like this, until I say otherwise." That's why every
Kubernetes object, no matter its type, shares the same four top-level
fields — the API server needs a uniform envelope so any client (kubectl,
a controller, another object's owner reference) can identify and route
any object without knowing its specific schema in advance.

## When to use it

Every single Kubernetes object, without exception, is defined this way —
Pods, Deployments, Services, ConfigMaps, everything you'll touch for the
rest of this course. There's no alternative format; YAML/JSON via the API
is the only way objects are created, whether you write it by hand or a
tool like Helm (Chapter 26) generates it for you.

## Internal architecture — the four fields every object has

```yaml
apiVersion: apps/v1     # 1. WHICH schema/API group+version
kind: Deployment          # 2. WHAT type of object
metadata:                 # 3. WHO/WHERE — identity
  name: my-app
  namespace: default
  labels:
    app: my-app
spec:                     # 4. WHAT — your desired state
  replicas: 3
  ...
# status:                 # NOT written by you — see below
#   readyReplicas: 3
```

- **`apiVersion`** — identifies which API group and version this object's
  schema belongs to. Core, stable, old types (Pod, Service, ConfigMap,
  Namespace) use just `v1` (the "core"/legacy group, no prefix). Newer,
  grouped types use `<group>/<version>`, e.g. `apps/v1` for
  Deployment/ReplicaSet/StatefulSet, `batch/v1` for Job/CronJob,
  `networking.k8s.io/v1` for Ingress/NetworkPolicy. The version suffix
  (`v1`, `v1beta1`, `v1alpha1`) signals stability — `v1` is safe for
  production; alpha/beta fields can change or disappear between
  Kubernetes releases. Getting `apiVersion` wrong for a given `kind` is
  one of the most common early errors — the API server rejects it
  outright with a clear error naming the mismatch.
- **`kind`** — literally which object schema to validate/store this as.
  Case-sensitive, exact match (`Deployment`, not `deployment`).
- **`metadata`** — identity and bookkeeping, *not* your application's
  desired behavior. `name` (unique within a namespace), `namespace`
  (Chapter 11), `labels` (Chapter 10 — arbitrary key/value tags used for
  selection), and `annotations` (arbitrary key/value metadata *not* used
  for selection — free-form notes tools attach, like the
  last-applied-configuration you saw in Chapter 4).
- **`spec`** — the actual desired state, and the only field whose internal
  shape differs per `kind`. Everything domain-specific you'll learn in
  Chapters 7 onward (replica counts, container specs, port mappings, probe
  configs) lives here.
- **`status`** (not shown above on purpose) — the API server's own record
  of *observed* state, written by controllers, never by you. When you
  `kubectl get -o yaml` a live object, you'll see a `status` block
  appended — this is the "observed state" half of the reconciliation loop
  from Chapter 1/2, made visible. Never hand-author this field; editing it
  yourself accomplishes nothing since a controller overwrites it on its
  next reconciliation pass.

## Compose vs. Kubernetes YAML — a direct field mapping

| Compose | Kubernetes | Note |
|---|---|---|
| `services.web.image` | `spec.containers[].image` (inside a Pod template) | Same concept, nested one level deeper because of the Pod/controller split |
| `services.web.ports` | Two separate things: `containerPort` (Pod) + a Service's `port`/`targetPort` (Ch. 9) | Compose conflates "container's port" and "how it's reachable"; Kubernetes splits them deliberately |
| `services.web.environment` | `spec.containers[].env` | Same idea, same list-of-key-value shape |
| `services.web.volumes` | `spec.volumes` + `volumeMounts` (Ch. 5, 14) | Kubernetes splits "the volume" from "where each container mounts it" |
| `services.web.deploy.replicas` | `spec.replicas` (on a Deployment, Ch. 8) | Compose bolts this on optionally; Kubernetes makes it a first-class field of a dedicated controller object |
| Whole file = one app | One YAML **file can, and typically does, contain multiple objects** separated by `---` | See below |

## Hands-on Example — multi-document files

Kubernetes YAML files commonly define several objects at once, separated
by `---` (standard YAML document separator) — this is how you'll usually
ship a Deployment + its Service together as one file starting Chapter 9:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: doc-a
spec:
  containers:
    - name: nginx
      image: nginx
---
apiVersion: v1
kind: Pod
metadata:
  name: doc-b
spec:
  containers:
    - name: nginx
      image: nginx
```
```bash
kubectl apply -f multi-doc.yaml
kubectl get pods
```
Both Pods are created from one `apply` call — the API server (and
`kubectl`) parses each `---`-delimited document as an independent object.

**See the status field appear**, proving it's the API server's territory,
not yours:
```bash
kubectl apply -f - <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: status-demo
spec:
  containers:
    - name: nginx
      image: nginx
EOF
kubectl get pod status-demo -o yaml | grep -A5 "^status:"
```
Notice `status` wasn't in your input file at all — the API server
appended it, and it's continuously updated by the kubelet reporting real
container state.

**Use `kubectl explain` instead of guessing field names** — this is the
single most useful habit for writing new YAML from now on:
```bash
kubectl explain pod.spec.containers
kubectl explain pod.spec.containers.resources
```
This queries the API server's own OpenAPI schema — always accurate for
*your* cluster's exact Kubernetes version, unlike a search result that
might describe a different version.

Cleanup:
```bash
kubectl delete pod doc-a doc-b status-demo
```

## Debugging Techniques

- **`error: error validating "file.yaml": ... unknown field`** — either a
  typo, wrong indentation nesting a field under the wrong parent, or a
  genuinely nonexistent field for that `apiVersion`/`kind` combination.
  Cross-check with `kubectl explain <kind>.<path>`.
- **`no matches for kind "X" in version "Y"`** — `apiVersion` doesn't
  support that `kind`, or (less often) your cluster is missing a CRD that
  would provide it (Custom Resource Definitions — beyond this course's
  scope, but you'll meet the error again with Helm charts in Chapter 26).
- **YAML indentation bugs** — the classic "list item silently attached to
  the wrong parent" — `kubectl apply --dry-run=client -o yaml -f file.yaml`
  round-trips your file through the client-side parser and prints exactly
  what would be sent, without touching the cluster — an excellent sanity
  check before applying anything you're unsure about.
- **Never hand-edit `status`** — if you're tempted to fix something by
  editing `status` directly, you're solving the wrong layer; fix `spec`
  and let the relevant controller reconcile `status` itself.

## Best Practices

- Keep `spec` and `metadata.labels` intentional and minimal — every label
  is a selector surface other objects (Services, NetworkPolicies) might
  match against, so accidental/inconsistent labels cause real bugs later
  (Chapter 10).
- Pin `apiVersion` to stable (`v1`, or GA-versioned group APIs), not
  `alpha`/`beta`, for anything you deploy to production — those schemas
  can change between Kubernetes upgrades.
- Store manifests in git, one logical unit of related objects per file
  (e.g. a Deployment + its Service in one `---`-joined file) — this is the
  foundation Helm (Ch. 26) and GitOps CI/CD (Ch. 29) both build on.
- Run `kubectl apply --dry-run=server -f file.yaml` before applying
  anything to a shared cluster — `server` dry-run validates against the
  live API server (catching things client-side dry-run can't, like RBAC
  or admission-webhook rejections) without persisting the change.

## Interview Questions

1. **What four fields does every Kubernetes object share, and what's each
   for?** `apiVersion` (schema/group+version), `kind` (object type),
   `metadata` (identity: name/namespace/labels/annotations), `spec`
   (desired state — the only field whose shape varies by `kind`).
2. **What is the `status` field, and who's allowed to write it?**
   The API server/controllers' record of *observed* state; you only ever
   read it, never author it — it's overwritten by reconciliation anyway.
3. **Why does `apiVersion` include a group, like `apps/v1`, for some kinds
   but not others?** Core/legacy types predate the API-group mechanism and
   stayed at bare `v1`; newer types are organized into groups
   (`apps`, `batch`, `networking.k8s.io`, etc.) for cleaner API
   versioning and extension.
4. **How can one YAML file define multiple Kubernetes objects?**
   `---`-separated YAML documents within one file; both `kubectl` and the
   API server treat each document as an independent object to create/
   update.
5. **What's the fastest way to check if a field name in a manifest is
   valid before applying it?** `kubectl explain <kind>.<field.path>` —
   queries your actual cluster's live OpenAPI schema.

## Mini Assignment

Take the multi-container Pod YAML from Chapter 5's Mini Assignment and
deliberately introduce three separate mistakes, one at a time: (1) an
invalid `apiVersion` for `kind: Pod`, (2) a field nested one level too
deep (e.g., put `image` directly under `spec` instead of under
`spec.containers[]`), (3) a real typo in a field name. For each, run
`kubectl apply --dry-run=client -o yaml -f file.yaml` first, then the real
`apply`, and write down the exact error text the API server gives you for
each — building a personal reference for what these error classes look
like before you hit them for real under pressure.

## Lesson Summary

- All Kubernetes YAML is just the serialized body of a REST request; every
  object shares `apiVersion`, `kind`, `metadata`, `spec` — and gets a
  controller-owned `status` appended once it exists.
- `apiVersion` tells the API server which schema to validate against;
  getting it wrong for a given `kind` is a very common, very clear error.
- Files can and routinely do define multiple objects via `---` separators
  — you'll lean on this constantly starting with Deployment+Service pairs.
- `kubectl explain` and `--dry-run=client|server` are your two core tools
  for writing and validating YAML correctly before it touches the cluster.

---

### Before Chapter 7 (ReplicaSets) — tell me:

1. Why does `status` exist as a separate field from `spec`, rather than
   Kubernetes just mutating `spec` to reflect reality?
2. What's the difference between `labels` and `annotations` in
   `metadata`, functionally?
3. What errors did you get for each of the three deliberate mistakes in
   the Mini Assignment — did `--dry-run=client` catch all of them, or only
   some?
