# Chapter 28 — Logging

## What it is

Kubernetes-native logging is the practice of collecting every
container's stdout/stderr, cluster-wide, into one centralized, searchable
store — because `kubectl logs` (Chapter 4), while fine for one Pod right
now, has real limits: it only shows what the kubelet currently has
buffered, it's gone the moment a Pod is deleted, and there's no way to
search across hundreds of Pods at once.

## Why it exists

You already know `docker logs` for one container on one host. Chapter 4's
`kubectl logs` is the direct equivalent for one Pod — genuinely useful for
live debugging, but it inherits the same fundamental limitation Chapters
5/7/8 have been building toward all course: Pods are ephemeral and get
replaced constantly (rollouts, crashes, autoscaling). The moment a Pod is
deleted, `kubectl logs` has nothing left to show you (the `--previous`
flag from Chapter 5 only survives one restart, not a full deletion). A
production incident review needs to search logs across every replica of
a service, across every restart, over the last week — which requires logs
to be **shipped off the node and centralized** the instant they're
produced, not queried live from wherever the container happens to be
running right now.

## When to use it

Any cluster running more than a handful of Pods, or anything you'll ever
need to debug after the fact rather than while actively watching it live.
Essentially every real production cluster.

## Internal architecture

- **The 12-factor logging convention, which Kubernetes assumes**: your
  application should log to **stdout/stderr**, never to a file inside the
  container. The container runtime (containerd, Chapter 1) captures this
  output and writes it to a per-container log file on the **node's own
  disk** — this node-local file is exactly what `kubectl logs` reads from
  directly, which is precisely why it's gone once the Pod (and its node
  -local log file) is cleaned up.
