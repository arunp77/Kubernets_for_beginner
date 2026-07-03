# Chapter 20 — DaemonSets

## What it is

A **DaemonSet** ensures **exactly one copy of a Pod runs on every node**
(or every node matching a selector) in the cluster — not N replicas
spread arbitrarily (ReplicaSet/Deployment), but precisely one per node,
automatically added to new nodes and removed as nodes leave.

## Why it exists

Recall Chapter 2: every node already runs certain per-node agents —
kubelet, kube-proxy, the container runtime. Those are baked into the node
image/Kind's node stack, but the same *pattern* — "one instance of this
software, on every node, always" — is exactly what you need for your own
infrastructure-level concerns: a log-shipping agent that needs to read
every node's local log files, a metrics-collection agent (like
node-exporter, which you'll actually deploy this way in Chapter 27), a
network plugin's per-node component, or a security/monitoring agent. A
Deployment can't express "one per node" — its replica count is a fixed
number independent of cluster size; a DaemonSet's "replica count" is
implicitly "however many nodes currently qualify," recalculated
automatically as nodes are added or removed.

## When to use it

Node-level infrastructure agents: log collectors (Fluentd/Fluent Bit,
Chapter 28), metrics exporters (node-exporter, Chapter 27), CNI network
plugins, node-level security scanners. Essentially never for your actual
application workloads — those want a Deployment, scaled independently of
node count.

## Internal architecture

- The DaemonSet controller watches for **nodes**, not just Pods — on
  every node-add event, it creates one matching Pod there; on node
  removal, that Pod is naturally gone with it (no rescheduling elsewhere,
  since "elsewhere" already has its own copy).
- Unlike ReplicaSet/Deployment Pods, DaemonSet Pods are typically **not**
  meant to be moved by the scheduler in the usual sense — they're
  pinned, one per qualifying node, by design (the DaemonSet controller
  itself assigns the node, historically bypassing the ordinary scheduler
  entirely in some configurations, though modern Kubernetes does route
  DaemonSet Pods through the scheduler too, just with an implicit
  node-affinity rule per Pod tying it to its specific node).
- **`spec.selector`+`nodeSelector`/`affinity`** (on the Pod template)
  lets you scope a DaemonSet to only *some* nodes — e.g., a GPU-monitoring
  DaemonSet that should only run on nodes labeled `gpu: true`, not the
  whole cluster.
- DaemonSet Pods commonly tolerate **taints** that would normally repel
  ordinary Pods (e.g., a node marked "not ready" or "control-plane only")
  — you often *want* your logging/monitoring agent running even on
  otherwise-restricted nodes, which is why `tolerations` show up in real
  DaemonSet manifests far more often than in typical Deployments.

## YAML Definition

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: node-logger
  labels:
    app: node-logger
spec:
  selector:
    matchLabels:
      app: node-logger
  template:
    metadata:
      labels:
        app: node-logger
    spec:
      tolerations:
        - key: node-role.kubernetes.io/control-plane
          effect: NoSchedule
      containers:
        - name: logger
          image: busybox
          command: ["sh", "-c", "while true; do echo $(hostname) alive at $(date); sleep 10; done"]
          resources:
            requests: { cpu: "50m", memory: "32Mi" }
            limits: { cpu: "100m", memory: "64Mi" }
```
- `spec.selector`/`template` — identical shape to ReplicaSet/Deployment
  (Chapters 7-8); a DaemonSet is, structurally, "yet another Pod-template
  based controller," differing only in *how* it decides Pod count (one
  per node, not a fixed number).
- `tolerations` — explicitly allows this Pod to be scheduled onto nodes
  the control plane normally reserves for its own components (recall
  Chapter 2's control-plane node) — without this, a DaemonSet meant to
  monitor *every* node, including control-plane ones, would silently
  skip them.
- No `replicas` field at all — notice its absence; that's the whole point,
  count is implicit from node count.

## Hands-on Example

Using your 3-node `masterclass` cluster from Chapter 3:

```bash
kubectl apply -f node-logger-ds.yaml
kubectl get daemonset node-logger
kubectl get pods -l app=node-logger -o wide
```
Confirm exactly one Pod per node, including (thanks to the toleration)
the control-plane node — `DESIRED`/`CURRENT`/`READY` in `get daemonset`
should all equal your node count.

**Prove it tracks cluster size automatically** — add a node and watch
without touching the DaemonSet at all:
```bash
docker exec masterclass-worker true   # (nodes are already up; to truly test, you'd add a node to the kind config and recreate,
                                        #  or use `kind` multi-node scaling patterns — conceptually: new node joins, DaemonSet
                                        #  controller sees it, schedules a Pod there automatically, no manifest change needed)
kubectl get pods -l app=node-logger -o wide -w
```

**Compare against a Deployment on purpose** — set `replicas: 3` on a
Deployment with the same Pod template and confirm all 3 could actually
land on the *same* node (no guarantee of spread), unlike the DaemonSet's
guaranteed one-per-node placement:
```bash
kubectl create deployment not-a-daemon --image=busybox --replicas=3 -- sh -c "sleep 3600"
kubectl get pods -l app=not-a-daemon -o wide
```

**Scope to a subset of nodes:**
```bash
kubectl label node masterclass-worker gpu=true
```
```yaml
# add to node-logger-ds.yaml's template.spec:
      nodeSelector:
        gpu: "true"
