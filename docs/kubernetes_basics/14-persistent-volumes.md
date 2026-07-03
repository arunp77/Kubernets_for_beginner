# Chapter 14 — Persistent Volumes

## What it is

A **PersistentVolume (PV)** represents a real piece of storage in the
cluster (a cloud disk, an NFS share, a local disk) as a Kubernetes object.
A **PersistentVolumeClaim (PVC)** is a Pod's *request* for storage meeting
certain criteria (size, access mode) — Kubernetes then *binds* a matching
PV to that claim. Pods never reference a PV directly; they always
reference a PVC, and the PVC is what actually gets bound to a PV.

## Why it exists

You know Docker volumes/bind mounts: `docker run -v mydata:/data` gives a
container persistent storage *on that specific host*. That single-host
assumption breaks in Kubernetes exactly like it did for networking
(Chapter 9) — a Pod can be rescheduled onto any node at any time, and
whatever storage it needs must be able to "follow" it (for network-backed
storage) or the scheduler must specifically place it back on a node that
still has the right local disk. Kubernetes needed an abstraction that
separates **"a Pod's request for storage"** from **"the actual storage
implementation"** — that's exactly the PVC/PV split, and it mirrors the
Service/Endpoints split from Chapter 9: a stable, generic front (PVC) over
a concrete, swappable backend (PV).

## When to use it

Any workload that needs data to survive a Pod being deleted/rescheduled —
databases, file uploads, anything that isn't safely disposable scratch
space (which is what `emptyDir`, Chapter 5, is for instead — that data
dies with the Pod, on purpose).

## Internal architecture

- **Static provisioning**: a cluster admin manually creates PV objects
  ahead of time, each describing real storage capacity, and PVCs bind to
  whichever PV satisfies their request. This is what we'll do by hand in
  this chapter, to see the mechanism plainly — Chapter 15's
  **StorageClasses** cover **dynamic provisioning**, where PVs are created
  automatically on demand instead, which is what real production clusters
  actually use almost exclusively.
- **Access modes** describe how many nodes can mount a volume
  simultaneously: `ReadWriteOnce` (RWO — one node at a time, the most
  common, e.g. typical cloud block storage), `ReadOnlyMany` (ROX — many
  nodes, read-only), `ReadWriteMany` (RWX — many nodes, read-write,
  needs storage that actually supports this, like NFS — most cloud block
  storage does *not*).
