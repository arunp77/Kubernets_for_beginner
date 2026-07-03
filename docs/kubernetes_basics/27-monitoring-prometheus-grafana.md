# Chapter 27 — Monitoring with Prometheus and Grafana

## What it is

**Prometheus** is a time-series metrics database that **pulls** (scrapes)
metrics from targets on a schedule and lets you query them with its own
query language, **PromQL**. **Grafana** is a visualization layer that
queries Prometheus (and other data sources) to build dashboards and
alerts. Together they form the standard Kubernetes-native monitoring
stack — and, not coincidentally, the same data Chapter 23's HPA can
consume via a Custom Metrics adapter for scaling decisions beyond basic
CPU/memory.

## Why it exists

Chapter 23 introduced metrics-server, a bare-bones "current CPU/memory
usage" snapshot with no history — enough for HPA math, useless for
answering "was memory usage trending up all week before this incident?"
or "what does p99 latency look like across the last month?" Production
operations needs **historical, queryable, alertable** metrics — from
infrastructure (node CPU, per-Pod memory) all the way up to your
application's own business metrics (requests per second, queue depth,
error rate). Prometheus is built specifically for this, and — crucially
for a Kubernetes course — it's built around the exact same
service-discovery pattern Kubernetes itself uses: Prometheus **discovers
scrape targets by watching the Kubernetes API** (Pods, Services, with
specific annotations/labels), the same "watch the API server, react"
pattern from Chapter 2, applied to "who do I collect metrics from" rather
than "who needs scheduling."

## When to use it

Every real cluster running anything you care about operationally. Even a
side project benefits enormously from being able to answer "what changed
right before this broke" with an actual graph instead of a guess.

## Internal architecture

- **Pull, not push.** Unlike some metrics systems where applications push
  data outward, Prometheus's server actively **scrapes** an HTTP endpoint
  (conventionally `/metrics`) on each target at a regular interval. Your
  application (or an **exporter** — a small sidecar/separate process that
  translates some other system's stats into Prometheus's text format)
  must expose metrics in Prometheus's plain-text exposition format for
  this to work.
- **Service discovery via Kubernetes** — Prometheus's Kubernetes SD
  config watches Pods/Services/Endpoints (exactly like kube-proxy does,
  Chapter 9) and automatically adds/removes scrape targets as Pods come
  and go — you don't hand-maintain a target list; this is the direct
  payoff of everything being labeled and API-discoverable since Chapter
  10.
- **node-exporter** — deployed as a **DaemonSet** (Chapter 20's exact use
  case: one per node, exposing that node's own CPU/memory/disk/network
  stats) — this is the concrete real-world DaemonSet example Chapter 20
  promised you'd meet again.
- **kube-state-metrics** — a separate exporter that watches the API
  server and turns *object state* (how many Pods are `Pending`, how many
  Deployment replicas are unavailable, PVC binding status) into
  Prometheus metrics — this is what lets you build dashboards/alerts
  about the *cluster's own health*, not just node/app resource usage.
- **PromQL** — Prometheus's query language; metrics are time series
  identified by a name plus key/value **labels** (yes, the same
  labeling concept from Chapter 10, just Prometheus's own separate label
  system on metrics rather than Kubernetes objects) — e.g.,
  `container_cpu_usage_seconds_total{pod="whoami-abc123", namespace="default"}`.
- **Grafana** connects to Prometheus as a "data source" and renders
  PromQL query results as dashboards; it also supports **alerting rules**
  that fire notifications (Slack, email, PagerDuty) when a query crosses
  a threshold for a sustained period.

## YAML Definition — installed via Helm (Chapter 26, in practice)

Hand-authoring the full Prometheus + Grafana + node-exporter +
kube-state-metrics stack from scratch is exactly the kind of substantial,
well-solved-already problem Chapter 26 said to reach for a chart for —
the **kube-prometheus-stack** community Helm chart bundles all of it,
pre-wired together:

```yaml
# values-override.yaml — the parts worth understanding, not memorizing every field
grafana:
  adminPassword: "demo-password-change-me"
  service:
    type: ClusterIP
prometheus:
  prometheusSpec:
    retention: 15d
    resources:
      requests: { cpu: 200m, memory: 512Mi }
      limits: { cpu: 500m, memory: 1Gi }
alertmanager:
  enabled: true
```
- `grafana.adminPassword` — should really come from a Secret (Chapter
  13) referenced via the chart's `admin.existingSecret` option in any
  real deployment, not a plain value in `values.yaml`, exactly per
  Chapter 13's "never commit real secret values" guidance.
