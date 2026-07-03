# Chapter 22 — Resource Requests and Limits

## What it is

`resources.requests` and `resources.limits` (first previewed in Chapter
5) declare, per container, how much CPU and memory it needs (`requests`)
and the hard ceiling it may consume (`limits`). This chapter gives them
full treatment because Chapter 23's autoscaler is mathematically defined
in terms of `requests` — you cannot understand HPA correctly without
understanding this chapter first, which is exactly why the original
curriculum order (HPA before this) was corrected in this course's README.

## Why it exists

Docker lets you cap a single container's resources (`docker run
--cpus`/`--memory`), but that's a per-host, per-container decision you
make manually. Kubernetes needs this information **before** a container
even starts, for a much bigger reason: **the scheduler (Chapter 2) uses
`requests` to decide which node a Pod can even go on** — it will only
place a Pod on a node that has enough *unreserved* capacity to satisfy
its requests. Without this number, the scheduler has no principled way to
avoid overpacking a node, and the whole promise of "fleet-wide bin-packing
instead of you manually deciding which host runs what" (Chapter 1's core
value proposition) falls apart.

## When to use it

Every container, in every Pod, in any cluster shared by more than one
team or workload — which is to say, essentially always in production.
Skipping this is one of the most common and most damaging beginner
mistakes; an unbounded container can starve every other Pod on its node.

## Internal architecture

- **`requests`** is a **scheduling-time** guarantee: the scheduler sums
  all requests already placed on a node and only assigns a new Pod there
  if the node has enough *requested-but-possibly-unused* capacity left.
  This is why a node can look "80% requested" in `kubectl describe node`
  while actual live usage is much lower — requests are reservations, not
  real-time measurements.
- **`limits`** is a **runtime-enforcement** ceiling, enforced differently
  per resource type because CPU and memory behave fundamentally
  differently under contention:
  - **CPU is compressible** — a container hitting its CPU limit is
    **throttled** (temporarily denied scheduler time by the kernel's CFS
    bandwidth controller), not killed. It just runs slower.
  - **Memory is incompressible** — there's no "slow down" for memory; a
    container that exceeds its memory `limit` is **OOMKilled** by the
    kernel immediately, a hard termination, visible in `kubectl describe
    pod` as `OOMKilled` in the last-state reason.
- **Quality of Service (QoS) classes** — derived automatically from how
  you set requests/limits, and directly determining **eviction priority**
  under node pressure:
  - **`Guaranteed`** — every container's `requests == limits` for both
    CPU and memory. Evicted last under node resource pressure.
  - **`Burstable`** — requests set but lower than limits (or limits set
    on only some resources/containers). Evicted before `Guaranteed`.
  - **`BestEffort`** — no requests/limits set at all. Evicted first,
    with no scheduling protection whatsoever — this is the class every
    Pod without explicit values falls into, and it's rarely what you
    actually want in production.
- **Namespace-level `LimitRange`** (mentioned in Chapter 11) can set
  default requests/limits automatically for any container that omits
  them — a safety net against `BestEffort` Pods slipping in by accident.

## YAML Definition

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: resource-demo
spec:
  containers:
    - name: app
      image: polinux/stress
      command: ["stress"]
      args: ["--vm", "1", "--vm-bytes", "150M", "--vm-hang", "1"]
      resources:
        requests:
          cpu: "100m"
          memory: "64Mi"
        limits:
          cpu: "250m"
          memory: "128Mi"
```
- `cpu: "100m"` — CPU is measured in **millicores**; `1000m` = 1 full
  CPU core. `100m` requests one-tenth of a core.
- `memory: "64Mi"` — memory uses binary suffixes (`Mi` = mebibytes,
  `Gi` = gibibytes) — note `Mi` (1024-based) versus a plain `M`
  (1000-based decimal SI unit, also valid but different math); mixing
  these up is an easy, real source of "why is my number slightly off"
  confusion.
- `limits.cpu: "250m"` — this container may be throttled once it tries to
  use more than a quarter of a core, even if the node has spare CPU
  sitting idle.
- `limits.memory: "128Mi"` — this container is a deliberate setup to
  demonstrate OOMKill: `stress --vm-bytes 150M` tries to allocate 150Mi,
  exceeding the 128Mi limit.

## Hands-on Example

```bash
kubectl apply -f resource-demo.yaml
kubectl describe pod resource-demo | grep -A6 "State\|Last State"
```
Watch it get `OOMKilled` — the stress tool deliberately tries to allocate
more memory than the limit allows, and the kernel intervenes.

**See CPU throttling instead of killing** — same idea, CPU this time:
```yaml
# swap args to: ["--cpu", "2", "--timeout", "60s"]  (tries to burn 2 full cores)
# keep limits.cpu: "250m"
```
```bash
kubectl apply -f cpu-demo.yaml
kubectl top pod cpu-demo   # (requires metrics-server, installed in Ch. 23) usage caps near 250m, never exceeds it
```
The container isn't killed — it simply cannot get more than its
allotted CPU share, confirmed by usage never climbing past the limit
even though it's actively trying to.

**See scheduling refuse to overcommit** — request more than your Kind
node actually has:
```yaml
resources:
  requests:
    cpu: "100"    # 100 full cores — almost certainly more than your laptop has
```
```bash
kubectl apply -f oversized-pod.yaml
kubectl get pod oversized-pod    # stuck Pending
kubectl describe pod oversized-pod | grep -A3 Events
```
`Events` shows `Insufficient cpu` — this is the scheduler (Chapter 2),
not the kubelet, refusing to place it anywhere, confirming `requests` is
consulted *before* the Pod ever reaches a node.

