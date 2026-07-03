# Chapter 2 — Kubernetes Architecture

## Lesson Objectives

By the end of this chapter you will be able to:

- Name every control-plane and node component and state, in one sentence
  each, what problem it solves.
- Trace the exact path a `kubectl apply` request takes from your terminal
  to a running container.
- Explain why Kubernetes is split into so many small processes instead of
  one daemon (contrast with Docker Engine, which *is* one daemon).
- Reason about what breaks when each component fails.

---

## Theory

In Lesson 1 you saw the shape of the idea: a **control plane** holds
desired state and continuously reconciles it against the real cluster.
This chapter opens that box. Where Docker Engine is a single daemon
(`dockerd`) that does everything — API, image management, container
runtime, networking — Kubernetes deliberately splits every responsibility
into its own small, single-purpose process. That's not accidental
complexity; it's the same Unix philosophy as running many small
cooperating tools instead of one monolith, applied to a distributed
system. Each piece can fail, restart, scale, or be replaced independently.

### Control plane components (the "brain")

- **kube-apiserver** — the *only* front door. Every read and write to
  cluster state — `kubectl`, the scheduler, every controller, kubelets on
  every node — goes through this one HTTP(S) REST API. It validates
  requests, enforces auth/RBAC, and is the only component allowed to talk
  to etcd. Stateless by design — you can (and production clusters do) run
  several replicas behind a load balancer.
- **etcd** — a distributed, strongly-consistent key-value store (uses the
  Raft consensus algorithm) holding the entire state of the cluster: every
  object you've ever `kubectl apply`'d lives here as a serialized record.
  This is the cluster's single source of truth — if etcd is lost with no
  backup, the cluster's history of "what should exist" is gone, even if
  containers happen to still be running.
- **kube-scheduler** — watches the API server for Pods that exist but have
  no node assigned (`spec.nodeName` empty), scores every node against the
  Pod's resource requests and constraints, and writes back "run this Pod
  on Node X." The scheduler never actually starts anything — it only makes
  the assignment decision.
- **kube-controller-manager** — actually a bundle of controllers running
  in one process for operational convenience (Node controller, Job
  controller, ReplicaSet controller, and many more). Each runs its own
  independent reconciliation loop: watch the API server for objects it
  owns, compare desired vs. observed, act.
- **cloud-controller-manager** — the cloud-specific glue: when you create a
  `LoadBalancer`-type Service on EKS, *this* is what actually calls the AWS
  API to provision a real ELB. Keeps cloud-vendor code out of core
  Kubernetes.

### Node components (the "muscle")

- **kubelet** — an agent running on every node. It watches the API server
  for Pods assigned to *its* node, and makes the container runtime create
  the actual containers to match the Pod spec. It also reports node/pod
  health back to the API server — this is the heartbeat that lets the
  control plane know a node (and everything on it) has died.
- **kube-proxy** — maintains the networking rules (iptables or IPVS rules)
  on each node that make a Service's stable virtual IP actually route to
  one of the real Pod IPs behind it. This is the piece that makes cluster
  networking "just work" from an app's point of view.
- **Container runtime** — containerd or CRI-O, spoken to via CRI, as
  covered in Lesson 1. This is the process that actually creates namespaces,
  cgroups, and starts the container process — the same primitives Docker
  Engine uses under the hood.

### The request lifecycle — tracing `kubectl apply -f deployment.yaml`

1. `kubectl` reads your YAML, converts it to JSON, and sends an HTTPS
   request to the **API server**.
2. The API server authenticates you, checks **RBAC** (Lesson 24) to
   confirm you're allowed to create this object, validates the object's
   schema, and writes it into **etcd**.
3. The **Deployment controller** (inside kube-controller-manager) is
   watching the API server for Deployment objects. It notices the new one
   and creates a matching **ReplicaSet** object (via the API server —
   controllers never touch etcd directly either).
4. The **ReplicaSet controller** notices the new ReplicaSet and creates
   the requested number of **Pod** objects — still with no node assigned.
5. The **scheduler** notices unscheduled Pods, picks nodes for them, and
   patches each Pod's `spec.nodeName`.
6. The **kubelet** on each chosen node notices (via watching the API
   server) that a Pod has been assigned to it, and instructs the local
   **container runtime** to pull the image and start the container.
7. **kube-proxy** on every node updates its routing rules so the new Pods
   are reachable through any Service that selects them.