- `prometheus.prometheusSpec.retention` — how long historical metrics are
  kept — a real cost/usefulness tradeoff, since Prometheus stores this on
  a PersistentVolume (Chapter 14) whose size scales with retention ×
  metric cardinality.
- `prometheus.prometheusSpec.resources` — Chapter 22's lesson applied to
  Prometheus itself: it's a real workload that needs its own
  requests/limits like anything else, and it can be a genuinely heavy one
  at scale.
- `alertmanager.enabled` — a companion component (bundled in this chart)
  that Prometheus sends firing alerts to, which then handles
  routing/deduplication/notification — worth knowing exists even without
  full depth here.

A **ServiceMonitor** (a custom object this chart's Prometheus Operator
introduces) is how you tell Prometheus about *your own* application's
metrics endpoint, declaratively:
```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: whoami-monitor
spec:
  selector:
    matchLabels:
      app: whoami
  endpoints:
    - port: metrics
      interval: 15s
```
`spec.selector` — Chapter 10's pattern again — targets any Service
labeled `app: whoami`; Prometheus's Operator watches for ServiceMonitor
objects and auto-configures scraping accordingly, no manual Prometheus
config file editing required.

## Hands-on Example

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm install monitoring prometheus-community/kube-prometheus-stack \
  -f values-override.yaml --create-namespace -n monitoring
