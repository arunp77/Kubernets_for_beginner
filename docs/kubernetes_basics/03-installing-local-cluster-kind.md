# Chapter 3 — Installing a Local Cluster with Kind

## Lesson Objectives

By the end of this chapter you will:

- Have a real, multi-node Kubernetes cluster running locally.
- Understand *why* Kind ("Kubernetes IN Docker") is the natural on-ramp
  for someone who already knows Docker, versus Minikube or a cloud
  cluster.
- Be able to create, inspect, and tear down clusters, and configure
  multi-node topologies.
- Verify (for real) your Chapter 2 predictions about what happens when
  control-plane components fail.

---

## Theory

Kind's core trick, and the reason it's the right starting point for you
specifically: **each "node" in a Kind cluster is itself just a Docker
container**, running a specially-built image that contains a full
Kubernetes node stack (kubelet, containerd, etc.) inside it. So "spinning
up a 3-node Kubernetes cluster" on your laptop is, under the hood,
`docker run` three (or more) containers configured to talk to each other
as if they were separate machines — plus one more acting as the control
plane. Since you already understand Docker containers intimately, you can
literally `docker ps` and see your Kubernetes nodes as ordinary containers,
which demystifies a lot of what would otherwise feel like magic.

This is different from alternatives you may have heard of:

- **Minikube** typically runs a single node inside a real VM (or a
  container, depending on driver) — historically more common, but usually
  single-node by default and heavier.
- **k3s / k3d** — a genuinely lightweight, production-capable Kubernetes
  distribution; k3d wraps it in Docker containers similarly to Kind. Great
  choice too; Kind is slightly more "upstream vanilla Kubernetes,"
  which is why we're using it — what you learn transfers with zero
  surprises to EKS/GKE/AKS (Lesson 31).
- **A real cloud cluster** — the eventual target, but slower to iterate on
  and costs money; Kind gives you the identical API and object model for
  free, locally, in seconds.

---

## Architecture Diagram

```
 Your laptop
 ┌───────────────────────────────────────────────────────────┐
 │  Docker Engine                                             │
 │                                                             │
 │  ┌─────────────────────┐   ┌─────────────────────┐        │
 │  │ kind-control-plane    │   │ kind-worker           │        │
 │  │ (a container!)        │   │ (a container!)        │        │
 │  │  kube-apiserver       │   │  kubelet              │        │
 │  │  etcd                 │   │  kube-proxy           │        │
 │  │  scheduler             │   │  containerd            │        │
 │  │  controller-manager     │   │  [your app's Pods]      │        │
 │  └─────────────────────┘   └─────────────────────┘        │
 │              ▲  both containers on a shared Docker network  │
 │              │  so they can reach each other like real hosts │
 └───────────────────────────────────────────────────────────┘
        kubectl on your laptop talks to kind-control-plane's
        exposed API server port, exactly as it would talk to a
        real cluster's endpoint.
```

---

## Docker Comparison

| Concern | Plain Docker | Kind |
|---|---|---|
| What a "node" is | N/A — you just run containers | A Docker container running a full node stack |
| Networking between nodes | A Docker bridge network you set up | Kind creates a dedicated Docker network automatically |
| How you interact | `docker` CLI | `kubectl` CLI, talking to the API server exposed from the control-plane container |
| Multi-host simulation | Not possible on one machine | Fully simulated — you can create many "nodes," all as containers on your one machine |

The reassuring fact to hold onto: everything you already know about
`docker ps`, `docker logs`, `docker exec` still works on the Kind node
containers themselves — you're not learning a new virtualization
technology, just running Kubernetes' own components inside containers you
already know how to inspect.

---

## Internal Working

- Kind builds special **node images** that bundle a container runtime
  (containerd), kubelet, and all control-plane binaries into one image,
  then runs that image as a privileged Docker container so it can itself
  run nested containers for your Pods.