Nobody in this chain talks to anyone else directly except through the API
server — this "hub and spoke, watch-and-react" design is why components
can be restarted, upgraded, or replaced independently without the whole
system falling over.

---

## Architecture Diagram

```
                        ┌───────────────────────────────────────────┐
                        │              CONTROL PLANE                  │
                        │                                             │
   kubectl / CI ───────▶│   kube-apiserver  ◀────────────────────┐   │
                        │        │   ▲                            │   │
                        │        ▼   │ (only writer/reader)       │   │
                        │      etcd                                │   │
                        │        (cluster state, Raft consensus)   │   │
                        │                                          │   │
                        │   kube-scheduler ───────────────────────┘   │
                        │   kube-controller-manager ───────────────────┘
                        │   cloud-controller-manager
                        └──────────────────┬──────────────────────────┘
                                           │ watches / patches via API server
              ┌────────────────────────────┼────────────────────────────┐
              │                            │                            │
        ┌───────────┐               ┌───────────┐                ┌───────────┐
        │   Node 1    │               │   Node 2    │                │   Node 3    │
        │ ┌───────┐ │               │ ┌───────┐ │                │ ┌───────┐ │
        │ │kubelet  │ │               │ │kubelet  │ │                │ │kubelet  │ │
        │ └───────┘ │               │ └───────┘ │                │ └───────┘ │
        │ ┌───────┐ │               │ ┌───────┐ │                │ ┌───────┐ │
        │ │kube-proxy│               │ │kube-proxy│               │ │kube-proxy│
        │ └───────┘ │               │ └───────┘ │                │ └───────┘ │
        │ containerd  │               │ containerd  │                │ containerd  │
        │ [Pod][Pod]  │               │ [Pod][Pod]  │                │ [Pod][Pod]  │
        └───────────┘               └───────────┘                └───────────┘
```

---

## Docker Comparison

| Concern | Docker Engine | Kubernetes |
|---|---|---|
| Process model | One daemon (`dockerd`) does everything | Split into ~7 single-purpose processes |
| Source of truth | In-memory + on-disk state on that one host | etcd, a replicated distributed database |
| API | Docker Engine API, one host | kube-apiserver, one cluster-wide endpoint, HA-able |
| Scheduling | None — you pick the host by running the command there | kube-scheduler picks a node automatically from the whole cluster |
| Networking | Docker's bridge/overlay networks, per host | kube-proxy + a cluster networking plugin (CNI) spanning all nodes |
| Failure blast radius | Daemon down = host's containers unmanaged | Each component can fail/restart independently; API server is typically run with 3 replicas for HA |

The mental shift: Docker Engine is "one program on one computer." A
Kubernetes cluster is a small distributed system in its own right, with
its own database, its own API, and its own consensus protocol — before
your application even enters the picture.

---

## Internal Working

- **Watch, don't poll.** Every controller and kubelet uses the API
  server's **watch** mechanism (a long-lived HTTP connection streaming
  change events) rather than repeatedly polling "has anything changed?"
  This is why reconciliation feels near-instant even though nothing is
  tightly coupled.
- **Optimistic concurrency via `resourceVersion`.** Every object stored in
  etcd carries a `resourceVersion`. When a controller updates an object, it
  must include the version it last read; if someone else changed it in the
  meantime, the write is rejected and the controller re-reads and retries.
  This is how many independent controllers can safely modify the cluster
  concurrently without a central lock.
- **Level-triggered, not edge-triggered.** Controllers don't react to "an
  event happened" so much as "here is the current desired state; here is
  the current observed state — reconcile them," every time they wake up.
  This makes the system self-healing even if a controller misses an event
  entirely (crashes, restarts) — on its next pass it just re-derives the
  full diff from scratch.

---

## Hands-on Lab

We don't have a cluster yet (Lesson 3 builds one with Kind), so this lab
is a **prediction exercise** you'll immediately verify once Kind is up.
Write down, right now, your answers to:

1. If `etcd` is destroyed with no backup but every Node keeps running,
   what happens to already-running containers? What happens the next time
   any of them crashes?
2. If the `kube-scheduler` process is killed and doesn't restart, what
   happens to Pods that are already running? What happens if you create a
   *new* Deployment while it's down?
3. If a Node's `kubelet` process dies but the Node itself and its
   containers keep running, how would the rest of the cluster find out
   something is wrong, and how long do you think that takes?

