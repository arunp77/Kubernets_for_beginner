# Chapter 13 — Secrets

## What it is

A **Secret** is, mechanically, almost identical to a ConfigMap (Chapter
12) — key/value data injectable as env vars or mounted files — but
intended for sensitive data (passwords, tokens, TLS certs, registry
credentials), stored **base64-encoded** by default, with distinct RBAC
treatment and integration points for real encryption-at-rest.

## Why it exists

Splitting Secrets from ConfigMaps as a *distinct type* — even though the
consumption mechanics barely differ — lets Kubernetes and its ecosystem
apply different handling by type: RBAC rules that grant "read ConfigMaps"
without also granting "read Secrets," admission controllers and audit
tooling that flag Secret access specially, and cluster-level encryption
options that specifically target `secrets` in etcd. If sensitive data
lived in ConfigMaps, none of that type-based policy would be possible.

## When to use it

Passwords, API keys, tokens, TLS certificates, and private
registry credentials (`imagePullSecrets`) — anything where exposure is a
security incident, not just a misconfiguration.

## Internal architecture — the fact everyone gets wrong at first

**Base64 is encoding, not encryption.** A Secret's `data` field is
base64-encoded purely so arbitrary binary content (like a TLS
certificate) can be represented safely in YAML/JSON text — it provides
**zero confidentiality**. Anyone with `kubectl get secret -o yaml` access
(or, in a production cluster, anyone with read access to etcd) can decode
it in one command: `echo <value> | base64 -d`. This is precisely why:

- **RBAC on Secrets matters enormously** (Chapter 24) — the actual
  security boundary is "who can read the Secret object via the API,"
  not the encoding.
- Production clusters should enable **encryption at rest for etcd**
  specifically for the `secrets` resource — otherwise anyone with etcd
  filesystem/backup access reads them in the clear, base64 notwithstanding.
- Many teams instead use **external secret managers** (AWS Secrets
  Manager, HashiCorp Vault, GCP Secret Manager) with an operator (like
  External Secrets Operator) that syncs values *into* Kubernetes Secrets
  at runtime — keeping the actual sensitive values out of git and out of
  etcd's plaintext-adjacent storage as much as possible.

Beyond that caveat, everything from Chapter 12 about env-var vs.
volume-mount update behavior applies identically here.

## YAML Definition

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: whoami-secret
type: Opaque
data:
  DB_PASSWORD: c3VwZXJzZWNyZXQ=      # base64 of "supersecret"
stringData:
  API_TOKEN: "plain-text-in-the-manifest-but-still-stored-base64"
```
- `type: Opaque` — the generic catch-all type (arbitrary key/value data);
  other built-in types exist for specific shapes: `kubernetes.io/
  dockerconfigjson` (registry credentials), `kubernetes.io/tls` (cert +
  key pair), `kubernetes.io/basic-auth`, etc. — the type mainly affects
  what fields are expected/validated, not how confidentially it's stored.
- `data` — values you must pre-base64-encode yourself before writing the
  YAML (`echo -n supersecret | base64`).
- `stringData` — a write-only convenience field: you write plain text
  here, and the API server base64-encodes it into `data` for you on
  creation — purely for manifest ergonomics, it does **not** change how
  the value is stored (still base64 in `data` afterward, still not
  encrypted).

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
          envFrom:
            - secretRef:
                name: whoami-secret
```
`secretRef` mirrors Chapter 12's `configMapRef` exactly — same injection
mechanism, different source type.

## Hands-on Example

```bash
echo -n "supersecret" | base64
kubectl apply -f whoami-secret.yaml
kubectl get secret whoami-secret -o yaml
```

**Prove base64 is not encryption, hands-on** — this is the point of the
entire chapter:
```bash
kubectl get secret whoami-secret -o jsonpath='{.data.DB_PASSWORD}' | base64 -d
```
No key, no password, no special privilege beyond ordinary read access —
the plaintext comes straight back.

```bash
kubectl apply -f whoami-deploy.yaml
kubectl exec deploy/whoami -- env | grep -E 'DB_PASSWORD|API_TOKEN'
```
Notice the env vars *inside the container* are plain text, decoded
automatically by the kubelet at injection time — your application never
has to base64-decode anything itself.

**Mount as files instead** (common for TLS certs, which don't fit neatly
into a single env var):
```bash
kubectl patch deployment whoami --type='json' -p='[{"op":"add","path":"/spec/template/spec/volumes","value":[{"name":"secret-vol","secret":{"secretName":"whoami-secret"}}]},{"op":"add","path":"/spec/template/spec/containers/0/volumeMounts","value":[{"name":"secret-vol","mountPath":"/etc/secret"}]}]'
kubectl exec deploy/whoami -- ls /etc/secret
kubectl exec deploy/whoami -- cat /etc/secret/DB_PASSWORD
```
Files mounted from a Secret are also automatically decoded — plain text
on disk inside the container, not base64.

