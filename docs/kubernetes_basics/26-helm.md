# Chapter 26 — Helm

## What it is

**Helm** is Kubernetes' package manager: it templates YAML manifests (so
the same underlying set of objects can be parameterized per environment),
bundles them into a versioned, distributable unit called a **chart**, and
tracks each install as a **release** with its own upgrade/rollback
history.

## Why it exists

By now you've written a real Deployment + Service + ConfigMap + Secret +
Ingress + HPA bundle by hand, and you've felt the friction: the same
shape of YAML, repeated with small variations, for every service and
every environment (`dev`/`staging`/`prod` each need different replica
counts, resource limits, hostnames). Copy-pasting and hand-editing YAML
across environments is exactly the kind of manual, error-prone toil this
whole course has been eliminating one layer at a time. Helm is the
natural analogy to something you already know well: think of a chart as
roughly "a Dockerfile + docker-compose.yml combined, but for an entire set
of Kubernetes objects, with build-time (well, install-time) variables."

## When to use it

Distributing/installing any non-trivial application (your own, or
third-party — Prometheus in Chapter 27 is virtually always installed via
Helm in practice) across multiple environments, or simply to get
versioned upgrade/rollback for a whole bundle of related objects as one
atomic unit, the way Chapter 8 gave you for a single Deployment.

## Internal architecture

- A **chart** is a directory (or packaged `.tgz`) with a defined
  structure: `Chart.yaml` (metadata: name, version), `values.yaml`
  (default configuration values), and a `templates/` directory of YAML
  files using **Go template syntax** (`{{ .Values.replicaCount }}`, etc.)
  referencing values from `values.yaml` or overrides you supply at
  install time.
- `helm install` **renders** every template in `templates/` by
  substituting values, concatenates the results (much like Chapter 6's
  multi-document `---`-separated files), and applies the result to the
  cluster — mechanically, this is really "`kubectl apply` on
  templated YAML," with extra bookkeeping layered on top.
- That bookkeeping is a **release**: Helm stores a record (as a Secret,
  by default, in the target namespace) of exactly which rendered
  manifest was applied for each revision. `helm upgrade` renders a new
  version and applies the diff; `helm rollback <revision>` re-applies a
  *previous* revision's exact rendered manifest — this is precisely
  Chapter 8's Deployment rollback concept, generalized to an entire
  multi-object bundle instead of just one Deployment's Pod template.
- **Subcharts and dependencies**: a chart can declare dependencies on
  other charts (e.g., your app's chart depending on a `postgresql`
  subchart) — `Chart.yaml`'s `dependencies` field, resolved via `helm
  dependency update`, letting you compose complex multi-service
  applications (exactly what this course's final project needs) from
  reusable pieces rather than hand-writing everything from scratch.
- Public chart repositories (Bitnami's being one of the largest
  historically) let you `helm install` a production-grade Postgres,
  Redis, or Prometheus stack with a single command and sane, tested
  defaults, rather than hand-authoring StatefulSets/ConfigMaps yourself.

## YAML Definition — chart structure

```
whoami-chart/
├── Chart.yaml
├── values.yaml
└── templates/
    ├── deployment.yaml
    ├── service.yaml
    └── hpa.yaml
```

```yaml
# Chart.yaml
apiVersion: v2
name: whoami-chart
description: A minimal chart wrapping traefik/whoami
version: 0.1.0
appVersion: "1.10"
```

```yaml
# values.yaml
replicaCount: 2
image:
  repository: traefik/whoami
  tag: "v1.10"
resources:
  requests: { cpu: 100m, memory: 64Mi }
  limits: { cpu: 200m, memory: 128Mi }
service:
  port: 80
```

```yaml
# templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Release.Name }}-whoami
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      app: {{ .Release.Name }}-whoami
  template:
    metadata:
      labels:
        app: {{ .Release.Name }}-whoami
    spec:
      containers:
        - name: whoami
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          resources:
            {{- toYaml .Values.resources | nindent 12 }}
```
- `Chart.yaml` — the chart's own identity/version metadata (distinct from
  `appVersion`, which just documents which version of the *wrapped
  application* this chart version corresponds to — the two version
  numbers are independent on purpose, since a chart can be updated
  (bug fixes to templates) without the app itself changing).
- `values.yaml` — every configurable knob, with sensible defaults —
  this is the file a consumer overrides per-environment, never the
  templates themselves.
- `{{ .Values.replicaCount }}` — Go template syntax pulling from
  `values.yaml` (or an override); `{{ .Release.Name }}` is a Helm
  -provided built-in referring to *this specific install's* name,
  letting the same chart be installed multiple times side-by-side
  (e.g., `helm install staging-whoami ./whoami-chart` and `helm install
  prod-whoami ./whoami-chart`) without name collisions.
- `{{- toYaml .Values.resources | nindent 12 }}` — a common real-world
  pattern: dump an entire nested values block back out as YAML,
  re-indented to fit correctly — avoids hand-templating every nested
  field of something like `resources` individually.

## Hands-on Example

```bash
helm create whoami-chart      # scaffolds the structure above with a working starter example
# (replace the generated templates/values with the simplified versions above for this lab)
```

**Render without installing** — see exactly what would be applied, the
Helm equivalent of Chapter 6's `--dry-run`:
```bash
helm template my-release ./whoami-chart
```

**Install for real:**
```bash
helm install my-release ./whoami-chart
helm list
kubectl get deploy,svc -l app=my-release-whoami
```

**Override values at install time**, proving the parameterization value
directly — install a second, independent copy for a "staging" scenario:
```bash
helm install staging-release ./whoami-chart --set replicaCount=1 --set image.tag=v1.9
helm list
kubectl get deploy staging-release-whoami -o jsonpath='{.spec.replicas}'   # 1, not 2
```
Two independent releases from the exact same chart, different config,
zero manifest duplication — the whole point of this chapter, demonstrated.

**Upgrade and roll back**, mirroring Chapter 8's Deployment rollout but
for the whole chart:
```bash
helm upgrade my-release ./whoami-chart --set replicaCount=4
helm history my-release
kubectl get deploy my-release-whoami -o jsonpath='{.spec.replicas}'   # 4

