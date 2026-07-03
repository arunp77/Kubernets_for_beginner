# Chapter 21 — StatefulSets

## What it is

A **StatefulSet** manages Pods that need **stable, unique identity** —
a predictable name and network address that survives rescheduling — and
**ordered, one-at-a-time** creation/scaling/deletion, unlike a
Deployment's interchangeable, arbitrarily-ordered Pods.

## Why it exists

Every controller so far (ReplicaSet, Deployment, DaemonSet) treats its
Pods as **interchangeable** — `whoami-7d9f8-x7k2p` and
`whoami-7d9f8-m3n1q` are equivalent; a Service load-balances across them
with no notion of "which one." That model is exactly right for stateless
apps and is wrong for a database cluster: node 0 of a Postgres replica
set is not interchangeable with node 1 — they have distinct roles
(primary vs. replica), distinct data on distinct volumes, and often must
start up in a specific order (primary before replicas can attach to it).
A StatefulSet gives each Pod a fixed ordinal identity (`db-0`, `db-1`,
`db-2`) that's **recreated with the same name and the same bound storage**
if it dies — solving the exact problem a Deployment's "any replacement
Pod, any new IP, any new PVC" model can't.

## When to use it

Anything with per-instance identity/state: databases (Postgres, MySQL,
MongoDB replica sets), distributed coordination systems (etcd, ZooKeeper,
Kafka), anything where "which specific instance am I, and what's *my*
data" matters. If instances are truly interchangeable, use a Deployment
instead — StatefulSets are strictly more operationally complex and should
only be reached for when that complexity is actually needed. In practice,
many teams run databases as **managed cloud services** instead of
StatefulSets precisely to avoid operating this complexity themselves —
worth keeping in mind as a legitimate alternative, not a cop-out.

## Internal architecture

- **Stable network identity**: each Pod gets a predictable hostname
  `<statefulset-name>-<ordinal>` (`db-0`, `db-1`, ...), and a
  **headless Service** (`clusterIP: None` — deliberately *no* single
  virtual IP/load-balancing) gives each Pod its own individually
  resolvable DNS name: `db-0.db-headless.default.svc.cluster.local`. This
  is the crucial contrast with Chapter 9's normal Service: a normal
  Service hides *which* Pod you're talking to; a headless Service exists
  specifically so you *can* address one specific Pod by name — essential
  for a database client that must talk to the primary specifically.
- **Stable storage**: each ordinal gets its **own** PersistentVolumeClaim,
  created from a `volumeClaimTemplate` — `db-0` always gets rebound to
  the *same* PVC (and thus the same underlying data) even after being
  deleted and recreated, unlike a Deployment's Pods, which share no such
  persistent per-replica binding.
- **Ordered, sequential lifecycle**: by default, scaling up creates `db-0`
  first, waits for it to be `Running and Ready` (your Chapter 17 probes
  matter enormously here), *then* creates `db-1`, and so on. Scaling down
  reverses the order (highest ordinal removed first). This ordering
  exists because many stateful systems have genuine startup dependencies
  (a replica needs the primary already up) that an unordered rollout
  would violate.
- `podManagementPolicy: Parallel` opts out of strict ordering when your
  specific workload doesn't need it (faster scaling), while
  `OrderedReady` (default) keeps the safety guarantee.

## YAML Definition

```yaml
apiVersion: v1
kind: Service
metadata:
  name: db-headless
spec:
  clusterIP: None
  selector:
    app: db
  ports:
    - port: 5432
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: db
spec:
  serviceName: db-headless
  replicas: 3
  selector:
    matchLabels:
      app: db
  template:
    metadata:
      labels:
        app: db
    spec:
      containers:
        - name: db
          image: postgres:16
          env:
            - name: POSTGRES_PASSWORD
              value: "demo"
          ports:
            - containerPort: 5432
          volumeMounts:
            - name: data
              mountPath: /var/lib/postgresql/data
  volumeClaimTemplates:
    - metadata:
        name: data
      spec:
        accessModes: ["ReadWriteOnce"]
        resources:
          requests:
            storage: 1Gi
```
- `Service.spec.clusterIP: None` — this is what makes it "headless";
  DNS resolves individual Pod names instead of one virtual IP.
