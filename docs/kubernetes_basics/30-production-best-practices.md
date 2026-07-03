# Chapter 30 — Production Best Practices

## Lesson Objectives

This chapter is a synthesis, not a new object type — by now you've met
every mechanism it references. The goal is to consolidate 29 chapters'
worth of individual best-practice call-outs into one coherent
"production-readiness" mental checklist, organized by failure mode
instead of by object type, and to be explicit about the handful of
cross-cutting decisions that don't belong to any single chapter.

## Theory — thinking in failure modes, not features

Every chapter so far taught you a mechanism and, along the way, called
out how to misuse it. Production readiness is really the discipline of
asking, for every workload you deploy: **"which of these failure modes
have I actually addressed, on purpose, versus left to chance?"** Group
them like this:

**1. "A Pod dies — does anything notice, and does it matter?"**
- Are you running this via a Deployment/StatefulSet (Ch. 7, 8, 21), never
  a bare Pod (Ch. 5)?
- Do you have a correct, non-conflated liveness vs. readiness probe
  (Ch. 17) — not checking downstream dependencies in liveness?
- Is `replicas` (and, if used, HPA's `minReplicas`, Ch. 23) at least 2 for
  anything user-facing, so a single Pod loss isn't a full outage?

**2. "A node dies — does anything notice, and does it matter?"**
- Do you have enough nodes, and enough spread (via
  `topologySpreadConstraints`/anti-affinity — a real production concept
  this course only gestures at) that losing one node doesn't take out
  every replica of a service at once?
- Is your storage (Ch. 14-15) genuinely able to follow a rescheduled Pod,
  or did you leave a `hostPath` shortcut (correct only for Ch. 3's local
  labs and Ch. 28's node-local logging) in a place it doesn't belong?

**3. "Traffic spikes — does the system adapt, or fall over?"**
- Are `resources.requests`/`limits` (Ch. 22) set deliberately, based on
  observed usage (Ch. 27), not guessed?
- Is HPA (Ch. 23) actually wired to a meaningful metric, with sane
  `min`/`maxReplicas` and asymmetric scale-up/scale-down stabilization?
- Does your Ingress/Service layer (Ch. 9, 16) have any rate limiting or
  is a single noisy client able to degrade everyone?

**4. "Something is broken right now — can you find out why, fast?"**
- Do you have centralized logs (Ch. 28) and metrics/dashboards (Ch. 27)
  covering both infrastructure *and* your own application's business
  metrics?
- Do you alert on user-visible symptoms (error rate, latency) as the
  primary signal, not just resource graphs?
- Is your team fluent in `describe`/`logs`/`get events` (Ch. 4-5) as a
  reflex, not a lookup?

**5. "Someone/something malicious or careless gets access — what can
they actually do?"**
- Is RBAC (Ch. 24) scoped to least privilege, with no casual
  `cluster-admin` bindings?
- Are NetworkPolicies (Ch. 25) enforcing default-deny with narrow allow
  rules, on a CNI that actually supports them?
- Are Secrets (Ch. 13) never committed to git in plaintext, with etcd
  encryption-at-rest enabled?
- Are container images scanned for known vulnerabilities, and are Pods
  running as non-root with a read-only root filesystem where possible
  (a `securityContext` concept this course has referenced implicitly but
  is worth naming explicitly here)?

**6. "A bad deploy goes out — how fast can you detect and undo it?"**
- Is your rollout strategy (Ch. 8) actually gated on real readiness
  probes, with `maxUnavailable`/`maxSurge` chosen deliberately?
- Is your CI/CD (Ch. 29) gating on rollout health, not just apply
  success, with an immutable image tag so rollback is unambiguous?
- Do you know, right now, without looking anything up, the exact command
  to roll back your most critical service?

## Architecture Diagram — the production-readiness stack, layered

```
   ┌─────────────────────────────────────────────────────────┐
   │  6. Deploy safety   → readiness-gated rollouts, GitOps,   │
   │                        immutable tags, rollback drills     │
   ├─────────────────────────────────────────────────────────┤
   │  5. Security         → RBAC least-privilege, NetworkPolicy,│
   │                        Secrets hygiene, non-root containers│
   ├─────────────────────────────────────────────────────────┤
   │  4. Observability    → metrics + logs + symptom-based alerts│
   ├─────────────────────────────────────────────────────────┤
   │  3. Elasticity        → requests/limits set from real data, │
   │                        HPA tuned, no single noisy neighbor  │
   ├─────────────────────────────────────────────────────────┤
   │  2. Node resilience    → spread across nodes, storage that  │
   │                        follows Pods, no hostPath misuse     │
   ├─────────────────────────────────────────────────────────┤
   │  1. Pod resilience     → controller-managed, correct probes,│
   │                        replicas ≥ 2 for anything user-facing│
   └─────────────────────────────────────────────────────────┘
   Each layer assumes the ones below it are solid — a beautifully
   tuned HPA (layer 3) is worthless if Pods have no readiness probe
   (layer 1) telling it when a new replica is actually usable.
```

## Docker Comparison

| Concern | Where Docker-alone leaves you | What "production-ready Kubernetes" adds |
|---|---|---|
| Pod resilience | `--restart=always`, same host only | Multi-node self-healing via controllers (Ch. 7-8) |
| Elasticity | Manual `docker compose up --scale` | HPA reacting to real metrics (Ch. 23) automatically |
| Observability | `docker logs`/`docker stats`, per host | Centralized metrics + logs across the whole fleet (Ch. 27-28) |
| Security | Host-level Linux permissions, manually managed | RBAC + NetworkPolicy + Secrets as first-class, auditable objects (Ch. 13, 24-25) |
| Deploy safety | Hand-choreographed stop/start (Lesson 1's lab) | Gated rolling updates + GitOps reconciliation (Ch. 8, 29) |

The throughline across this whole course: nearly every "production best
practice" is really just **using the mechanism from an earlier chapter
the way it was designed to be used**, rather than reaching for a new
tool. Production-readiness is discipline and completeness, not a
different technology stack.

## Internal Working — the audit, as a runnable exercise

Rather than new internals, this chapter's "internal working" is a
concrete audit script you can run against any real Deployment, tying
directly back to specific `kubectl` commands from earlier chapters:

```bash
# 1. Pod resilience
kubectl get deploy <name> -o jsonpath='{.spec.replicas}'
kubectl get deploy <name> -o jsonpath='{.spec.template.spec.containers[0].readinessProbe}'
kubectl get deploy <name> -o jsonpath='{.spec.template.spec.containers[0].livenessProbe}'

# 2. Node resilience
kubectl get pods -l app=<name> -o wide   # spread across distinct NODE values?

# 3. Elasticity
kubectl get deploy <name> -o jsonpath='{.spec.template.spec.containers[0].resources}'
kubectl get hpa | grep <name>

# 4. Observability
kubectl get servicemonitor | grep <name>   # (if using Prometheus Operator)

# 5. Security
kubectl get deploy <name> -o jsonpath='{.spec.template.spec.serviceAccountName}'
kubectl auth can-i --list --as=system:serviceaccount:<ns>:<sa>
kubectl get networkpolicy -n <namespace>

# 6. Deploy safety
kubectl rollout history deployment/<name>
```

## Hands-on Lab

Run the full audit script above against your Chapter 8 `whoami`
Deployment as it stands right now, and honestly grade it against all six
failure-mode categories — most will fail, since this course built each
mechanism in isolation, one chapter at a time, without ever combining all
of them onto one workload. Then, deliberately, bring it up to a genuine
production-readiness bar:

```bash
# 1 & 2: replicas ≥ 2, proper probes (Ch. 8, 17) — already partly done in Ch. 8's lab
# 3: real resources.requests/limits (Ch. 22) + an HPA (Ch. 23)
# 4: a ServiceMonitor (Ch. 27) if you still have the monitoring stack installed
# 5: a dedicated ServiceAccount with a scoped Role (Ch. 24), plus a default-deny
#    NetworkPolicy with a narrow allow rule (Ch. 25)
# 6: confirm `kubectl rollout history` shows real revisions, and that you can
#    state the exact rollback command without looking it up
```
Re-run the audit script afterward and confirm every check now passes —
this is the actual deliverable of the chapter: one workload, audited and
hardened end-to-end, not just individually-taught mechanisms.

## Troubleshooting — the meta-skill

- **"It works in Kind/staging but breaks in production"** — almost always
  traces to a difference in one of the six categories above that wasn't
  exercised in the smaller environment: real multi-node spread, real
  traffic volume hitting under-provisioned `requests`, or RBAC/
  NetworkPolicy restrictions that don't exist in a permissive local
  cluster.
- **An incident review reveals "we didn't know X was even happening"** —
  that's specifically a category-4 (observability) gap; the fix is
  always a new metric/log/alert, never just "be more careful" as a
  process fix.
- **A postmortem reveals a security exposure that "shouldn't have been
  possible"** — walk it back through categories 5 and 6 specifically:
  was RBAC actually scoped, was the NetworkPolicy actually enforced by
  the CNI (Ch. 25's "silent no-op" trap), was the image tag genuinely
  immutable and traceable.

## Best Practices — the meta-list

- Treat this six-category audit as a genuine pre-production checklist for
  every new workload, not just a one-time read — revisit it whenever a
  workload's traffic pattern, team ownership, or criticality changes.
- Resist the urge to over-invest in one category (elaborate autoscaling
  tuning) while neglecting another (no readiness probe at all) — the
  layered diagram above is deliberately ordered: lower layers being weak
  undermines every layer built on top.
- Practice failure, don't just prepare for it on paper — deliberately
  kill a Pod, cordon a node, revoke a NetworkPolicy's assumption, and
  confirm your system (and your team) responds the way you expect,
  before an actual incident forces you to find out for the first time.

## Interview Questions

1. **How would you evaluate whether a Kubernetes workload is
   "production-ready"?** Walk through failure-mode categories
   systematically (Pod loss, node loss, traffic spikes, observability,
   security, bad deploys) rather than a feature checklist — for each, ask
   whether the relevant mechanism (probes, HPA, RBAC, rollout gating,
   etc.) is actually configured deliberately, not left at defaults by
   accident.
2. **What's the single most common gap you'd expect to find auditing a
   real cluster?** Answers vary, but defensible ones include: missing or
   conflated liveness/readiness probes (Ch. 17), `resources` left unset
   (Ch. 22, defaulting to `BestEffort` QoS), or overly broad RBAC/
   NetworkPolicy gaps (Ch. 24-25) — all common because they're invisible
   until something goes wrong.
3. **Why is "it works in staging" not sufficient evidence of
   production-readiness?** Staging often differs precisely in the
   dimensions that matter most under real failure: traffic volume (does
   HPA/resources actually hold up), multi-node scale (does the workload
   actually survive losing a node), and security posture (RBAC/
   NetworkPolicy often looser in non-prod).
4. **What's the relationship between the six failure-mode categories in
   this chapter and the individual chapters earlier in the course?**
   Each category is a lens grouping several earlier mechanisms by the
   failure they defend against, rather than by object type — the goal is
   recognizing that "production readiness" is disciplined, complete use
   of tools you already learned, not new technology.

## Mini Assignment

Pick any workload you've built during this course (or the final project,
if you've started it) and produce a written production-readiness report
against all six categories: what's covered, what's missing, and — for
each gap — which specific earlier chapter's mechanism would close it.
This is the artifact a real team would produce before an on-call rotation
takes ownership of a service.

## Lesson Summary

- Production-readiness is best organized around failure modes (Pod loss,
  node loss, traffic spikes, observability gaps, security exposure, bad
  deploys), not a feature checklist.
- Nearly every production best practice is a mechanism from an earlier
  chapter, used deliberately and completely, rather than a new tool.
- The layered stack matters: weak Pod-level resilience undermines
  autoscaling, which undermines observability's usefulness, and so on —
  audit from the bottom layer up.
- The real test isn't reading this checklist once — it's practicing
  failure deliberately and confirming the system (and your team) behaves
  as expected before a real incident does it for you.

---

### Before Chapter 31 (Managed Kubernetes) — tell me:

1. Pick one of the six failure-mode categories and name, from memory,
   which two or three earlier chapters' mechanisms it draws on.
2. Why is "weak Pod-level resilience undermines autoscaling" true,
   specifically — walk through the causal chain.
3. From the Mini Assignment — what was the single biggest gap you found
   in your own workload, and which chapter's mechanism closes it?
