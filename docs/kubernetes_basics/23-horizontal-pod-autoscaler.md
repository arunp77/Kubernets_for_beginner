# Chapter 23 — Horizontal Pod Autoscaler

## What it is

The **Horizontal Pod Autoscaler (HPA)** is a controller that automatically
adjusts a Deployment's (or StatefulSet's) `replicas` count based on
observed metrics — most commonly CPU or memory utilization, expressed as
a **percentage of the resources `requests`** you set in Chapter 22 (which
is exactly why this course teaches Chapter 22 first).

## Why it exists

You've been setting `replicas` by hand this whole course — a fixed number
you chose ahead of time. Real traffic isn't fixed: a nightly batch spike,
a marketing campaign, a regional daytime traffic curve. Manually watching
dashboards and running `kubectl scale` in response is exactly the kind of
constant human-in-the-loop toil Kubernetes exists to eliminate (Chapter
1). HPA closes that loop automatically: it's a controller (the same
pattern as everything else in this course) whose desired state is
"replicas such that average utilization stays near this target," reconciled
continuously against live metrics.

## When to use it

Any workload with genuinely variable load and a metric that correlates
with load (CPU usage for compute-bound APIs, memory for cache-like
workloads, or custom metrics like queue depth/requests-per-second for more
precise scaling). Not useful for workloads with a fixed, predictable load
shape, or where scaling reaction time (HPA reacts over tens of seconds to
minutes, not instantly) is too slow for the traffic pattern.

## Internal architecture