- A **log-shipping agent** — commonly **Fluent Bit** or **Fluentd** —
  runs, almost always, as a **DaemonSet** (Chapter 20's pattern, again:
  one instance per node, reading every container's log files on that
  node's local disk) and forwards them to a centralized backend:
  Elasticsearch (the "EFK" stack: Elasticsearch, Fluentd, Kibana), Loki
  (paired with Grafana, Chapter 27's visualization layer, for a unified
  metrics+logs dashboarding experience), or a cloud provider's own log
  service (CloudWatch Logs, GCP Cloud Logging).
- Crucially, the shipping agent typically enriches each log line with
  **Kubernetes metadata** — Pod name, namespace, labels, container name —
  by querying the API server, so that even after a Pod is long gone, you
  can still filter centralized logs by `namespace=default AND
  label.app=whoami`, exactly the label-based query pattern from Chapter
  10, now applied to log search instead of object selection.
- **Structured (JSON) logging** — having your application emit JSON lines
  instead of arbitrary free text lets the shipping/storage layer index
  individual fields (e.g., `request_id`, `status_code`, `duration_ms`)
  for real queries, rather than only full-text search across opaque log
  lines.

## YAML Definition — Fluent Bit as a DaemonSet (conceptual shape)

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluent-bit
  namespace: logging
spec:
  selector:
    matchLabels: { app: fluent-bit }
  template:
    metadata:
      labels: { app: fluent-bit }
    spec:
      serviceAccountName: fluent-bit
      tolerations:
        - operator: Exists
      containers:
        - name: fluent-bit
          image: fluent/fluent-bit:latest
          volumeMounts:
            - name: varlog
              mountPath: /var/log
              readOnly: true
            - name: varlibcontainers
              mountPath: /var/lib/docker/containers
              readOnly: true
      volumes:
        - name: varlog
          hostPath:
            path: /var/log
        - name: varlibcontainers
          hostPath:
            path: /var/lib/docker/containers
```
- `kind: DaemonSet` — the direct real-world confirmation of Chapter 20's
  claim: log shippers are a canonical DaemonSet use case, one per node.
- `tolerations: [{operator: Exists}]` — tolerates *every* taint, including
  control-plane ones (Chapter 20's toleration concept) — you want log
  collection running literally everywhere, no exceptions.
- `volumes[].hostPath` — deliberately reads the **node's own local log
  directory** directly (Chapter 14's warning about `hostPath` for
  application data doesn't apply here — this is precisely the
  node-local, per-node-only use case `hostPath` is actually appropriate
  for, since a log-shipping DaemonSet Pod only ever needs *its own
  node's* logs, never another node's).
- `serviceAccountName` — needs RBAC (Chapter 24) permission to query the
  API server for Pod metadata to enrich each log line, another real
  -world confirmation of "your own workloads need scoped ServiceAccounts."

## Hands-on Example

**Prove the core limitation first** — the entire motivation for this
chapter:
```bash
kubectl run log-demo --image=busybox -- sh -c "for i in $(seq 1 5); do echo log line $i; sleep 1; done"
kubectl wait --for=jsonpath='{.status.phase}'=Succeeded pod/log-demo --timeout=30s
kubectl logs log-demo    # works — Pod still exists (Completed, not deleted)
kubectl delete pod log-demo
kubectl logs log-demo    # Error: pod not found — those 5 log lines are just GONE
```

**Install a real logging stack** — Loki + Grafana (paired with your
Chapter 27 Prometheus install, since Grafana already queries both from
one place):
```bash
helm repo add grafana https://grafana.github.io/helm-charts
helm install loki grafana/loki-stack \
  --set fluent-bit.enabled=true \
  --set grafana.enabled=false \
  -n monitoring
kubectl get pods -n monitoring -l app.kubernetes.io/name=fluent-bit -o wide
```
Confirm Fluent Bit really is one Pod per node — `-o wide` and compare
against `kubectl get nodes`, exactly per Chapter 20's DaemonSet guarantee.

**Query centralized logs through Grafana's Explore view**, using your
existing Chapter 27 Grafana install (add Loki as a data source pointing
at `http://loki.monitoring:3100`):
```
{namespace="default", app="whoami"}
```
This is **LogQL** (Loki's query language, deliberately similar to
PromQL) — filtering by exactly the Kubernetes labels Fluent Bit attached
automatically, across every Pod/restart that ever matched, not just
whichever one happens to be running right now.

**Re-run the earlier "deleted Pod" experiment, but through Loki instead
of `kubectl logs`:**
```bash
kubectl run log-demo2 --image=busybox --labels="app=log-demo2" -- sh -c "for i in $(seq 1 5); do echo log line $i; sleep 1; done"
kubectl wait --for=jsonpath='{.status.phase}'=Succeeded pod/log-demo2 --timeout=30s
kubectl delete pod log-demo2
```
In Grafana's Explore, query `{app="log-demo2"}` — the log lines are still
there, fully searchable, even though `kubectl logs log-demo2` would now
error exactly as before. This is the entire value proposition of this
chapter, proven directly.

Cleanup:
```bash
helm uninstall loki -n monitoring
```

## Debugging Techniques

- **Logs missing from the centralized store for a Pod you know
  produced output** — check whether the app is actually logging to
  stdout/stderr (not a file inside the container, which the shipping
  agent never sees) — this single mistake accounts for the overwhelming
  majority of "my logs aren't showing up" reports.
- **Fluent Bit Pod itself crashing/high resource usage** — check for a
  runaway log volume from one noisy Pod (e.g., a debug-level log left on
  in production, or a crash loop spamming stack traces) — the shipping
  agent processes whatever volume your workloads actually produce, and a
  sudden spike is a real signal worth investigating at the source, not
  just resourcing around.
- **Log timestamps look wrong / out of order across Pods** — check node
  clock synchronization (NTP) across the cluster; centralized log
  ordering depends on consistent node clocks in a way single-node
  `docker logs` never had to worry about.
- **Can find logs by Pod name but not by label** — verify the shipping
  agent's Kubernetes metadata enrichment is actually configured/working
  (check its own logs/config) — this is a separate, sometimes
  independently-broken piece from "logs are being shipped at all."

## Best Practices

- Log to stdout/stderr, always — never write application logs to a file
  inside the container; this is the one hard requirement everything else
  in this chapter depends on.
- Prefer structured (JSON) log output for anything you'll want to filter
  precisely later (status codes, durations, request IDs) — full-text
  search across free-form log lines doesn't scale to real incident
  investigation.
- Set log retention deliberately (cost vs. investigative value, exactly
  like Chapter 27's metrics retention tradeoff) — indefinite retention of
  every log line from every Pod gets expensive fast at real scale.
- Never log secrets/PII in plain text (a very real, very common
  compliance incident) — treat "what ends up in centralized,
  widely-searchable logs" with the same care as Chapter 13's Secrets
  discussion.

## Interview Questions

1. **Why isn't `kubectl logs` sufficient for production log management?**
   It only reads a log file local to whichever node the Pod is/was on,
   and that file (and thus the logs) disappears once the Pod is deleted
   — no cross-Pod search, no retention beyond the Pod's own lifetime.
2. **What's the standard architecture for cluster-wide log
   collection?** A log-shipping agent (Fluent Bit/Fluentd) deployed as a
   DaemonSet (one per node), reading each node's local container log
   files and forwarding them, enriched with Kubernetes metadata, to a
   centralized backend (Elasticsearch, Loki, or a cloud log service).
3. **Why must applications log to stdout/stderr rather than a file
   inside the container?** Because the entire collection pipeline is
   built around the container runtime capturing stdout/stderr to a
   node-local file the shipping agent reads — logs written to an
   arbitrary file inside the container are invisible to that pipeline
   entirely.
4. **How does a centralized logging backend know which Pod/namespace/
   labels a given log line came from, after the Pod itself no longer
   exists?** The shipping agent enriches each log line with Kubernetes
   metadata (Pod name, namespace, labels) at collection time, before
   forwarding it — that metadata is stored alongside the log line itself,
   independent of whether the source Pod still exists.
5. **Why is `hostPath` an appropriate volume type for a logging
   DaemonSet, when Chapter 14 warned against it for application data?**
   Because a logging DaemonSet Pod only ever needs its *own* node's local
   log directory — it's not meant to follow a Pod across nodes the way
   persistent application data must; the single-node-only nature that
   makes `hostPath` wrong for portable app data is exactly right for a
   strictly node-scoped log collector.

## Mini Assignment

Deploy two Pods with the same label but different containers, each
emitting distinctly recognizable log lines, then delete both Pods
entirely. Using only Grafana's Loki Explore view (not `kubectl logs`),
write one LogQL query that returns both Pods' log lines together by
their shared label, and a second query that isolates just one of them by
adding a more specific label filter. Confirm both queries return results
even though neither Pod still exists in the cluster.

## Lesson Summary

- `kubectl logs` reads a node-local file tied to a specific Pod's
  lifetime — fine for live debugging, useless once a Pod is gone, which
  is constantly, by Kubernetes' own design (Chapters 5-8).
- Centralized logging ships every container's stdout/stderr off-node via
  a DaemonSet-based agent (Fluent Bit/Fluentd) — another concrete,
  real-world confirmation of Chapter 20's "one per node" pattern.
- Logs are enriched with Kubernetes metadata (Pod, namespace, labels) at
  collection time, making them searchable by label long after the source
  Pod is deleted — you proved this directly against `kubectl logs`'s
  failure in the same scenario.
- Structured logging and stdout/stderr-only output are the two
  non-negotiable application-level requirements this entire pipeline
  depends on.

---

### Before Chapter 29 (CI/CD with Kubernetes) — tell me:

1. Why does deleting a Pod make its logs permanently unavailable via
   `kubectl logs`, specifically?
2. Why must a log-shipping agent run as a DaemonSet rather than, say, a
   single centralized Deployment replica?
3. From the Mini Assignment — did your label-based LogQL query
   successfully return logs from a Pod that no longer exists in the
   cluster?