- `StatefulSet.spec.serviceName` — must name the headless Service that
  provides each Pod's individual DNS identity — the API server requires
  this link explicitly.
- `spec.volumeClaimTemplates` — the key structural difference from every
  earlier chapter's Deployment/Pod: instead of one shared `volumes` +
  `persistentVolumeClaim.claimName` (Chapter 14, one PVC for the whole
  Deployment's Pods to fight over — which wouldn't even make sense with
  `ReadWriteOnce`), each ordinal gets its **own** PVC automatically
  created from this template: `data-db-0`, `data-db-1`, `data-db-2`.

## Hands-on Example

```bash
kubectl apply -f db-headless-svc.yaml -f db-statefulset.yaml
kubectl get statefulset db -w
```
Watch Pods appear **one at a time**, each reaching `Running` before the
next is created — contrast this deliberately against Chapter 8's
Deployment rollout, where multiple Pods came up concurrently.

```bash
kubectl get pods -l app=db -o wide
kubectl get pvc
```
Confirm three separate PVCs (`data-db-0`, `data-db-1`, `data-db-2`), one
per ordinal — and three Pods named `db-0`, `db-1`, `db-2`, not random
suffixes.

**Prove individual DNS addressability** — the headless Service's whole
point:
```bash
kubectl run tmp --rm -it --image=busybox -- sh
# inside:
nslookup db-0.db-headless
nslookup db-1.db-headless
nslookup db-headless        # returns ALL pod IPs, since it's headless — no single VIP to hide behind
```

**Prove identity/storage survives Pod deletion** — write data to `db-0`
specifically, delete it, and confirm the replacement `db-0` gets the same
name and same data back:
```bash
kubectl exec db-0 -- psql -U postgres -c "CREATE TABLE proof (x int); INSERT INTO proof VALUES (42);"
kubectl delete pod db-0
kubectl wait --for=condition=ready pod db-0
kubectl exec db-0 -- psql -U postgres -c "SELECT * FROM proof;"   # still shows 42
kubectl get pvc | grep db-0   # SAME pvc, rebound to the new db-0 automatically
```
Compare mentally against Chapter 7: a ReplicaSet's replacement Pod gets a
*new* name and *no* attached history at all — this is the fundamental
difference StatefulSets exist to provide.

**Scale down and watch reverse-order removal:**
```bash
kubectl scale statefulset db --replicas=1
kubectl get pods -l app=db -w
```
`db-2` is removed first, then `db-1` — highest ordinal first, `db-0`
(commonly the "primary" by convention) preserved longest.

**Note PVCs are NOT deleted on scale-down** (a deliberate safety default —
scaling down shouldn't silently destroy that ordinal's data in case you
scale back up later):
```bash
kubectl get pvc   # data-db-1 and data-db-2 still exist, orphaned but not deleted
```

Cleanup:
```bash
kubectl delete -f db-statefulset.yaml -f db-headless-svc.yaml
kubectl delete pvc -l app=db
```

## Debugging Techniques

- **StatefulSet stuck, only `db-0` ever created** — with default
  `OrderedReady` management, a failing/not-ready `db-0` blocks every
  subsequent ordinal from ever being created; check `db-0`'s readiness
  probe (Chapter 17) and logs first, always, before assuming a broader
  problem.
- **PVCs piling up after repeated scale down/up cycles** — expected,
  by design (data preservation); clean up orphaned PVCs deliberately if
  you're sure you don't need that ordinal's data back.
- **Client can't resolve `db-1.db-headless`** — confirm the headless
  Service's `clusterIP: None` and that `serviceName` on the StatefulSet
  actually matches its name exactly; a common copy-paste mismatch.
