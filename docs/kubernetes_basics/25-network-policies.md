# Chapter 25 — Network Policies

## What it is

A **NetworkPolicy** restricts which Pods may communicate with which other
Pods (and external addresses), at the network layer — the piece that
finally provides the network isolation Chapter 11 explicitly warned you
Namespaces do **not** give you by default.

## Why it exists

Chapter 9 established that any Pod can reach any other Pod's IP/port by
default, cluster-wide — flat networking, no boundaries. That's simple and
convenient for getting things working, and completely wrong from a
security standpoint at any real scale: a compromised low-privilege Pod
(e.g., a public-facing web frontend with a vulnerability) should not be
able to open a connection directly to your database Pods, your internal
admin API, or anything else it has no legitimate reason to reach.
NetworkPolicy is Kubernetes' native mechanism for **micro-segmentation** —
enforcing "least privilege," but for network reachability instead of API
permissions (RBAC, Chapter 24's equivalent concern one layer up).

## When to use it

Any cluster handling real traffic, especially multi-tenant ones (Chapter
11's shared-cluster scenario) or anything security-sensitive (payment
processing, PII, internal admin tooling). A common, strong baseline:
**default-deny all ingress**, then explicitly allow only the specific
paths traffic legitimately needs to take.

## Internal architecture

- **NetworkPolicy is enforced by the CNI plugin, not the API server or
  kube-proxy.** Creating a NetworkPolicy object with no compatible CNI
  installed does **nothing** — a silent, dangerous trap for the unwary,
  since `kubectl apply` succeeds either way. Kind's default CNI
  (`kindnet`) does **not** enforce NetworkPolicies — you must swap in a
  policy-capable CNI (Calico is the standard choice, both for Kind labs
  and in real clusters) for this chapter's lab to actually do anything.
- A NetworkPolicy selects a set of Pods via `podSelector` (Chapter 10's
  mechanism, again) and then defines `ingress` (incoming) and/or `egress`
  (outgoing) rules that whitelist specific sources/destinations.
- **Policies are additive, not first-match** — if *any* NetworkPolicy
  selects a given Pod for `ingress`, that Pod's incoming traffic becomes
  **default-deny** except what's explicitly allowed by the union of every
  policy that selects it. A Pod selected by zero NetworkPolicies remains
  fully open (the Chapter 9 default) — this "selecting a Pod flips it
  from open to deny-by-default-except-listed" behavior is the single most
  important mental model in this chapter.
- Rules can select allowed peers by `podSelector` (other Pods, optionally
  scoped further by `namespaceSelector`), `namespaceSelector` alone
  (whole namespaces), or `ipBlock` (raw CIDR ranges, for external
  traffic) — mixing and matching these gives fairly expressive
  micro-segmentation without needing per-Pod IP management (which,
  recall Chapter 5, would be pointless anyway given ephemeral Pod IPs).

## YAML Definition

```yaml
# 1. Default-deny all ingress in a namespace — the recommended baseline
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
  namespace: default
spec:
  podSelector: {}
  policyTypes:
    - Ingress
---
# 2. Explicitly allow only what's needed on top of that baseline
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-to-backend
  namespace: default
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: frontend
      ports:
        - protocol: TCP
          port: 8080
```
- `spec.podSelector: {}` (empty) on the deny-all policy — an empty
  selector matches **every** Pod in the namespace, meaning this policy's
  deny-by-default reach applies cluster-namespace-wide once created.
- `policyTypes: [Ingress]` — this policy only governs incoming traffic;
  `Egress` (outgoing) is a separate, independently-declarable type — a
  policy can cover either or both.
- The second policy's `podSelector: {matchLabels: {app: backend}}` —
  targets only Pods labeled `app: backend`; only *those* Pods become
  subject to this policy's rules (in addition to the namespace-wide
  deny-all above).
- `ingress[].from[].podSelector` — allows traffic specifically from Pods
  labeled `app: frontend`, on TCP port 8080 only — everything else
  remains denied by policy #1.
- Note both policies **compose**: a `backend` Pod ends up allowing
  ingress *only* from `frontend` Pods on port 8080, and nothing else at
  all — the deny-all baseline plus one narrow allow-rule.

## Hands-on Example