helm rollback my-release 1
kubectl get deploy my-release-whoami -o jsonpath='{.spec.replicas}'   # back to 2
```

**Install a real third-party chart**, proving the ecosystem value:
```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
helm install my-redis bitnami/redis --set auth.enabled=false
kubectl get pods -l app.kubernetes.io/instance=my-redis
```
A production-grade Redis StatefulSet, Service, ConfigMap, and Secret
bundle, running from one command — compare the effort against
hand-authoring Chapter 21's StatefulSet yourself for something this
featureful.

Cleanup:
```bash
helm uninstall my-release staging-release my-redis
```

## Debugging Techniques

- **`helm install` fails with a YAML/templating error** — run `helm
  template` first, always; it renders locally without touching the
  cluster and shows you the exact malformed output before Helm even
  tries to apply it.
- **Values aren't taking effect** — check precedence: `--set` flags
  override a supplied `-f custom-values.yaml`, which overrides the
  chart's own `values.yaml` defaults; a common mistake is editing the
  wrong layer and wondering why nothing changed.
- **`helm upgrade` seems to have done nothing** — check `helm diff`
  (a popular plugin, not built-in by default) or re-run `helm template`
  with the new values and diff it manually against the previous render
  before assuming the upgrade is broken.
- **Uninstall doesn't remove everything** — some resources (notably
  CRDs some charts install) are deliberately **not** removed by `helm
  uninstall` by default, to avoid destroying data/config other releases
  might depend on — check the specific chart's documentation for
  post-uninstall cleanup steps if you need a truly clean slate.

## Best Practices

- Pin exact chart versions in any real deployment pipeline (Chapter 29)
  — `helm install chart-name` without a version can silently pull a newer
  chart version than what you tested.
- Keep environment-specific overrides in separate `values-<env>.yaml`
  files committed to git, applied via `-f`, rather than long strings of
  ad-hoc `--set` flags in a shell history somewhere — this is your
  actual environment configuration, treat it with the same care as code.
- Review third-party charts' `values.yaml` and generated manifests
  (`helm template`) before installing anything into a real cluster —
  you're trusting a stranger's templating logic with real workloads.
- Use `helm diff` (plugin) or a GitOps tool that shows a plan before
  apply (Chapter 29) for any production `helm upgrade` — never fly
  blind on a multi-object bundle change.

## Interview Questions

1. **What is a Helm chart, structurally?**
   A directory (or packaged archive) containing `Chart.yaml` (metadata),
   `values.yaml` (default configuration), and `templates/` (Go-template
   -parameterized Kubernetes YAML) — rendered and applied together as one
   unit.
2. **How is a Helm "release" different from just running `kubectl apply`
   on rendered YAML?** Helm tracks each install/upgrade as a numbered
   revision with its exact rendered manifest stored, enabling `helm
   rollback` to re-apply any previous revision precisely — bundle-level
   versioning that plain `kubectl apply` has no concept of.
3. **How can the same chart produce two independent, non-conflicting
   installs in the same cluster?** Via `{{ .Release.Name }}` (and similar
   built-ins) templated into object names — each `helm install <name>`
   uses a distinct release name, so object names never collide even from
   the identical chart source.
4. **What's the practical difference between a chart's own `version` and
   its `appVersion`?** `version` tracks the chart's own packaging/template
   revisions; `appVersion` just documents which version of the wrapped
   application it currently deploys — independent numbers, since chart
   template fixes don't necessarily mean the app itself changed.
5. **Why would a team install something like Redis or Prometheus via a
   public Helm chart instead of hand-writing the manifests?** A
   well-maintained chart already encodes tested, production-grade
   defaults (proper StatefulSet, resource limits, security settings) for
   a nontrivial multi-object application — reproducing that correctly by
   hand is substantial, error-prone effort with no real benefit over
   reusing a vetted chart.

## Mini Assignment

Take your `whoami-chart` and add a `templates/hpa.yaml` templating an
HPA (Chapter 23) with `minReplicas`/`maxReplicas`/target CPU utilization
all pulled from new `values.yaml` entries you define yourself. Install two
releases with different HPA bounds via `--set`, and confirm via `kubectl
get hpa` that each release's HPA reflects its own distinct values — no
hand-editing of rendered YAML.

## Lesson Summary

- Helm packages a set of Kubernetes objects into a versioned, templated
  chart, letting the same underlying manifests be parameterized per
  environment/install without copy-pasting YAML.
- `helm install`/`upgrade`/`rollback` give you Chapter 8's per-Deployment
  rollout/rollback concept, generalized to an entire multi-object bundle,
  tracked as numbered releases.
- The public chart ecosystem (Bitnami and others) lets you install
  production-grade third-party stacks (Redis, Postgres, Prometheus) with
  tested defaults in one command.
- `helm template` (render-only, no cluster touched) is your equivalent of
  `kubectl apply --dry-run` for validating chart output before installing.

---

### Before Chapter 27 (Monitoring with Prometheus and Grafana) — tell me:

1. What does `{{ .Release.Name }}` solve, specifically, that lets the
   same chart be installed twice in one cluster without conflict?
2. What's the practical difference between a chart's `version` and its
   `appVersion`?
3. From the Mini Assignment — did your two releases' HPAs actually show
   different `minReplicas`/`maxReplicas`/target values, confirmed via
   `kubectl get hpa`?