```
```bash
kubectl apply -f node-logger-ds.yaml
kubectl get pods -l app=node-logger -o wide
```
Now only the labeled node runs a Pod — proving DaemonSets aren't
literally "every node," but "every node matching the template's
placement constraints."

Cleanup:
```bash
kubectl delete -f node-logger-ds.yaml
kubectl delete deployment not-a-daemon
kubectl label node masterclass-worker gpu-
```

## Debugging Techniques

- **Fewer Pods than nodes** — check `nodeSelector`/`affinity`/
  `tolerations` first; a DaemonSet that's "missing" a node is almost
  always intentionally or accidentally scoped away from it, not failing
  to schedule.
- **`kubectl describe daemonset <name>`** — shows
  `Desired`/`Current`/`Ready`/`Up-to-date`/`Available` node counts side
  by side, exactly like Chapter 7's ReplicaSet columns, adapted to "per
  node" semantics.
- **DaemonSet Pod missing specifically from the control-plane node** —
  check for a missing toleration matching that node's taints
  (`kubectl describe node <control-plane-node> | grep Taints`).
- **Rolling out a DaemonSet update seems to "skip" nodes** — DaemonSets
  support the same `RollingUpdate` strategy concept as Deployments
  (`maxUnavailable`, no `maxSurge` — there's nowhere to "surge" to, since
  it's always exactly one Pod per node); check `kubectl rollout status
  daemonset/<name>` exactly as you would for a Deployment.

## Best Practices

- Reserve DaemonSets strictly for genuine per-node infrastructure
  concerns — using one for application workloads (assuming "one per node"
  happens to match your desired scale) couples your app's scaling to
  cluster node count, which is almost never actually what you want (use
  HPA, Chapter 23, for real application scaling).
- Set conservative `resources.requests/limits` on DaemonSet Pods — they
  run on literally every node, so an oversized DaemonSet Pod steals
  capacity from every single node in the cluster simultaneously, a much
  larger blast radius than an oversized Deployment replica.
- Add tolerations deliberately when you genuinely need coverage on
  control-plane or otherwise-tainted nodes (monitoring/logging usually
  does; most application DaemonSets, if you ever have one, usually
  shouldn't).

## Interview Questions

1. **What does a DaemonSet guarantee, precisely?**
   Exactly one Pod per node matching its placement constraints (or all
   nodes, if unconstrained) — automatically added on new nodes and
   removed when nodes leave, with no fixed replica count to configure.
2. **Why can't a Deployment achieve the same "one per node" behavior
   reliably?** A Deployment's replica count is a fixed number independent
   of cluster size, and the scheduler is free to place multiple replicas
   on the same node or leave some nodes with none — it optimizes for
   "N replicas exist somewhere," not "exactly one per node."
3. **Give three real examples of workloads that belong in a DaemonSet.**
   Log-shipping agents, metrics exporters (e.g., node-exporter), CNI
   network plugin components — all fundamentally per-node infrastructure
   concerns.
4. **Why do DaemonSet Pods often need `tolerations` that ordinary
   application Pods don't?** Because you frequently want infrastructure
   -level DaemonSets (monitoring, logging) to run even on nodes reserved
   for control-plane components via taints, which ordinary Pods are
   correctly repelled from by default.
5. **How would you run a DaemonSet on only a subset of nodes?**
   Add a `nodeSelector` (or node affinity rule) to the Pod template
   matching a label only present on the intended nodes.

## Mini Assignment

Deploy the `node-logger` DaemonSet across your full 3-node cluster,
confirm one Pod per node via `-o wide`, then cordon one node
(`kubectl cordon <node>` — marks it unschedulable for *new* Pods without
evicting existing ones) and observe whether its existing DaemonSet Pod
stays running or gets removed. Then drain that node
(`kubectl drain <node> --ignore-daemonsets --delete-emptydir-data`) and
observe the difference — document exactly what `--ignore-daemonsets`
had to account for and why.

## Lesson Summary

- A DaemonSet runs exactly one Pod per qualifying node, with count
  implicitly tied to cluster size rather than a fixed `replicas` number —
  structurally similar to ReplicaSet/Deployment otherwise.
- It exists for genuine per-node infrastructure concerns (logging,
  metrics, network plugins), never for application workloads that should
  scale independently of node count.
- `nodeSelector`/affinity scope a DaemonSet to a subset of nodes;
  `tolerations` let it run on otherwise-restricted nodes like
  control-plane ones.
- Resource limits matter more here than almost anywhere else — an
  oversized DaemonSet Pod affects every single node simultaneously.

---

### Before Chapter 21 (StatefulSets) — tell me:

1. Why doesn't setting a Deployment's `replicas` equal to your node count
   give you the same guarantee as a DaemonSet?
2. When would you deliberately scope a DaemonSet to only a subset of
   nodes, and how?
3. From the Mini Assignment — what happened to the DaemonSet Pod on a
   cordoned node versus a drained one, and why did `drain` need the
   `--ignore-daemonsets` flag specifically?