**Create an `imagePullSecret`** (the practical real-world use you'll need
for CI/CD in Chapter 29, if pulling from a private registry):
```bash
kubectl create secret docker-registry regcred \
  --docker-server=<your-registry> \
  --docker-username=<user> \
  --docker-password=<password>
```
Then reference it in a Pod spec via `spec.imagePullSecrets: [{name:
regcred}]` so the kubelet can authenticate when pulling a private image.

Cleanup:
```bash
kubectl delete -f whoami-deploy.yaml -f whoami-secret.yaml
kubectl delete secret regcred
```

## Debugging Techniques

- **`ImagePullBackOff` pulling from a private registry** — almost always
  a missing or incorrectly-referenced `imagePullSecrets`; `kubectl
  describe pod` shows an explicit auth-failure reason in Events.
- **Secret value looks "wrong" in the Pod** — check whether you used
  `data` (must be pre-base64-encoded by you) versus `stringData` (plain
  text you write, auto-encoded) — mixing them up is a very common source
  of double-encoded or garbled values.
- **Someone asks "is this secret actually secure?"** — the honest answer:
  only as secure as (a) RBAC restricting who can `get`/`list` Secret
  objects, and (b) whether etcd encryption-at-rest is enabled for the
  `secrets` resource. Base64 alone provides none of that.
- **`kubectl get secret -o yaml` in a shared terminal/screen-share** — a
  very real, very common accidental-leak vector; be as careful running
  this command live as you would be pasting a password into chat.

## Best Practices

- Never commit raw Secret YAML (with real `data`/`stringData` values) to
  git — use tools like Sealed Secrets, SOPS, or an external secret
  manager integration so only encrypted or reference values ever reach
  version control.
- Enable etcd encryption-at-rest for Secrets in any real cluster —
  ask this explicitly when evaluating a managed Kubernetes offering
  (Chapter 31 — this is enabled by default on most, but verify).
- Scope RBAC tightly around `secrets` (Chapter 24) — "can read
  ConfigMaps" and "can read Secrets" should never be granted as a bundle
  by habit.
- Prefer mounted-file delivery over env vars for highly sensitive values
  where practical — env vars are more likely to leak accidentally (e.g.,
  dumped in a crash log, visible via `/proc/<pid>/environ` to anything
  with host access, or accidentally logged by application code that
  prints its environment for debugging).

## Interview Questions

1. **Is a Kubernetes Secret encrypted?**
   Not by default — its `data` is base64-*encoded*, which is trivially
   reversible by anyone with read access to the object; real
   confidentiality depends on RBAC restricting access and (for a fully
   hardened cluster) enabling etcd encryption-at-rest for the `secrets`
   resource.
2. **What's the actual difference between a ConfigMap and a Secret,
   mechanically?** Almost none in terms of injection (env var/volume
   mount work identically) — the difference is intent-signaling and
   ecosystem/RBAC treatment: tools and policies can target `secrets`
   specifically for stricter access control and encryption.
3. **What's the difference between `data` and `stringData` in a Secret
   manifest?** `data` requires values you've pre-base64-encoded yourself;
   `stringData` lets you write plain text, which the API server encodes
   into `data` automatically on creation — purely a manifest-writing
   convenience, not a security difference.
4. **How would you let a Pod pull an image from a private registry?**
   Create a `kubernetes.io/dockerconfigjson`-type Secret (via `kubectl
   create secret docker-registry`) and reference it in the Pod's
   `spec.imagePullSecrets`.
5. **What real-world practice do many teams adopt instead of storing raw
   secret values in Kubernetes manifests/git?** Using an external secret
   manager (Vault, AWS/GCP Secret Manager) combined with an operator that
   syncs values into Kubernetes Secrets at runtime, or encrypting secret
   manifests themselves before they ever reach git (Sealed Secrets, SOPS).

## Mini Assignment

Create a Secret with a fake database password, mount it into a Pod both
as an environment variable and as a mounted file simultaneously, and
confirm both surfaces show plain text (not base64) inside the running
container. Then run `kubectl get secret <name> -o yaml` yourself and
decode the raw `data` value by hand with `base64 -d` — walk through
exactly why this is not, by itself, a security control, and write down
what *would* actually need to be true for this Secret to be considered
properly protected in a real cluster.

## Lesson Summary

- Secrets are mechanically almost identical to ConfigMaps — the real
  difference is intent and ecosystem/RBAC treatment, not encryption.
- Base64 encoding is reversible by anyone with read access to the object;
  actual security comes from RBAC scoping and etcd encryption-at-rest,
  neither of which is automatic just because you used `kind: Secret`.
- `data` requires pre-encoded values; `stringData` is a plaintext
  convenience field, auto-encoded on write — both end up equally
  "unencrypted" once stored.
- Real production practice often layers external secret managers or
  encrypted-manifest tooling on top, specifically because raw Secret
  YAML shouldn't live in git.

---

### Before Chapter 14 (Persistent Volumes) — tell me:

1. In your own words, why is base64 encoding not a security control?
2. What two things would actually need to be true for Secrets in a real
   cluster to be considered properly protected?
3. From the Mini Assignment — could you decode the Secret's value
   yourself with nothing more than ordinary `kubectl get` access?