- On cluster creation, Kind uses `kubeadm` internally (the same tool used
  to bootstrap real production clusters) to initialize the control plane
  and join worker nodes — so the cluster you get is a real, standard
  Kubernetes cluster, not a simplified stand-in.
- Kind exposes the control plane's API server port to your host machine by
  mapping it, similarly to `-p` port publishing you already use with
  `docker run`, which is how `kubectl` running directly on your laptop can
  reach it.

---

## Hands-on Lab

**1. Install `kubectl`** (the CLI you'll use for the rest of the course):

```bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
kubectl version --client
```
This downloads the latest stable `kubectl` binary, installs it with
correct ownership/permissions into your PATH, and confirms it works.
`kubectl` itself does nothing without a cluster to talk to — it's just an
HTTP client for the API server, which is why the client-only version check
works even with no cluster yet.

**2. Install `kind`:**

```bash
curl -Lo ./kind https://kind.sigs.k8s.io/dl/latest/kind-linux-amd64
chmod +x ./kind
sudo mv ./kind /usr/local/bin/kind
kind version
```
Downloads the Kind binary, marks it executable, and moves it into PATH.

**3. Create a multi-node cluster.** First, define the topology in a config
file rather than relying on Kind's single-node default — this maps
directly to how real clusters are actually shaped:

```yaml
# kind-cluster.yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
  - role: control-plane
  - role: worker
  - role: worker
```
`kind: Cluster` / `apiVersion` identify this as Kind's own config schema
(not a Kubernetes object itself — Kind reads this before any cluster
exists). `nodes` lists the containers Kind should create: one
control-plane node and two worker nodes.

```bash
kind create cluster --name masterclass --config kind-cluster.yaml
```
`--name` lets you run multiple independent Kind clusters side by side;
`--config` points at the topology file above. This takes 1-2 minutes and,
behind the scenes, pulls the Kind node image, starts 3 containers, and
runs `kubeadm init`/`kubeadm join` across them.

**4. Confirm it's real Docker underneath:**

```bash
docker ps --filter "label=io.x-k8s.kind.cluster=masterclass"
```
You'll see 3 containers named like `masterclass-control-plane`,
`masterclass-worker`, `masterclass-worker2` — ordinary containers you
could `docker exec` into.

**5. Point kubectl at it and check node health:**

```bash
kubectl cluster-info --context kind-masterclass
kubectl get nodes
```
`cluster-info` confirms `kubectl` can reach the API server and shows its
address. `get nodes` lists the 3 nodes and their `Ready` status — this is
literally the API server reporting kubelet heartbeats, exactly as
described in Chapter 2.

**6. Verify your Chapter 2 predictions for real.** Kill the scheduler
inside the control-plane container and watch what happens:

```bash
docker exec masterclass-control-plane crictl ps | grep scheduler
docker exec masterclass-control-plane crictl stop <scheduler-container-id>
kubectl get pods -n kube-system | grep scheduler   # confirm it's gone/restarting
```
(`crictl` is the CRI-compatible CLI — like `docker ps`/`docker stop` but
speaking to containerd directly, since remember: no Docker Engine runs
*inside* the node itself, Chapter 1.) With the scheduler down, try:
```bash
kubectl create deployment test --image=nginx --replicas=1
kubectl get pods -o wide
```
Notice the Pod stays `Pending` — no node gets assigned — confirming your
Chapter 2 prediction. Kubernetes' own **static pod** mechanism restarts
the scheduler automatically within moments in a real cluster (kubelet
watches a local manifest directory for control-plane pods), so don't be
surprised if it self-heals quickly; that's the reconciliation loop working
even on itself.

---

## YAML Walkthrough

The only YAML this chapter introduces is Kind's own cluster config, walked
through inline in Step 3 above — it is *not* a Kubernetes object (no
`apiVersion`/`kind` you'd `kubectl apply`), it's config Kind itself
consumes before any cluster exists. True Kubernetes YAML (Pods,
Deployments, etc.) starts in Chapter 5–6.

---

## Troubleshooting

- **`kind create cluster` hangs on "Waiting for control-plane"** — usually
  Docker resource limits (memory) too low; Kind's control-plane container
  needs real headroom. Check `docker stats` while it's starting.
- **`kubectl` says `connection refused` / `context not found`** — you have
  more than one cluster context; run `kubectl config get-contexts` and
  `kubectl config use-context kind-masterclass` to select the right one.
- **Node shows `NotReady` and stays that way** — often no CNI (pod
  networking plugin) installed; Kind bundles a default one (`kindnet`),
  but if you disabled it in config, Pods and even node-readiness checks
  will never succeed. `kubectl describe node <name>` shows the exact
  condition holding it back.
- **Port already in use when creating cluster** — another Kind cluster or
  local service is bound to the same host port; `kind get clusters` then
  `kind delete cluster --name <old-one>` to clear stale ones.

---

## Best Practices

- Give Kind clusters an explicit `--name`; the default anonymous cluster
  gets confusing once you have more than one.
- Delete clusters you're not using — `kind delete cluster --name X` —
  they're cheap to recreate and each one is real running containers
  consuming your machine's resources.
- Use a multi-node config (as above) even locally, not the single-node
  default — scheduling/affinity/node-selector concepts (later chapters)
  behave invisibly-wrong on a single node and only reveal bugs once you
  have more than one place a Pod *could* have gone.
- Never use Kind for anything resembling production — it exists purely for
  local development/CI, matching upstream Kubernetes' API precisely.

---

## Interview Questions

1. **What is Kind, mechanically?**
   A tool that runs each Kubernetes "node" as a Docker container running a
   full node stack (kubelet, containerd, control-plane binaries via
   kubeadm), letting you run a real multi-node Kubernetes cluster on one
   machine.
2. **How is Kind different from Minikube?**
   Both are local dev clusters; Kind is Docker-container-based and
   explicitly multi-node-capable by design, closely tracking upstream
   Kubernetes; Minikube traditionally favored a single VM/node with
   pluggable drivers. Functionally similar for learning purposes.
3. **Why is Kind a good fit for CI pipelines?**
   Because "a cluster" is just a few Docker containers, it can be created
   and destroyed in under two minutes inside any CI runner that already
   has Docker — no cloud account, no cost, fully reproducible.
4. **What does `kubectl cluster-info` actually verify?**
   That `kubectl`'s configured context can reach the API server and get a
   response — the very first thing to check when anything else seems
   broken.

---

## Mini Assignment

Create a **5-node** Kind cluster (1 control-plane, 4 workers) named
`practice`, confirm all 5 show `Ready`, then deliberately stop the
container behind one worker node with plain `docker stop`. Watch how long
`kubectl get nodes` takes to mark it `NotReady`, and write down what you
observe. (Hint: there's a default node-monitor-grace-period at play — you
don't need to know the exact value yet, just observe the delay.)

---

## Lesson Summary

- Kind runs each Kubernetes node as a Docker container, giving you a real,
  standard, multi-node cluster in seconds, using tooling (Docker) you
  already know intimately.
- `kubectl` is just an HTTP client for the API server — install it once,
  point it at any cluster's context, and every command in this course
  works identically against Kind, EKS, GKE, or AKS.
- You verified, hands-on, that killing the scheduler stalls new pod
  placement without affecting already-running Pods — direct evidence of
  Chapter 2's architecture, not just theory.

---

### Before Lesson 4 (kubectl Deep Dive) — tell me:

1. What does `docker ps` show you about a Kind cluster that a real cloud
   cluster wouldn't show you the same way?
2. What did you observe when you stopped a worker node's container in the
   Mini Assignment — how long until `NotReady`?
3. Is your cluster still running? (We'll keep using it through the next
   several chapters.)
