# Chapter 9 — Services

## What it is

A **Service** is a stable network identity — a fixed virtual IP and DNS
name — that load-balances traffic across a dynamic, ever-changing set of
Pods matched by a label selector. It's Kubernetes' answer to the exact
problem you felt by hand in Lesson 1's lab: "which of these 3 ports do I
call, and what happens when one dies or a new one appears?"

## Why it exists

Pods are ephemeral — Chapter 5 established their IPs change on every
recreation, and a Deployment (Chapter 8) is constantly creating and
destroying Pods during rollouts and scaling. Nothing that talks to your
app should ever hold a Pod IP directly, because it will go stale.
A Service solves this by giving you one address that *never* changes,
while continuously tracking, in the background, which Pod IPs are
currently valid targets behind it.

## When to use it

Every single Deployment/StatefulSet that needs to receive traffic — from
other Pods inside the cluster, or from outside it — needs an accompanying
Service. There's no scenario in real usage where you talk to Pods
directly instead.

## Internal architecture

- A Service does not proxy traffic through any central process. Instead,
  **kube-proxy** (Chapter 2), running on every node, watches the API
  server for Services and their matching Pods (via an intermediate object
  called an **Endpoints**/`EndpointSlice`, listing current healthy Pod
  IPs), and programs local **iptables (or IPVS) rules** on that node so
  that traffic to the Service's virtual IP is transparently rewritten to
  one of the real Pod IPs — a local kernel-level NAT rule, not a proxy
  process traffic must hop through.
- This means a Service's "load balancing" happens as a side effect of
  packet rewriting rules present on *every* node — highly scalable, no
  single bottleneck, but also why a Service's IP (`ClusterIP`) only makes
  sense *within* the cluster (it's a virtual address kube-proxy's local
  rules resolve — nothing outside the cluster's nodes knows what to do
  with it).
- **CoreDNS** (a cluster add-on) watches Services too, and creates a DNS
  record for each one: `<service-name>.<namespace>.svc.cluster.local`
  (commonly just `<service-name>` works from within the same namespace).
  This is what lets your application code refer to `whoami-svc` by name,
  the same convenience Compose's embedded DNS gave you on one host —
  Kubernetes gives you the cluster-wide equivalent.
- Four Service **types**, in increasing scope:
  - `ClusterIP` (default) — reachable only from inside the cluster.
  - `NodePort` — additionally opens the same port on *every* node's own
    IP, so external traffic can reach it via `<any-node-ip>:<nodePort>`.
  - `LoadBalancer` — additionally asks the cloud-controller-manager
    (Chapter 2) to provision a real cloud load balancer (an AWS ELB, e.g.)
    pointing at the NodePort. On Kind/local clusters this type stays
    `<pending>` forever, since there's no real cloud to provision from —
    expected, not a bug.
  - `ExternalName` — a pure DNS alias to an external hostname, no
    proxying at all; niche, worth knowing exists.

## YAML Definition

```yaml
apiVersion: v1
kind: Service
metadata:
  name: whoami-svc
spec:
  type: ClusterIP
  selector:
    app: whoami
  ports:
    - port: 80
      targetPort: 80
      protocol: TCP
```

- `apiVersion: v1` — Service is a core/legacy type, like Pod.
- `spec.type` — one of the four above; `ClusterIP` if omitted.
- `spec.selector` — the exact same label-matching mechanism as a
  ReplicaSet's selector (Chapter 7) — any Pod with `app: whoami` becomes a
  valid target, regardless of which Deployment/ReplicaSet created it. This
  is the same selector-based coupling pattern reused a third time now.
- `spec.ports.port` — the port the Service itself listens on (what
  clients connect to).
- `spec.ports.targetPort` — the port on the *Pod* traffic gets forwarded
  to — can differ from `port` (e.g., expose `80` externally while the
  container actually listens on `8080`), directly analogous to Docker's
  `-p hostPort:containerPort` split, just at a different layer.

## Hands-on Example — solve Lesson 1's pain point for real

Reuse the Deployment from Chapter 8 (or recreate it), then add a Service:

```bash
kubectl apply -f whoami-deploy.yaml
kubectl apply -f whoami-svc.yaml
kubectl get svc whoami-svc
```

**Prove it load-balances across all matching Pods**, exactly like Lesson
1's manual `curl`-three-ports exercise — but now with one address:
```bash
kubectl run tmp-client --rm -it --image=busybox -- sh
# inside the shell:
wget -qO- whoami-svc
wget -qO- whoami-svc
wget -qO- whoami-svc
```
Run the `wget` three or four times — different `Hostname` values come
back, proving one stable name is silently distributing across multiple
Pods, with zero manual port bookkeeping.

**Prove the address survives Pod churn** — the entire point of this
chapter:
```bash
kubectl delete pod -l app=whoami --field-selector=status.phase=Running --limit=1
kubectl get svc whoami-svc    # the ClusterIP is UNCHANGED
wget -qO- whoami-svc          # still works, seamlessly, once the replacement Pod is ready
```

**Inspect the Endpoints backing the Service** — the live list kube-proxy
actually uses:
```bash
kubectl get endpoints whoami-svc
```
This updates automatically as Pods come and go — you never touch it
directly, but it's the exact bridge between "selector" and "which real
IPs traffic goes to right now."