- HPA needs a metrics source. The baseline is the **metrics-server**
  add-on (not installed by default on Kind — you'll install it in this
  chapter's lab), which scrapes each kubelet for basic CPU/memory
  usage and exposes it via the **Metrics API** (a separate, lightweight
  API the HPA controller queries — distinct from Prometheus/Grafana's
  full monitoring stack in Chapter 27, which can *also* feed HPA via
  the **Custom Metrics API** for more sophisticated scaling signals).
- On each reconciliation tick (every 15 seconds by default), the HPA
  controller: fetches current average utilization across all Pods
  matched by the target Deployment's selector, compares it to the
  configured target, and computes a new desired replica count via
  roughly:
  `desiredReplicas = ceil(currentReplicas * (currentMetricValue / targetMetricValue))`
- Crucially, **"currentMetricValue" for CPU is expressed as a percentage
  of each Pod's `requests.cpu`** (Chapter 22) — not an absolute number,
  and not related to `limits` at all. A Pod with no CPU `request` set has
  **no defined utilization percentage** and HPA cannot make a scaling
  decision from it — this is the concrete, mechanical reason Chapter 22
  had to come first.
- **Stabilization windows** and a built-in **cooldown** prevent
  "flapping" — rapidly scaling up and down in response to noisy,
  short-lived metric spikes; `behavior.scaleDown.stabilizationWindowSeconds`
  (and the equivalent for scale-up) let you tune how conservative the
  autoscaler is about each direction independently — scaling up fast
  (protect users) but scaling down slowly (avoid thrashing) is the common
  production posture.
- HPA only changes `spec.replicas` on the target — it does not create
  Pods itself; the underlying Deployment/ReplicaSet (Chapters 7-8) still
  does the actual Pod creation/deletion work, exactly per their own
  reconciliation loops. HPA is a controller that drives *another*
  controller's input, one more layer in the same pattern.

## YAML Definition

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: whoami-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: whoami
  minReplicas: 2
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 50
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 120
    scaleUp:
      stabilizationWindowSeconds: 0
```
- `apiVersion: autoscaling/v2` — its own API group; `v2` (versus the
  older `v1`) is what supports multiple/custom metrics and the
  `behavior` block — always prefer `v2` today.
- `spec.scaleTargetRef` — which object HPA drives; note it references the
  target by `apiVersion`/`kind`/`name`, not a label selector this time —
  a direct reference is appropriate here since an HPA genuinely does only
  make sense paired with one specific target.
- `minReplicas`/`maxReplicas` — hard floor and ceiling; HPA will never
  scale outside this range regardless of metrics — your safety rails
  against both "scaled to zero and can't serve any traffic" and "scaled
  to 500 and blew the cloud budget."
- `metrics[].resource.target.averageUtilization: 50` — target 50% of
  each Pod's CPU **request** (Chapter 22), averaged across all Pods.
- `behavior.scaleDown.stabilizationWindowSeconds: 120` — wait 2 minutes
  of sustained lower usage before actually scaling down, damping
  flapping; `scaleUp` left at `0` here for a fast reaction to genuine
  load spikes.

## Hands-on Example

```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```
(On Kind specifically, metrics-server commonly needs
`--kubelet-insecure-tls` added to its args due to Kind's self-signed
certs — patch it if `kubectl top` below fails with a TLS error.)
```bash
kubectl wait --for=condition=available deployment/metrics-server -n kube-system --timeout=90s
kubectl top nodes
kubectl top pods
```
Confirm `kubectl top` returns real numbers — this confirms metrics-server
is working *before* layering HPA on top of it.

**Deploy a target with proper requests set** (Chapter 22's lesson,
applied):
```yaml
# whoami-deploy.yaml, containers[0].resources:
resources:
  requests: { cpu: "100m", memory: "64Mi" }
  limits: { cpu: "200m", memory: "128Mi" }
```
```bash
kubectl apply -f whoami-deploy.yaml
kubectl apply -f whoami-hpa.yaml
kubectl get hpa whoami-hpa -w
```
`TARGETS` column shows `<unknown>/50%` briefly, then real numbers once
metrics-server has a data point.

**Generate real CPU load and watch it scale up live:**
```bash
kubectl run load-generator --rm -it --image=busybox -- sh -c \
  "while true; do wget -q -O- http://whoami-svc; done"
```
In another terminal:
```bash
kubectl get hpa whoami-hpa -w
kubectl get pods -l app=whoami -w
```
Watch `TARGETS` climb above 50%, then `REPLICAS` increase — the
Deployment controller (Chapter 8) creating the actual new Pods that HPA's
decision requested.

**Stop the load and watch the stabilization window in action:**
```bash
# Ctrl+C the load-generator
kubectl get hpa whoami-hpa -w
```
Notice replicas *don't* drop immediately — they hold for your configured
`stabilizationWindowSeconds` (120s here) of sustained low usage before
scaling back down, confirmed by watching the timestamp gap.

**Prove the requests-dependency directly** — remove `resources.requests`
from the Deployment and reapply:
```bash
kubectl get hpa whoami-hpa
```
`TARGETS` shows `<unknown>/50%` permanently — HPA has no percentage to
compute without a `requests.cpu` baseline, exactly as the Theory section
predicted.

Cleanup:
```bash
kubectl delete -f whoami-hpa.yaml -f whoami-deploy.yaml
```

## Debugging Techniques

- **HPA shows `<unknown>` in `TARGETS` forever** — check, in order:
  metrics-server actually running and healthy (`kubectl top pods` working
  at all), then whether the target Deployment's containers have
  `resources.requests` set (the single most common cause).
- **HPA not scaling despite clearly high load** — check
  `minReplicas`/`maxReplicas` bounds aren't already saturated, and check
  `kubectl describe hpa` for recent scaling events/reasons.
- **Replicas flapping up and down repeatedly** — tune
  `behavior.scaleDown.stabilizationWindowSeconds` upward; this is exactly
  the damping mechanism designed for this symptom.
- **Scaled up, but new Pods aren't helping (still overloaded)** — check
  whether the actual bottleneck is downstream (a database, an external
  API) rather than CPU on these Pods — HPA can only address the
  specific metric it's watching, not the workload's true bottleneck.

## Best Practices

- Always set `resources.requests` deliberately (Chapter 22) before adding
  HPA — this isn't optional, HPA's CPU/memory scaling is mathematically
  undefined without it.
- Set `minReplicas` to at least 2 for anything user-facing, for basic
  availability even at zero load — never let HPA's floor be `1` for a
  production service where a single Pod restart would otherwise cause a
  brief total outage.
- Tune scale-up and scale-down `stabilizationWindowSeconds` asymmetrically
  — fast up (protect users during a spike), slow down (avoid cost/thrash
  from reacting to noise).
- For workloads with a scaling signal that isn't CPU/memory (queue depth,
  requests-per-second), invest in **custom metrics** via Prometheus
  adapter (Chapter 27) rather than forcing CPU utilization to be a proxy
  for something it doesn't actually measure well.

## Interview Questions

1. **What formula, roughly, does HPA use to decide the new replica
   count?** `desiredReplicas = ceil(currentReplicas * (currentMetricValue
   / targetMetricValue))` — scaling proportionally to how far current
   utilization is from the target.
2. **Why must a container have `resources.requests.cpu` set for HPA's
   CPU-based scaling to work at all?** Because CPU utilization for HPA is
   expressed as a percentage of `requests.cpu`, not an absolute value or
   a percentage of `limits` — with no request set, there's no baseline to
   compute a percentage against.
3. **What component actually creates the new Pods when HPA decides to
   scale up?** HPA only updates the target Deployment/StatefulSet's
   `replicas` field; the Deployment's own ReplicaSet controller (Chapters
   7-8) does the actual Pod creation, exactly per its normal
   reconciliation loop.
4. **How does HPA avoid rapidly flapping replica counts up and down on
   noisy metrics?** `behavior.scaleUp`/`scaleDown.stabilizationWindowSeconds`
   require metrics to stay past the threshold for a configured window
   before actually acting, damping short-lived spikes/dips.
5. **What's the difference between the basic Metrics API
   (metrics-server) and Custom Metrics used with HPA?** metrics-server
   provides only basic CPU/memory usage; Custom Metrics (commonly via a
   Prometheus adapter, Chapter 27) let HPA scale on arbitrary
   application-specific signals like queue depth or requests-per-second.

## Mini Assignment

Configure an HPA targeting memory utilization instead of CPU
(`resource.name: memory`), generate memory pressure on the target Pods
(e.g., using `polinux/stress` with `--vm-bytes` from Chapter 22, scaled to
approach but not exceed the container's memory limit), and confirm HPA
scales up in response. Document what target percentage you chose and why,
given memory's OOMKill-on-exceed behavior from Chapter 22 — how does that
change your safety margin thinking versus CPU-based scaling?

## Lesson Summary

- HPA is a controller that adjusts `replicas` automatically based on
  observed metrics compared to a target — the same reconciliation-loop
  pattern as every other controller in this course, applied to scaling
  decisions.
- CPU/memory-based scaling is mathematically defined as a percentage of
  `resources.requests` (Chapter 22) — without requests set, HPA has no
  baseline and shows `<unknown>` targets forever.
- `minReplicas`/`maxReplicas` are hard safety rails; stabilization
  windows prevent flapping from noisy short-term metric spikes.
- HPA only decides *how many* replicas should exist — the actual Pod
  creation/deletion is still done by the target's own controller.

---

### Before Chapter 24 (RBAC) — tell me:

1. Why does removing `resources.requests` from a Deployment break HPA's
   CPU-based scaling specifically, mechanically?
2. What's the practical reason to set scale-up stabilization faster than
   scale-down stabilization?
3. From the Mini Assignment — what target memory percentage did you
   choose, and how did OOMKill risk factor into that choice?
