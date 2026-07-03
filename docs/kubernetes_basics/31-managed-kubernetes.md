# Chapter 31 — Managed Kubernetes (EKS / GKE / AKS)

## What it is

A **managed Kubernetes service** — Amazon EKS, Google GKE, Azure AKS —
is a cloud provider running and operating the **control plane** (Chapter
2: kube-apiserver, etcd, scheduler, controller-manager) on your behalf,
while you bring (and usually still partially manage) the **worker
nodes**. You never SSH into a control-plane machine, never manage etcd
backups yourself, never patch a scheduler binary — the provider does, and
you consume the API server endpoint exactly as you've done all course,
just pointed at a real, provider-hosted cluster instead of Kind.

## Why it exists

Chapter 2 gave you a genuine appreciation for how much *is* a Kubernetes
control plane: etcd's Raft consensus and backup story, API server HA, the
scheduler and controller-manager as continuously-running processes that
themselves need monitoring, patching, and upgrading. Running that
correctly, with real availability guarantees, is substantial specialized
operational work — and it's largely **undifferentiated** work: your
etcd's job is identical to every other company's etcd, so there's no
competitive advantage in operating it yourself. Managed Kubernetes exists
because cloud providers can operate that shared, undifferentiated layer
at a scale and reliability individual teams rarely justify building
in-house — letting you spend your operational effort on your own
application and its worker-node/workload-level concerns instead (exactly
Chapter 30's six categories), not on etcd quorum health.

## When to use it

Virtually always, for anything beyond local development/learning
(Kind, Chapters 3-30) or genuine on-prem/air-gapped requirements. Running
your own control plane (via `kubeadm`, the same tool Kind uses internally,
Chapter 3) is a legitimate choice only when you have a specific, justified
reason — regulatory/data-residency constraints, extreme customization
needs, or simply not being able to use a public cloud at all.

## Internal architecture — what changes, what doesn't

- **What doesn't change at all**: every object type, every `kubectl`
  command, every YAML field from Chapters 5-30. A Deployment, Service,
  StatefulSet, RBAC Role — identical schema and behavior on Kind, EKS,
  GKE, or AKS. This is the entire point of Kubernetes being a
  standardized API (CNCF-conformant, in fact — there's a real
  conformance test suite every managed offering must pass) rather than
  each cloud inventing its own orchestration API.
- **What changes**: everything **around** the API, specific to each
  provider's integration points:
  - **Authentication** — instead of a static token/cert in your
    kubeconfig, cloud clusters typically use an **exec plugin** that
    calls the cloud CLI (`aws eks get-token`, `gcloud container
    clusters get-credentials`, `az aks get-credentials`) to fetch a
    short-lived token on demand, tied to your cloud IAM identity — a
    direct extension of Chapter 24's RBAC, now bridged to your cloud
    provider's own identity system (AWS IAM, GCP IAM, Azure AD) via a
    mapping layer (EKS's `aws-auth` ConfigMap or newer access-entry API;
    GKE's Workload Identity; AKS's Azure AD integration).
  - **`LoadBalancer`-type Services and Ingress** (Chapters 9, 16) —
    finally do something real: the **cloud-controller-manager** (Chapter
    2) provisions an actual ELB/Cloud Load Balancer/Azure Load Balancer
    automatically — this is the exact `<pending>` state you saw forever
    on Kind, now resolving to a real IP within minutes.
  - **StorageClasses** (Chapter 15) ship with real, provider-specific
    default provisioners (`ebs.csi.aws.com`, `pd.csi.storage.gke.io`,
    Azure Disk/Files CSI) — dynamic provisioning that was simulated
    locally now creates real cloud disks.
  - **Node provisioning/autoscaling** — a **Cluster Autoscaler** (or
    Karpenter on AWS) adds/removes entire **nodes** based on unschedulable
    Pods — a distinct, complementary concern to Chapter 23's HPA, which
    only ever changes replica *counts* assuming nodes already have room;
    Cluster Autoscaler is what actually grows the cluster itself when
    HPA's desired replica count can't fit anywhere.
  - **Managed node groups / node pools** — the provider automates
    worker-node OS patching, kubelet version upgrades, and node
    replacement, though (importantly) you're still often responsible for
    triggering/scheduling these upgrades and choosing node instance
    types/sizes.
- **What is *explicitly still your job* on every managed offering**:
  everything from Chapter 30's six categories — your own workloads'
  probes, resource requests, RBAC scoping, NetworkPolicies, monitoring,
  and deploy safety. Managed Kubernetes removes control-plane
  operational burden; it does not remove any of the workload-level
  discipline this entire course has been building.

## YAML/Config Walkthrough — what's genuinely provider-specific

```yaml
# a Service, byte-for-byte identical whether on Kind or EKS/GKE/AKS —
# the ONLY provider-specific part is an annotation, an escape hatch
# exactly like Ingress's (Chapter 16)
apiVersion: v1
kind: Service
metadata:
  name: whoami-svc
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-type: "nlb"   # AWS-specific
spec:
  type: LoadBalancer
  selector:
    app: whoami
  ports:
    - port: 80
      targetPort: 80
```
- Every field except `metadata.annotations` is identical to Chapter 9's
  Service — confirming the earlier claim directly: the object schema
  doesn't change, only a small, clearly-marked provider-specific
  extension point does.
- `type: LoadBalancer` — the exact field from Chapter 9 that stayed
  `<pending>` forever on Kind — on a real cloud, this now triggers
  `cloud-controller-manager` to actually provision infrastructure.

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-ssd
provisioner: ebs.csi.aws.com     # was rancher.io/local-path on Kind (Ch. 15)
parameters:
  type: gp3
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer
```
Identical shape to Chapter 15's StorageClass — only `provisioner` and its
`parameters` are cloud-specific; `volumeBindingMode`'s reasoning
(zone-aware provisioning) is now *actually* load-bearing, not just
theoretical, since real clusters genuinely span multiple availability
zones.

## Hands-on Example — conceptual walkthrough (no real cloud account required for this course)

Since standing up a real EKS/GKE/AKS cluster costs real money and isn't
required to complete this course, treat this as a guided comparison
rather than a lab — but the commands are real and worth knowing:

```bash
# EKS
aws eks update-kubeconfig --name my-cluster --region us-east-1
# GKE
gcloud container clusters get-credentials my-cluster --zone us-central1-a
# AKS
az aks get-credentials --resource-group my-rg --name my-cluster
```
Each of these does exactly one thing: merges a new context into your
`~/.kube/config` (Chapter 4) with an exec-plugin auth entry — after this,
every single `kubectl` command you've used this entire course works
identically:
```bash
kubectl get nodes
kubectl apply -f whoami-deploy.yaml -f whoami-svc.yaml
kubectl get svc whoami-svc -w    # watch EXTERNAL-IP go from <pending> to a real address — the payoff
```

**Confirm the control plane is genuinely managed** — try to find it the
way you did on Kind in Chapter 3 (`docker ps` showing control-plane
containers) — you can't; there's no control-plane container/VM visible to
you at all, only worker nodes:
```bash
kubectl get nodes   # only worker nodes are ever listed — the control plane is invisible by design
```

**Install the same Helm charts from earlier chapters, unmodified:**
```bash
helm install monitoring prometheus-community/kube-prometheus-stack -n monitoring --create-namespace
```
Identical command to Chapter 27 on Kind — this is the conformance
guarantee made concrete: ecosystem tooling built against the standard
Kubernetes API works unmodified across every conformant distribution.

## Debugging Techniques

- **`kubectl` fails with an auth error specifically on a managed
  cluster, not on Kind** — check the exec-plugin/IAM-mapping layer first
  (has your IAM/service identity actually been granted cluster access —
  EKS's `aws-auth`/access entries, GKE IAM bindings, AKS Azure AD role
  assignments) — this is an added layer beyond Chapter 24's in-cluster
  RBAC, not a replacement for it; you need both correctly configured.
- **`LoadBalancer` Service stuck `<pending>` even on a real cloud** —
  check cloud-provider IAM permissions for the cluster's own
  cloud-controller-manager identity, and check cloud-side quotas (a
  region's load-balancer limit, for instance) — the exact same symptom
  as Kind, but now with a real, debuggable cloud-side cause instead of
  "expected, no cloud controller exists."
- **Nodes not scaling up despite unschedulable Pods** — check the
  Cluster Autoscaler (or Karpenter)'s own logs/config specifically; this
  is a separate system from HPA (Chapter 23), and HPA alone cannot create
  new nodes, only request more replicas.
- **"Works on my Kind cluster, fails on the managed one"** — almost
  always one of the concerns this chapter names explicitly: IAM/auth
  mapping, provider-specific StorageClass parameters, or a
  `LoadBalancer`/Ingress annotation that's provider-specific and was
  silently ignored (not erroring) on a different provider.

## Best Practices

- Treat cloud IAM-to-Kubernetes-RBAC mapping as its own security surface,
  reviewed with the same rigor as Chapter 24's in-cluster RBAC — a
  correctly-scoped Role means nothing if the IAM mapping grants broader
  cluster access than intended.
- Pair HPA (Chapter 23, scales replicas) with a node autoscaler (Cluster
  Autoscaler/Karpenter, scales nodes) deliberately — they solve different
  layers of the same elasticity problem, and relying on only one leaves a
  real gap (replicas that want to exist but have nowhere to be
  scheduled).
- Pin Kubernetes version upgrades and worker-node image upgrades to a
  deliberate, tested schedule — managed doesn't mean "automatic and
  invisible"; you're generally still responsible for triggering and
  validating upgrades, even though the mechanics are provider-assisted.
- Use each provider's native cost/usage tooling early — cloud-managed
  clusters make it very easy to provision real, billed infrastructure
  (real load balancers, real disks, real autoscaled nodes) far faster
  than a local Kind cluster ever could, and costs can grow unnoticed.

## Interview Questions

1. **What specifically does a managed Kubernetes service (EKS/GKE/AKS)
   take off your plate?** Operating the control plane — etcd, API server,
   scheduler, controller-manager — including their availability, backups,
   and patching; you still generally manage (or at least own decisions
   about) worker nodes and entirely own workload-level concerns.
2. **Why does the same YAML/kubectl workflow work identically across
   Kind, EKS, GKE, and AKS?** Because Kubernetes is a standardized,
   CNCF-conformance-tested API — the object schema and control-plane
   behavior are consistent by design; only small, explicitly
   provider-specific extension points (annotations, StorageClass
   provisioners) differ.
3. **What's the difference between the Horizontal Pod Autoscaler and a
   Cluster Autoscaler?** HPA changes a workload's replica count based on
   metrics, assuming node capacity exists to place new replicas; Cluster
   Autoscaler (or Karpenter) adds/removes actual nodes based on
   unschedulable Pods — they're complementary, and relying on only HPA
   leaves replicas stuck `Pending` with nowhere to be scheduled.
4. **How does cloud-provider authentication typically integrate with
   Kubernetes RBAC on a managed cluster?** Via an exec-plugin in
   kubeconfig that fetches a short-lived token tied to your cloud IAM
   identity, combined with a provider-specific mapping (EKS's `aws-auth`/
   access entries, GKE Workload Identity, AKS Azure AD integration) that
   ties that cloud identity to in-cluster RBAC subjects — both layers
   must be correctly configured.
5. **Does "managed Kubernetes" mean you no longer need Chapter 30's
   production-readiness practices?** No — it removes control-plane
   operational burden specifically; every workload-level concern (probes,
   resource requests, RBAC scoping, NetworkPolicies, observability,
   deploy safety) remains entirely your responsibility regardless of who
   runs the control plane.

## Mini Assignment

Without provisioning a real cloud cluster, write out (as documentation,
not code) the exact sequence of steps and cloud-specific
configuration you'd need to take your Chapter 30 hardened `whoami`
workload — Deployment, HPA, ServiceAccount+Role, NetworkPolicy, and a
LoadBalancer Service — from your local Kind cluster to a real EKS
cluster. Call out explicitly, for each object, whether it needs zero
changes, an added annotation, or a genuinely different configuration —
and identify which two additional systems (beyond anything from Chapters
1-30) you'd need to newly configure specifically because you're now on a
real cloud (hint: IAM mapping and node/cluster autoscaling).

## Lesson Summary

- Managed Kubernetes (EKS/GKE/AKS) means the cloud provider operates your
  control plane — you never touch etcd, the API server, or the scheduler
  directly, but everything you've learned about the Kubernetes API itself
  (Chapters 4-30) is unchanged and directly portable.
- The provider-specific surface is narrow and identifiable: cloud
  IAM-to-RBAC auth mapping, real `LoadBalancer`/Ingress provisioning,
  real StorageClass provisioners, and node-level autoscaling/upgrades —
  everything else is standard Kubernetes.
- Cluster Autoscaler/Karpenter (nodes) and HPA (replicas, Chapter 23) are
  complementary, not substitutes — both are needed for real end-to-end
  elasticity.
- Managed Kubernetes eliminates control-plane operational burden; it does
  not eliminate any of Chapter 30's workload-level production-readiness
  responsibilities — those remain entirely yours, on any cluster.

---

### Course reflection — you've completed the masterclass. Before moving to the final project build:

1. Name, from memory, the one thing that's genuinely different about
   `kubectl` usage on a managed cloud cluster versus Kind, and the one
   thing that's completely unchanged.
2. Why can't HPA alone guarantee your workload scales successfully under
   real load spikes on a real cloud cluster?
3. From the Mini Assignment — which two systems did you identify as
   newly needed specifically because you're moving off a local cluster
   onto a real cloud one?

You now have the complete conceptual and hands-on foundation — Chapters
1-31 — needed to design, deploy, troubleshoot, and operate production
Kubernetes clusters. The natural next step is applying every mechanism
here, together, on this course's final project (React + FastAPI +
Postgres + Redis + worker + Prometheus + Grafana + Ingress), which is
where isolated chapter lessons become one coherent system.