- **Reclaim policy** governs what happens to the underlying storage when
  its PVC is deleted: `Retain` (data kept, PV becomes `Released` and
  needs manual cleanup/reuse), `Delete` (underlying storage is actually
  destroyed — common default for dynamically-provisioned cloud disks),
  `Recycle` (deprecated, don't use).
- **Binding is one-to-one and (mostly) permanent**: once a PVC binds to a
  PV, that PV is claimed exclusively — even if another PVC's request
  would technically also fit, it won't be considered while the first
  binding holds.
- **PVCs are namespaced; PVs are cluster-scoped** (Chapter 11's
  distinction) — this is precisely why the claim/actual-resource split
  exists structurally, not just conceptually: a PVC lives inside your
  team's namespace, while the PV it binds to is a cluster-wide resource
  an admin manages.

## YAML Definition

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: whoami-pv
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  hostPath:
    path: /data/whoami-pv
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: whoami-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 500Mi
```
- `PersistentVolume.spec.capacity.storage` — total size this PV offers.
- `spec.accessModes` — must include whatever a claim will request; a PV
  can list several supported modes.
- `spec.persistentVolumeReclaimPolicy` — what happens on PVC deletion, as
  above.
- `spec.hostPath` — a **local-node-directory-backed** volume type, used
  here purely because it works trivially on a single-machine Kind
  cluster with zero cloud dependency; genuinely wrong for real multi-node
  production use (a Pod rescheduled to a *different* node wouldn't see
  the same host directory at all — the antithesis of the "storage should
  follow the Pod" goal this chapter opened with). Real clusters use
  cloud-block-storage or network-storage PV types instead (provisioned
  dynamically via StorageClasses, Chapter 15).
- `PersistentVolumeClaim.spec.resources.requests.storage` — the *minimum*
  size being requested; Kubernetes binds it to any PV with enough
  capacity, size need not match exactly.
- Note the PVC has no direct reference to `whoami-pv` by name — binding
  happens by Kubernetes matching accessModes/capacity/(optionally
  StorageClass, Chapter 15) automatically, another example of loose
  coupling rather than a hardcoded reference.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: whoami
spec:
  replicas: 1
  selector:
    matchLabels: { app: whoami }
  template:
    metadata:
      labels: { app: whoami }
    spec:
      containers:
        - name: whoami
          image: traefik/whoami
          volumeMounts:
            - name: data
              mountPath: /data
      volumes:
        - name: data
          persistentVolumeClaim:
            claimName: whoami-pvc
```
`volumes[].persistentVolumeClaim.claimName` — the Pod only ever names the
**PVC**, never the PV directly, exactly mirroring how a Service consumer
only ever names the Service, never a Pod IP.

## Hands-on Example

```bash
kubectl apply -f whoami-pv.yaml
kubectl apply -f whoami-pvc.yaml
kubectl get pv,pvc
```
Watch the PVC's `STATUS` become `Bound` and `VOLUME` show `whoami-pv` —
the binding happened automatically based on matching accessModes/capacity.

```bash
kubectl apply -f whoami-deploy.yaml
kubectl exec deploy/whoami -- sh -c "echo hello > /data/test.txt"
kubectl exec deploy/whoami -- cat /data/test.txt
```

**Prove data survives Pod deletion** — the entire point:
```bash
kubectl delete pod -l app=whoami
kubectl wait --for=condition=ready pod -l app=whoami
kubectl exec deploy/whoami -- cat /data/test.txt   # still "hello" — new Pod, same underlying data
```

**Prove `Retain` keeps data after the PVC is gone:**
```bash
kubectl delete -f whoami-deploy.yaml
kubectl delete pvc whoami-pvc
kubectl get pv whoami-pv     # STATUS is now "Released", not deleted
docker exec masterclass-control-plane cat /data/whoami-pv/test.txt   # data is genuinely still there on disk
```

Cleanup:
```bash
kubectl delete pv whoami-pv
```

## Debugging Techniques

- **PVC stuck `Pending`** — no PV satisfies its request (size too small,
  accessMode mismatch, or — in dynamic-provisioning clusters, Chapter 15
  — no matching/default StorageClass). `kubectl describe pvc <name>`
  names the exact reason.
- **`FailedMount` / `FailedAttachVolume` in Pod events** — commonly a Pod
  got scheduled onto a node that can't actually reach the underlying
  storage (relevant for network-attached storage across availability
  zones on real clouds) — check node/zone placement versus where the PV
  actually lives.
- **Data "disappeared" after a Pod restart** — check whether the volume
  was genuinely a PVC-backed PersistentVolume, or accidentally an
  `emptyDir` (Chapter 5) — an extremely common copy-paste mistake between
  the two, since their `volumeMounts` block looks identical; only the
  `volumes[].<type>` block differs.
- **Can't delete a PV** — if its reclaim policy is `Retain` and a PVC
  still references it, or if there's a lingering finalizer, deletion will
  hang; check `kubectl describe pv` for the exact blocking condition.

## Best Practices

- Never use `hostPath` in real multi-node clusters — it silently breaks
  the moment a Pod is rescheduled to a different node; it exists here
  purely as a zero-dependency teaching tool on Kind.
  Use dynamically-provisioned cloud/network storage instead (Chapter 15).
- Default to `Retain` for anything genuinely important (databases) and
  `Delete` only for storage you're truly fine losing when its claim is
  removed — `Delete` as a default has caused real production data-loss
  incidents when someone deleted a PVC assuming it was harmless.
- Size PVC requests realistically, but remember many storage backends
  don't support shrinking afterward — growing is commonly supported,
  shrinking often isn't; plan capacity conservatively.
- Treat `ReadWriteMany` requirements as a flag to double-check your
  storage backend actually supports it — defaulting to expecting RWX like
  a shared NFS mount, on backends that only offer RWO, is a common
  planning mistake.

## Interview Questions

1. **What's the relationship between a PersistentVolume and a
   PersistentVolumeClaim?** A PV represents real, provisioned storage
   (cluster-scoped); a PVC is a Pod's namespaced request for storage
   meeting certain criteria. Kubernetes binds a PVC to a satisfying PV;
   Pods only ever reference the PVC, never the PV directly.
2. **Why can't Kubernetes just use bind mounts the way Docker does?**
   Because a Pod can be rescheduled onto any node at any time; storage
   tied to one host's filesystem wouldn't "follow" the Pod the way
   network-backed persistent storage can (or the scheduler must pin the
   Pod back to the right node, which local-storage setups do support but
   at the cost of that flexibility).
3. **What are the three PersistentVolume access modes?**
   ReadWriteOnce (one node, read-write), ReadOnlyMany (many nodes,
   read-only), ReadWriteMany (many nodes, read-write — requires storage
   that genuinely supports concurrent multi-node writes).
4. **What's the difference between `Retain` and `Delete` reclaim
   policies?** `Retain` keeps the underlying storage (and its data) after
   the PVC is deleted, marking the PV `Released` for manual handling;
   `Delete` actually destroys the underlying storage automatically —
   a meaningful, sometimes irreversible difference to get right.
5. **A PVC is stuck in `Pending` — what do you check first?**
   `kubectl describe pvc` for the exact reason — usually no PV (or, with
   dynamic provisioning, no StorageClass) matches the requested size/
   accessMode.

## Mini Assignment

Create two PVs with different capacities (`500Mi` and `2Gi`) and one PVC
requesting `1Gi`. Predict, before running anything, which PV it will bind
to and why (hint: think about what "satisfies a request" means given the
accessModes/capacity matching rule — it isn't necessarily "the smallest
one that fits"). Apply it and check `kubectl get pv,pvc` to confirm or
correct your prediction.

## Lesson Summary

- PersistentVolumes represent real storage; PersistentVolumeClaims are a
  Pod's request for storage, bound automatically to a satisfying PV —
  the same stable-front/swappable-backend pattern as Services/Endpoints.
- Pods only ever reference a PVC by name, never a PV — loose coupling,
  consistent with everything else in this course so far.
- Access modes (RWO/ROX/RWX) and reclaim policy (Retain/Delete) are the
  two decisions with real, sometimes irreversible, production
  consequences.
- `hostPath` is a single-node teaching shortcut only — real clusters use
  dynamically-provisioned storage, which Chapter 15 covers next.

---

### Before Chapter 15 (Storage Classes) — tell me:

1. Why does a Pod reference a PVC instead of a PV directly — what pattern
   from an earlier chapter does this mirror?
2. What's the practical, real-world risk of defaulting a PV's reclaim
   policy to `Delete`?
3. Which PV did your PVC actually bind to in the Mini Assignment, and did
   it match your prediction?
