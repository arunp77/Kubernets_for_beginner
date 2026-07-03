# Chapter 15 — Storage Classes

## What it is

A **StorageClass** is a template that tells Kubernetes *how* to
dynamically create a PersistentVolume on demand, instead of an admin
hand-creating PVs ahead of time (Chapter 14's approach). It names a
**provisioner** (a plugin — cloud-disk driver, network-storage driver,
etc.) plus parameters that provisioner needs (disk type, replication,
filesystem).

## Why it exists

Chapter 14's static approach doesn't scale operationally: an admin
manually pre-creating PVs for every possible size/type a team might need,
guessing capacity ahead of time, is exactly the kind of manual toil
Kubernetes exists to eliminate (Chapter 1's whole thesis). A StorageClass
flips this: a developer just creates a PVC naming a StorageClass, and a
**provisioner controller** watches for unbound PVCs referencing it and
creates a matching PV automatically, on the spot, backed by real
newly-provisioned storage (an actual new cloud disk, e.g.) — no admin
in the loop per-request.

## When to use it

Virtually always, in any real (non-toy) cluster — dynamic provisioning
via StorageClasses is the default, expected way persistent storage is
requested in production. Static PVs (Chapter 14) are reserved for
pre-existing storage you need to import into Kubernetes' management
(migrating existing data, or storage types with no dynamic provisioner
available).

## Internal architecture

- A **provisioner** is typically a CSI (Container Storage Interface)
  driver — CSI is to storage what CRI (Chapter 1) is to container
  runtimes: a standard plugin interface so Kubernetes core doesn't need
  built-in code for every possible storage backend. Cloud providers ship
  their own CSI drivers (`ebs.csi.aws.com`, `pd.csi.storage.gke.io`, etc).
- When a PVC names a StorageClass and no matching PV exists yet, the
  **provisioner controller** (watching, exactly like every other
  controller in this course, via the API server) calls out to the actual
  storage backend to create new physical storage, then creates a PV
  object representing it, and binds it to the waiting PVC — all
  automatically, typically within seconds.
- **`volumeBindingMode`** matters more than it looks: `Immediate`
  provisions storage the moment the PVC is created; `WaitForFirstConsumer`
  delays provisioning until a Pod actually using the PVC is scheduled —
  critical in multi-zone clusters, since it lets the *scheduler's* node
  choice inform *where* the storage is provisioned (e.g., in the same
  availability zone as the chosen node), rather than provisioning storage
  in a zone the Pod then can't be scheduled into.
- A cluster can mark one StorageClass as the **default**
  (`storageclass.kubernetes.io/is-default-class: "true"` annotation) — a
  PVC that specifies no `storageClassName` at all uses whichever
  StorageClass is marked default.

## YAML Definition

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-ssd
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: rancher.io/local-path      # Kind's built-in dynamic provisioner
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer
```
- `provisioner` — which plugin handles requests for this class; Kind
  ships a simple built-in one (`rancher.io/local-path`) purely so dynamic
  provisioning works out-of-the-box locally without any cloud account —
  in a real cluster this would be your cloud provider's CSI driver name.
- `reclaimPolicy` — same meaning as Chapter 14's PV field; here it's the
  default applied to every PV this class dynamically creates.
- `volumeBindingMode` — as explained above.

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: whoami-dyn-pvc
spec:
  storageClassName: fast-ssd
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```
The only new thing versus Chapter 14's PVC: `storageClassName`. No PV was
pre-created anywhere — that's the entire point.

## Hands-on Example

```bash
kubectl get storageclass
```
Kind ships a default StorageClass called `standard` out of the box —
confirm it's marked default:
```bash
kubectl get sc standard -o jsonpath='{.metadata.annotations}'
```

**Dynamic provisioning, live** — create only a PVC, no PV at all:
```bash
kubectl apply -f whoami-dyn-pvc.yaml
kubectl get pvc whoami-dyn-pvc
kubectl get pv    # a PV now exists — YOU never created it
```
Compare this directly against Chapter 14, where you had to author the PV
yourself first — this is the entire value proposition, demonstrated.

**Use `WaitForFirstConsumer` to prove binding timing matters:**
```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: delayed
provisioner: rancher.io/local-path
volumeBindingMode: WaitForFirstConsumer
```
```bash
kubectl apply -f delayed-sc.yaml
kubectl apply -f - <<'EOF'
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: delayed-pvc
spec:
  storageClassName: delayed
  accessModes: ["ReadWriteOnce"]
  resources: { requests: { storage: 1Gi } }
EOF
kubectl get pvc delayed-pvc    # STATUS: Pending — waiting for a consumer, not an error
```
Notice it stays `Pending` — not stuck, *waiting*, by design — until a Pod
actually mounts it:
```bash
kubectl run consumer --image=busybox --restart=Never --overrides='
{"spec":{"containers":[{"name":"consumer","image":"busybox","command":["sleep","3600"],"volumeMounts":[{"mountPath":"/data","name":"v"}]}],"volumes":[{"name":"v","persistentVolumeClaim":{"claimName":"delayed-pvc"}}]}}'
kubectl get pvc delayed-pvc   # now Bound
```

Cleanup:
```bash
kubectl delete pod consumer
kubectl delete pvc whoami-dyn-pvc delayed-pvc
kubectl delete sc fast-ssd delayed
```

## Debugging Techniques

- **PVC stuck `Pending` with `volumeBindingMode: Immediate`** — check
  `kubectl describe pvc` for the provisioner's actual error (quota
  exceeded on the cloud side, invalid parameters, wrong zone) — this is a
  real provisioning failure, not expected waiting.
- **PVC `Pending` with `WaitForFirstConsumer`** — expected and harmless
  until a Pod referencing it is actually scheduled; don't mistake this for
  a failure.
- **`no persistent volumes available for this claim and no storage class
  is set`** — no `storageClassName` specified and no default StorageClass
  exists on the cluster; either set one explicitly or mark a class
  default.
- **Provisioned storage lands in the "wrong" availability zone,
  Pod can't be scheduled** — classic symptom of using `Immediate` binding
  mode in a multi-zone cluster; switching to `WaitForFirstConsumer` is
  the standard fix.

## Best Practices

- Default to dynamic provisioning (StorageClasses) for everything in
  production — reserve static PVs (Chapter 14) for importing pre-existing
  storage only.
- Use `WaitForFirstConsumer` as your default binding mode on any
  multi-zone cluster — it's rarely wrong and avoids an entire class of
  scheduling/zone mismatch bugs.
- Mark exactly one StorageClass as default per cluster — having zero
  causes the "no storage class is set" error above; having more than one
  marked default is rejected/ambiguous.
- Match StorageClass choice to actual workload needs — cheaper/slower
  storage for logs or scratch-like persistent data, faster (and pricier)
  storage for databases; this is a real, ongoing cost-vs-performance
  decision, not a one-time setup detail.

## Interview Questions

1. **What problem does a StorageClass solve versus statically creating
   PersistentVolumes?** It eliminates the need for an admin to
   pre-provision storage manually — a provisioner controller creates
   real storage and a matching PV automatically the moment a PVC
   references the class, on demand.
2. **What is a CSI driver?**
   A Container Storage Interface plugin — a standard interface letting
   any storage backend (cloud disks, NFS, etc.) integrate with
   Kubernetes' dynamic provisioning without needing backend-specific code
   built into Kubernetes core, analogous to CRI for container runtimes.
3. **What's the difference between `Immediate` and `WaitForFirstConsumer`
   volume binding modes?** `Immediate` provisions storage as soon as the
   PVC is created; `WaitForFirstConsumer` waits until a Pod using the PVC
   is actually scheduled, so storage can be provisioned in the same zone
   the scheduler picked — important in multi-zone clusters.
4. **What happens if a PVC specifies no StorageClass and there's no
   default one set?** It fails to bind, with an explicit "no storage
   class is set" error — dynamic provisioning has no class to use.
5. **When would you still use a static PersistentVolume instead of a
   StorageClass?** When importing existing storage/data into Kubernetes,
   or when using a storage type with no available dynamic provisioner.

## Mini Assignment

Create two StorageClasses with different `volumeBindingMode` values, and
for each, create a PVC referencing it *before* creating any Pod that uses
it. Observe and record the `STATUS` of each PVC immediately after
creation (one should bind right away, one should stay `Pending`), then
create a consuming Pod for the pending one and confirm it binds only at
that point.

## Lesson Summary

- StorageClasses turn Chapter 14's manual "admin pre-creates a PV" model
  into automatic, on-demand provisioning via a CSI provisioner —
  eliminating per-request admin toil.
- `volumeBindingMode: WaitForFirstConsumer` defers provisioning until a
  Pod is actually scheduled, avoiding zone-mismatch problems `Immediate`
  binding can cause on multi-zone clusters.
- Exactly one StorageClass should be marked default per cluster; PVCs
  omitting `storageClassName` use it automatically.
- In real production clusters, dynamic provisioning via StorageClasses,
  not static PVs, is the default and expected approach.

---

### Before Chapter 16 (Ingress) — tell me:

1. Why does `WaitForFirstConsumer` matter specifically in a multi-zone
   cluster, and not really in a single-zone one?
2. What's the practical difference between a `Pending` PVC that's
   waiting-as-designed versus one that's genuinely stuck?
3. From the Mini Assignment — which binding mode's PVC bound immediately,
   and which waited for the consumer Pod?