**Check QoS class directly:**
```bash
kubectl get pod resource-demo -o jsonpath='{.status.qosClass}'
```

**Compare a Pod with no resources set at all:**
```bash
kubectl run no-resources --image=nginx
kubectl get pod no-resources -o jsonpath='{.status.qosClass}'   # BestEffort
```

Cleanup:
```bash
kubectl delete pod resource-demo cpu-demo oversized-pod no-resources
```

## Debugging Techniques

- **`OOMKilled` in `kubectl describe pod`** — the container exceeded its
  memory limit; either raise the limit (if the workload genuinely needs
  more) or fix an actual memory leak — don't just raise the limit
  reflexively without checking which case you're in.
- **App feels "randomly slow" under load, no errors, no restarts** —
  classic CPU throttling signature; check `kubectl describe pod` isn't
  the right tool here (throttling isn't an "event"), use `kubectl top
  pod` (Chapter 23's metrics-server) or Prometheus (Chapter 27)'s
  `container_cpu_cfs_throttled_seconds_total` metric to confirm.
- **Pod stuck `Pending`, `describe` shows `Insufficient cpu`/`Insufficient
  memory`** — a scheduling-time problem: no node has enough unreserved
  capacity for the requested amount; either lower the request, add
  node capacity, or check for other Pods over-requesting on that node.
- **Under real node memory pressure, some Pods get evicted even though
  they're within their own limits** — check their QoS class; `BestEffort`
  and low-priority `Burstable` Pods are evicted first to protect
  `Guaranteed` ones, entirely by design.

## Best Practices

- Always set both `requests` and `limits` for every container in
  production — never ship a `BestEffort` Pod deliberately.
- Set memory `requests == limits` for most workloads (`Guaranteed` or at
  least tight `Burstable`) — since memory limit breaches are fatal
  (OOMKill), you want little daylight between "what I asked for" and
  "what I'll be killed for exceeding."
- CPU `limits` are more debatable — some teams deliberately omit CPU
  limits (leaving only requests) to let a container burst into genuinely
  idle capacity, accepting the tradeoff that "noisy neighbor" throttling
  risk is lower priority than raw throughput; know this is a real,
  actively-debated production tradeoff, not a mistake either way.
- Base actual values on observed usage (Chapter 27's Prometheus/Grafana),
  not guesses — both over-requesting (wastes cluster capacity, costs
  money) and under-requesting (invites throttling/OOMKill) are real,
  common production mistakes.
- Set namespace-level `LimitRange` defaults as a safety net against
  Pods that forget to set resources entirely.

## Interview Questions

1. **What's the difference between `requests` and `limits`?**
   `requests` is what the scheduler guarantees is available when
   choosing a node (a reservation, checked at placement time); `limits`
   is the hard ceiling the container runtime/kernel enforces once
   running.
2. **What happens when a container exceeds its CPU limit versus its
   memory limit?** CPU is throttled (slowed down, not killed) since it's
   a compressible resource; memory triggers an OOMKill (hard termination)
   since memory can't be "slowed down" the same way.
3. **What are the three QoS classes and how are they determined?**
   `Guaranteed` (requests == limits for all resources/containers),
   `Burstable` (requests set, lower than limits, or partial), `BestEffort`
   (nothing set) — determined automatically from the Pod's resource
   configuration, and used to decide eviction order under node pressure.
4. **A Pod is stuck `Pending` with an `Insufficient cpu` event — what
   layer is failing, and why?** The scheduler, not the kubelet or
   container runtime — no node currently has enough unreserved CPU
   capacity to satisfy the Pod's `requests.cpu`.
5. **Why does Kubernetes need `requests` at all, when `limits` alone
   would still prevent one container from consuming unlimited
   resources?** Because scheduling decisions happen before any container
   runs, and need a stable, pre-agreed number to bin-pack against —
   `limits` alone says nothing about how much a node should reserve for
   this Pod versus others when deciding placement.

## Mini Assignment

Create three Pods with identical images but different resource
configurations designed to land in each QoS class deliberately
(`Guaranteed`, `Burstable`, `BestEffort`). Confirm each one's actual
`status.qosClass` matches your intent, then read (don't need to fully
implement) how `kubectl describe node` shows "Allocated resources" for
whichever node they land on, and reconcile that against the sum of your
Pods' `requests`.

## Lesson Summary

- `requests` drives scheduling (a reservation checked before placement);
  `limits` drives runtime enforcement (CPU throttled, memory OOMKilled)
  — genuinely different mechanisms for genuinely different resource
  types.
- QoS class (`Guaranteed`/`Burstable`/`BestEffort`), derived automatically
  from your requests/limits configuration, determines eviction priority
  under node pressure.
- Every production container should set both explicitly — this is not
  optional hygiene, it's the foundation the scheduler's entire
  bin-packing promise depends on, and it's the field Chapter 23's
  autoscaler is mathematically built on top of.

---

### Before Chapter 23 (Horizontal Pod Autoscaler) — tell me:

1. Why is memory limit enforcement (OOMKill) fundamentally different
   from CPU limit enforcement (throttling)?
2. A node shows 80% of its CPU "requested" in `kubectl describe node`,
   but actual usage is only 20%. Is that a contradiction? Why or why not?
3. From the Mini Assignment — did your three Pods land in the QoS classes
   you intended?