Keep these answers — in Lesson 3, once `kind` is running, we'll actually
`docker exec` into the control-plane container, kill some of these
processes for real, and check your predictions with `kubectl get
componentstatuses` / `kubectl get nodes` / pod behavior.

---

## YAML Walkthrough

Not applicable — this chapter has no new object type. Every Kubernetes
object you'll define starting in Lesson 5 is stored as etcd records
exactly as described above; that mechanism is what this chapter explains.

---

## Troubleshooting

- **`kubectl` hangs or times out** — almost always means it can't reach
  the **kube-apiserver** (wrong context, control plane down, network
  issue) — check with `kubectl cluster-info` first, always, before
  debugging anything downstream.
- **Object created but "nothing happens"** — check whether the relevant
  **controller** is running at all (`kubectl get pods -n kube-system`);
  a stuck/crashed controller means desired state is recorded but nobody
  is reconciling it.
- **Pod stuck in `Pending` forever** — almost always a **scheduler**
  problem: no node satisfies the Pod's resource requests/constraints. `
  kubectl describe pod <name>` shows scheduler events explaining exactly
  why (you'll use this constantly starting Lesson 5).
- **`kubectl get nodes` shows `NotReady`** — the API server hasn't heard a
  heartbeat from that node's kubelet recently; investigate the node itself
  (kubelet crashed, network partition), not the control plane.

---

## Best Practices

- In production, run **at least 3 etcd replicas** (odd numbers, for Raft
  quorum) and back them up regularly and verifiably — etcd loss is the
  single worst-case disaster in a cluster.
- Run the **API server** behind a load balancer with multiple replicas;
  it's stateless, so this is cheap redundancy to buy.
- Treat control-plane components as infrastructure you either operate
  very carefully (self-managed clusters) or hand entirely to a cloud
  provider (Lesson 31) — very few teams should be hand-rolling etcd
  operations themselves.
- Learn to read `kubectl describe` and `kubectl get events` output fluently
  before anything else — nearly every debugging session in this course
  starts there.

---

## Interview Questions

1. **Name the control-plane components and what each does.**
   kube-apiserver (front door/validation), etcd (state store), scheduler
   (node assignment), controller-manager (reconciliation loops),
   cloud-controller-manager (cloud-provider integration).
2. **What does the scheduler actually do, mechanically?**
   Watches for Pods with no assigned node, scores eligible nodes against
   the Pod's requirements, and writes the chosen node back via the API
   server. It never starts containers itself.
3. **Why does only the API server talk to etcd?**
   Single point of validation, auth, and consistency control — every
   other component goes through the same schema/RBAC checks, and etcd's
   consistency guarantees aren't accidentally bypassed by a stray writer.
4. **What happens if etcd loses quorum?**
   The cluster becomes read-only/unable to accept new writes (no new
   objects, no reconciliation of new desired state), even though already
   -running Pods keep running until something needs to change.
5. **How does a kubelet know what to run?**
   It watches the API server for Pods whose `spec.nodeName` matches its
   own node, and drives the local container runtime via CRI to match that
   spec.

---

## Mini Assignment

Draw (on paper or in any tool) the full request lifecycle from
"`kubectl apply -f deployment.yaml`" to "container running on a node,"
labeling every component involved and, at each arrow, saying whether that
communication happens over a **watch** or a one-shot **write**. Compare
your diagram against the "Request lifecycle" list above — don't peek until
you've tried.

---

## Lesson Summary

- Kubernetes splits Docker Engine's single-daemon responsibilities into
  independent processes: **kube-apiserver** (front door), **etcd** (state),
  **kube-scheduler** (node assignment), **kube-controller-manager**
  (reconciliation), **cloud-controller-manager** (cloud integration) on the
  control plane; **kubelet**, **kube-proxy**, and the **container runtime**
  on every node.
- Every interaction goes through the API server; only the API server talks
  to etcd; everyone else watches the API server and reacts.
- This design — small single-purpose watchers reacting to shared state —
  is exactly the controller pattern from Lesson 1, applied recursively to
  build the control plane itself.

---

### Before Lesson 3 (Installing a Local Cluster) — tell me:

1. Walk me through what happens, component by component, from `kubectl
   apply` to a container running.
2. Why is it safe for many controllers to modify cluster state
   concurrently without a central lock?
3. Your three predictions from the Hands-on Lab — we'll verify them for
   real once Kind is running.