- **Data "lost" after what looked like a routine Pod restart** — verify
  the StatefulSet's `volumeClaimTemplates` is actually configured (versus
  accidentally using a plain Deployment + shared PVC, or no persistent
  storage at all) — this is the most consequential mistake to catch
  early, given it's a database.

## Best Practices

- Reach for a StatefulSet only when Pod identity/ordering genuinely
  matters — don't default to it "because it's for databases" if your
  actual workload has no ordering/identity requirement; the added
  complexity (headless Service, per-ordinal PVCs, ordered lifecycle) has
  a real operational cost.
- Seriously evaluate managed database services (Chapter 31's cloud
  context) before committing to operating a stateful system yourself via
  StatefulSet — replication, backup, and failover logic for real
  databases is substantial engineering that a managed service already
  solves.
- Always pair with a proper readiness probe (Chapter 17) — since
  `OrderedReady` blocks subsequent ordinals on it, a misconfigured probe
  here has a bigger blast radius than on a Deployment.
- Never assume PVCs are cleaned up automatically on scale-down — decide
  and document your team's policy for orphaned per-ordinal PVCs
  explicitly.

## Interview Questions

1. **What does a StatefulSet guarantee that a Deployment doesn't?**
   Stable, predictable per-Pod identity (fixed name/ordinal, individually
   resolvable DNS via a headless Service) and stable per-ordinal storage
   (a dedicated PVC that follows that ordinal across recreation), plus
   ordered, sequential Pod creation/scaling.
2. **What is a headless Service, and why does a StatefulSet need one?**
   A Service with `clusterIP: None` — instead of hiding Pods behind one
   load-balanced virtual IP, it lets each Pod be resolved individually by
   its own DNS name, which is essential when clients need to reach a
   *specific* instance (e.g., a database primary) rather than "any
   instance."
3. **What happens to a StatefulSet Pod's PVC when the Pod is deleted and
   recreated?** It's rebound to the same PVC (and therefore the same
   underlying data) automatically — the replacement Pod for ordinal N
   always gets ordinal N's original storage back.
4. **What happens to per-ordinal PVCs when you scale a StatefulSet
   down?** They are not deleted automatically — orphaned PVCs remain, by
   design, so scaling back up later can recover that ordinal's original
   data; cleaning them up is a deliberate separate action.
5. **When would you choose a Deployment over a StatefulSet for a
   database-adjacent workload?** When instances are genuinely
   interchangeable/stateless — e.g., a read-through cache with no
   persistent per-instance state, or a stateless API layer in front of a
   separately-managed database.

## Mini Assignment

Scale your 3-replica StatefulSet down to 1 and back up to 3, and predict,
before checking, whether `db-1` and `db-2` come back with their *original*
data intact (given PVCs aren't deleted on scale-down) or start fresh.
Verify your prediction using the same `CREATE TABLE`/`INSERT`/`SELECT`
proof-of-data technique from the hands-on lab, applied to `db-1`
specifically.

## Lesson Summary

- StatefulSets provide stable per-Pod identity (name + DNS via a headless
  Service) and stable per-ordinal storage (individual PVCs via
  `volumeClaimTemplates`), plus ordered, sequential lifecycle management —
  everything a Deployment deliberately does not guarantee.
- Use them only when identity/ordering genuinely matters — databases,
  coordination systems — not as a default "more advanced" choice.
- PVCs survive both Pod deletion and StatefulSet scale-down, by design,
  as a data-preservation safety default.
- Consider managed database services as a legitimate alternative to
  operating this complexity yourself.

---

### Before Chapter 22 (Resource Requests and Limits) — tell me:

1. Why can't a normal (non-headless) Service give you what a StatefulSet
   client needs when it must reach one *specific* instance?
2. What's the practical consequence of `db-0`'s readiness probe failing
   under the default `OrderedReady` policy?
3. From the Mini Assignment — did `db-1`'s data survive the scale-down
   -then-up cycle, matching your prediction?