**See it's not magic** — inspect the iptables rules kube-proxy actually
wrote, inside a node container (recall Kind nodes are just Docker
containers, Chapter 3):
```bash
docker exec masterclass-worker iptables -t nat -L -n | grep whoami
```

Try `NodePort` briefly to see the difference in scope:
```bash
kubectl patch svc whoami-svc -p '{"spec":{"type":"NodePort"}}'
kubectl get svc whoami-svc   # note the NodePort in the 3xxxx range
```

Cleanup:
```bash
kubectl delete -f whoami-svc.yaml -f whoami-deploy.yaml
```

## Debugging Techniques

- **Service exists, but `curl <service-name>` times out or connection
  refused** — check `kubectl get endpoints <svc-name>`. Empty output means
  the selector matches *zero* Pods — almost always a label typo/mismatch
  between the Service's `selector` and the Deployment's Pod template
  labels (Chapter 10 is essential here).
- **Endpoints has IPs, but traffic still fails** — check `targetPort`
  actually matches the port the container process is really listening on
  inside the Pod (`kubectl exec` in and check, or `kubectl logs` for a
  "listening on port X" startup message).
- **DNS name doesn't resolve from inside a Pod** — confirm CoreDNS itself
  is healthy: `kubectl get pods -n kube-system -l k8s-app=kube-dns`.
- **`LoadBalancer` type stuck at `<pending>` forever on Kind** — expected;
  no cloud-controller-manager exists locally to provision a real load
  balancer. This is not a bug to chase on a local cluster.

## Best Practices

- Never let application code hardcode a Pod IP — always use the Service's
  DNS name; this is non-negotiable given Chapter 5's ephemeral-IP
  guarantee.
- Default to `ClusterIP` for anything that's purely internal (databases,
  caches, internal APIs) — only use `NodePort`/`LoadBalancer` for what
  genuinely needs external exposure, and in production prefer routing
  external HTTP traffic through an **Ingress** (Chapter 16) rather than
  many individual `LoadBalancer` Services, for cost and manageability.
- Keep Service selectors and Deployment Pod-template labels reviewed
  together — they're two separate YAML blocks that must agree, and
  nothing enforces that agreement for you at write-time (Chapter 6's
  "no cross-object schema check" limitation).

## Interview Questions

1. **How does a Service actually route traffic to Pods — is there a proxy
   process in the path?** No dedicated proxy process handles each
   packet — kube-proxy on every node programs local iptables/IPVS rules
   that rewrite the Service's virtual IP to a real Pod IP at the kernel
   level, based on the live Endpoints list.
2. **Why doesn't a Service's ClusterIP change when its Pods are
   replaced?** Because the ClusterIP is assigned to the Service object
   itself, independent of any Pod — Pods come and go, tracked via
   Endpoints, while the Service's own identity is stable for its entire
   lifetime.
3. **What's the difference between `port` and `targetPort`?**
   `port` is what clients connect to on the Service; `targetPort` is the
   actual port the container is listening on inside the Pod — they can
   differ, analogous to Docker's host-port vs. container-port split.
4. **What are the four Service types and their scopes?**
   ClusterIP (internal only, default), NodePort (internal + every node's
   IP on a fixed port), LoadBalancer (adds a real cloud LB, provisioned via
   cloud-controller-manager), ExternalName (pure DNS alias, no proxying).
5. **A Service has no Endpoints — what's the most common cause?**
   A label selector mismatch between the Service and the target Pods'
   labels — always check this before assuming anything is broken at the
   networking layer.

## Mini Assignment

Deploy two separate Deployments with intentionally *different* Pod-label
values (e.g., `app: whoami-a` and `app: whoami-b`), then create one
Service whose selector matches *both* on purpose (e.g., select on a common
label you add to both, like `tier: demo`). Confirm via repeated `wget`
that traffic really does load-balance across Pods from both Deployments —
proving Services couple purely on labels, with zero awareness of which
Deployment/ReplicaSet actually created a given Pod.

## Lesson Summary

- A Service gives a stable virtual IP + DNS name to a dynamic set of Pods,
  selected purely by label — solving Lesson 1's "which port do I call, and
  what about failover" problem for real.
- kube-proxy implements this as local kernel-level packet rewriting on
  every node, driven by a live Endpoints list — no central proxy process,
  no bottleneck.
- Always talk to Services, never Pod IPs — the mismatch/empty-Endpoints
  case is the single most common Service-related bug, always traceable to
  label selectors.
- Four types (ClusterIP/NodePort/LoadBalancer/ExternalName) trade off
  scope of exposure; default to ClusterIP and add Ingress (Chapter 16) for
  real external HTTP routing.

---

### Before Chapter 10 (Labels and Selectors) — tell me:

1. Explain, mechanically, how a packet addressed to a Service's ClusterIP
   actually ends up at a real Pod — no hand-waving "it load balances."
2. Why does `LoadBalancer` type stay `<pending>` forever on your Kind
   cluster, and is that a bug?
3. From the Mini Assignment — did a Service really route to Pods from two
   completely separate Deployments once their labels matched its
   selector?