kubectl get pods -n monitoring
```
Notice, among the Pods created: a `node-exporter` **DaemonSet** (one Pod
per node — confirm with `-o wide`, directly recalling Chapter 20), a
`kube-state-metrics` Deployment, the Prometheus server itself (often as a
StatefulSet — Chapter 21's pattern, since it needs persistent storage for
its time-series database and a stable identity), and Grafana.

**Access Grafana and explore real cluster data:**
```bash
kubectl port-forward -n monitoring svc/monitoring-grafana 3000:80
```
Open `localhost:3000` (default user `admin`, password from your
override), and open the pre-built "Kubernetes / Compute Resources /
Namespace (Pods)" dashboard — real CPU/memory data for every Pod in your
cluster, built from `node-exporter`+`kube-state-metrics`, zero manual
setup.

**Query PromQL directly**, to see what's actually underneath the
dashboard:
```bash
kubectl port-forward -n monitoring svc/monitoring-kube-prometheus-prometheus 9090:9090
```
Open `localhost:9090`, and try:
```promql
sum(rate(container_cpu_usage_seconds_total{namespace="default"}[5m])) by (pod)
kube_deployment_status_replicas_unavailable
```
The second query comes straight from **kube-state-metrics** — proving
cluster *object* health (not just resource usage) is queryable the exact
same way.

**Wire up your own app** — expose a `/metrics` endpoint on `whoami`
(traefik/whoami doesn't natively expose Prometheus metrics, so
conceptually: any real FastAPI app would use a library like
`prometheus-fastapi-instrumentator` to expose one), then apply a
ServiceMonitor selecting its Service, and confirm it appears as a new
scrape target:
```bash
kubectl apply -f whoami-servicemonitor.yaml
# in Prometheus UI: Status > Targets — confirm the new target appears and is "UP"
```

Cleanup:
```bash
helm uninstall monitoring -n monitoring
kubectl delete namespace monitoring
```

## Debugging Techniques

- **A target shows `DOWN` in Prometheus's Targets page** — check the
  target Pod/Service is actually reachable and genuinely serving
  `/metrics` (`kubectl exec` into a debug Pod and `curl` it directly,
  exactly like Chapter 9's Endpoints debugging) before assuming
  Prometheus configuration is wrong.
- **ServiceMonitor applied, but Prometheus never scrapes it** — check the
  ServiceMonitor's `selector` actually matches the target Service's
  labels (Chapter 10, again), and confirm the Prometheus Operator's own
  `serviceMonitorSelector` (a chart-level setting) isn't itself scoped to
  ignore ServiceMonitors outside certain namespaces/labels — a common,
  confusing "why isn't my ServiceMonitor picked up" gotcha specific to
  this Operator pattern.
- **Grafana dashboard shows "No Data"** — check the underlying PromQL
  query directly in Prometheus's own UI first, isolating whether the
  problem is "no data exists" (a scraping/target problem) versus "data
  exists, Grafana's querying it wrong" (a dashboard/data-source
  configuration problem).
- **Prometheus Pod itself gets OOMKilled** — a very real, common
  production issue as metric cardinality (unique label combinations)
  grows; check `resources.limits` (Chapter 22) against actual usage, and
  investigate high-cardinality labels (like raw user IDs) as a root cause
  if usage keeps climbing.

## Best Practices

- Set real `resources.requests/limits` on Prometheus and Grafana
  themselves — they are real, sometimes resource-hungry workloads, not
  exempt from Chapter 22's lesson just because they're "infrastructure."
- Avoid high-cardinality labels (anything with unbounded unique values,
  like raw request IDs or user IDs) on your own application's exposed
  metrics — this is the single most common cause of a Prometheus
  instance's memory usage growing unmanageably over time.
- Alert on **symptoms users would notice** (error rate, latency) as the
  primary signal, and use resource-level metrics (CPU, memory) mainly for
  root-causing, not as your only alerts — a healthy-looking CPU graph
  doesn't mean users are happy.
- Set a deliberate metrics retention period balancing cost (storage) vs.
  usefulness (historical incident analysis) — 15 days is a common
  starting point, tuned from there based on real need.

## Interview Questions

1. **Does Prometheus push or pull metrics?**
   Pull — it scrapes a `/metrics` HTTP endpoint on each target at a
   configured interval; applications/exporters must expose metrics in
   Prometheus's text format for this to work.
2. **How does Prometheus know which Pods/Services to scrape in a
   Kubernetes cluster?** Via Kubernetes service discovery — it watches
   the API server (the same "watch and react" pattern as every
   controller in this course) for Pods/Services/Endpoints, often
   configured declaratively via ServiceMonitor objects that select
   targets by label, exactly like Chapter 10's selector pattern.
3. **What are node-exporter and kube-state-metrics, and how do they
   differ?** node-exporter (deployed as a DaemonSet, one per node)
   exposes node-level OS resource metrics; kube-state-metrics exposes
   Kubernetes *object state* metrics (replica counts, Pod phases, PVC
   binding status) by watching the API server — one covers
   infrastructure, the other covers cluster/object health.
4. **What's a common cause of Prometheus itself running out of
   memory?** High-cardinality metric labels (unbounded unique values like
   raw user/request IDs) causing the number of distinct time series to
   grow unmanageably.
5. **How would you get your own FastAPI application's custom metrics into
   Prometheus?** Expose a `/metrics` endpoint in the app (e.g., via a
   Prometheus client library), then create a ServiceMonitor whose
   selector matches the app's Service, letting the Prometheus Operator
   auto-configure scraping with no manual Prometheus config editing.

## Mini Assignment

Using the installed kube-prometheus-stack, write a PromQL query that
shows total memory usage per namespace across your cluster, and a second
query showing how many Pods are currently *not* in the `Running` phase
(hint: this comes from kube-state-metrics, not node-exporter). Build a
one-panel Grafana dashboard from each query and take a screenshot-worth of
notes on what real data you saw — no solution query given here, work it
out from the metric names visible in Prometheus's own metric browser.

## Lesson Summary

- Prometheus pulls metrics via HTTP scraping, using Kubernetes-native
  service discovery to find targets — the same watch-the-API-server
  pattern as every controller in this course, applied to metrics
  collection.
- node-exporter (a DaemonSet, Chapter 20's real-world example) and
  kube-state-metrics cover infrastructure and cluster-object health
  respectively; your own app's custom metrics plug in via a
  ServiceMonitor.
- Grafana visualizes and alerts on Prometheus data; the whole stack is,
  in practice, installed via Helm (Chapter 26) rather than hand-authored.
- High-cardinality labels are the classic Prometheus scaling failure
  mode — worth designing your own app's metrics around from the start.

---

### Before Chapter 28 (Logging) — tell me:

1. Why does Prometheus need Kubernetes service discovery at all, instead
   of a static list of targets?
2. What's the difference between what node-exporter and
   kube-state-metrics each expose?
3. From the Mini Assignment — what PromQL query did you land on for
   "Pods not currently Running," and which metric did it come from?