**1. Install a policy-enforcing CNI** (replace Kind's default, a
one-time cluster-level setup — check the Calico-for-Kind quickstart for
the exact current manifest URL, as it's versioned):
```bash
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.28.0/manifests/calico.yaml
kubectl wait --for=condition=ready pod -l k8s-app=calico-node -n kube-system --timeout=180s
```

**2. Prove the Chapter 9/11 flat-network default first**, before adding
any policy:
```bash
kubectl create deployment backend --image=traefik/whoami
kubectl expose deployment backend --port=8080 --target-port=80
kubectl run attacker --image=busybox --labels="role=untrusted" -- sleep 3600
kubectl exec attacker -- wget -qO- --timeout=2 backend:8080   # succeeds — flat network, no restriction yet
```

**3. Apply default-deny, then re-test:**
```bash
kubectl apply -f default-deny-ingress.yaml
kubectl exec attacker -- wget -qO- --timeout=2 backend:8080   # now times out — denied
```

**4. Apply the narrow allow-rule, but from a Pod that ISN'T labeled
`frontend`, proving the allow-rule is genuinely selective:**
```bash
kubectl apply -f allow-frontend-to-backend.yaml
kubectl exec attacker -- wget -qO- --timeout=2 backend:8080   # STILL denied — attacker isn't labeled app=frontend
```

**5. Now prove the legitimate path works:**
```bash
kubectl run frontend --image=busybox --labels="app=frontend" -- sleep 3600
kubectl exec frontend -- wget -qO- --timeout=2 backend:8080   # succeeds
```

This four-step sequence — open by default, denied by default-deny,
still denied for the wrong identity, allowed for the right one — is the
complete mental model this chapter teaches, demonstrated rather than just
asserted.

Cleanup:
```bash
kubectl delete deployment backend
kubectl delete pod attacker frontend
kubectl delete svc backend
kubectl delete networkpolicy default-deny-ingress allow-frontend-to-backend
```

## Debugging Techniques

- **NetworkPolicy applied, but traffic is still unrestricted** — first
  check whether your CNI actually **enforces** NetworkPolicy at all
  (Kind's default `kindnet` silently does not); `kubectl apply` succeeding
  is not evidence the policy is doing anything.
- **Legitimate traffic unexpectedly blocked** — check every NetworkPolicy
  selecting the destination Pod (there may be more than one, all
  additive) and confirm the source Pod's actual labels
  (`kubectl get pod <name> --show-labels`) genuinely match what an
  `ingress[].from[].podSelector` expects — label typos are, again
  (Chapter 10), the most common root cause.
- **DNS itself breaks after adding a default-deny egress policy** — a
  very common real incident: denying all egress also blocks a Pod's
  ability to reach CoreDNS (Chapter 9); a real default-deny-egress
  baseline must explicitly allow egress to CoreDNS (typically port 53
  UDP/TCP to the `kube-system` namespace) or literally everything breaks.
- **Policy seems to apply to the wrong Pods** — double check
  `spec.podSelector` on the policy itself (which Pods it *restricts*) is
  not confused with `ingress[].from[].podSelector` (which Pods are
  *allowed in*) — these are easy to mix up when scanning YAML quickly.

## Best Practices

- Start every namespace with a **default-deny-ingress** (and, once
  you've accounted for DNS, often default-deny-egress too) baseline, then
  add narrow, explicit allow rules — deny-by-default is dramatically
  safer than trying to enumerate every bad actor to block individually.
- Always explicitly allow DNS egress (to CoreDNS) before enabling any
  default-deny-egress policy — forgetting this breaks nearly everything
  in confusing, hard-to-diagnose ways.
- Choose a CNI with NetworkPolicy support as a hard requirement when
  selecting infrastructure for any real cluster — verify this explicitly;
  it's not universal across every CNI plugin.
- Combine with RBAC (Chapter 24) and Namespaces (Chapter 11) as layered
  defenses — network segmentation, API authorization, and namespace
  scoping are three independent controls, not substitutes for each
  other.

## Interview Questions

1. **Do Namespaces provide network isolation on their own?**
   No (revisited directly from Chapter 11) — NetworkPolicies are the
   actual mechanism for restricting Pod-to-Pod network traffic; namespace
   boundaries alone allow fully open traffic across them by default.
2. **What enforces a NetworkPolicy — the API server, kube-proxy, or
   something else?** The CNI plugin. The API server only stores the
   object; if the installed CNI doesn't implement NetworkPolicy
   enforcement, the policy object exists but has zero actual effect.
3. **What happens to a Pod's traffic the moment any NetworkPolicy
   selects it for `Ingress`?** It flips from fully open (the cluster
   default) to deny-by-default, allowing only what's explicitly permitted
   by the union of every NetworkPolicy that selects it — Pods selected by
   no policy remain fully open.
4. **What's a common, serious mistake when adopting default-deny-egress
   policies?** Forgetting to explicitly allow egress to CoreDNS — without
   it, Pods lose DNS resolution entirely, breaking nearly all outbound
   communication in a confusing way.
5. **How would you allow traffic only from Pods in a specific other
   namespace, not just specific Pods?** Use `namespaceSelector` in the
   `ingress[].from[]` rule (optionally combined with `podSelector` for an
   AND condition), rather than `podSelector` alone which only matches
   within the same namespace by default.

## Mini Assignment

Set up three Pods labeled `tier: frontend`, `tier: backend`, and
`tier: database` respectively (busybox is fine, no need for real
services). Write NetworkPolicies enforcing: frontend can reach backend,
backend can reach database, but frontend can **not** reach database
directly. Verify all three reachability outcomes explicitly with `wget
--timeout` calls between each pair, not just the ones you expect to
succeed — the negative tests (confirming denial) are the actual point of
the exercise.

## Lesson Summary

- NetworkPolicies restrict Pod-to-Pod (and Pod-to-external) network
  traffic, filling exactly the gap Chapter 11 warned about: Namespaces
  alone provide no network isolation.
- Enforcement happens in the CNI plugin, not the API server — a policy
  can be successfully created and do absolutely nothing if the CNI
  doesn't support it (Kind's default doesn't; Calico does).
- Once any policy selects a Pod for a given direction (ingress/egress),
  that traffic direction becomes deny-by-default except what's
  explicitly allowed — Pods selected by no policy stay fully open.
- Default-deny-ingress (and, carefully, egress with DNS explicitly
  allowed) is the standard, strong baseline production posture.

---

### Before Chapter 26 (Helm) — tell me:

1. Why can `kubectl apply -f my-network-policy.yaml` succeed with no
   error, yet have zero actual effect on traffic?
2. What happens to a Pod's network reachability the instant a
   NetworkPolicy selects it, versus before any policy selected it?
3. From the Mini Assignment — did you verify the *denied* path (frontend
   to database) explicitly, and did it actually fail as expected?
