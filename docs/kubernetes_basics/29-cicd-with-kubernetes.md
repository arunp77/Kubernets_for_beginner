# Chapter 29 — CI/CD with Kubernetes

## What it is

CI/CD with Kubernetes is the practice of automating the path from "code
merged" to "running, verified, in the cluster" — building an image
(Continuous Integration, something you already know) and then getting
that new image running in the right cluster/namespace, safely, without a
human running `kubectl apply` by hand (Continuous Deployment/Delivery).
This chapter is where every earlier chapter's mechanism gets wired
together into one automated pipeline.

## Why it exists

You already know CI: on every push, build the image, run tests, push to
a registry. Everything past that point — actually getting the new image
into the cluster — is where teams historically improvised inconsistently:
SSH-ing in and running `kubectl set image` by hand, a Jenkins job with a
raw `kubectl apply`, or worse. That's exactly the kind of manual,
human-in-the-loop toil this entire course has been eliminating, layer by
layer — CI/CD for Kubernetes closes the last gap: connecting "a new image
exists" to "the cluster's declared desired state (Chapter 6) now
references it," automatically, safely, and auditably.

## When to use it

Any team deploying to Kubernetes more than a handful of times — which is
essentially every real team. The specific pattern (push-based vs.
pull-based/GitOps) is the actual design decision worth understanding
deeply.

## Internal architecture — two fundamentally different models

- **Push-based (traditional CI/CD)**: your CI pipeline (GitHub Actions,
  GitLab CI, Jenkins) builds the image, pushes it to a registry, and then
  the *pipeline itself* runs `kubectl apply`/`helm upgrade` directly
  against the cluster, using credentials it holds. Simple to reason
  about, but means your CI system needs live, standing credentials with
  write access to your production cluster — a real security surface
  (if CI is compromised, so is your cluster).
- **Pull-based / GitOps (the increasingly standard production pattern)**:
  your CI pipeline builds the image and pushes it, **then updates a
  manifest in a git repository** (e.g., bumping an image tag in a
  Deployment YAML or a Helm `values.yaml`) — nothing touches the cluster
  directly. A separate **in-cluster agent** (ArgoCD or Flux are the two
  dominant tools) continuously watches that git repo and *pulls* changes,
  applying them to the cluster itself. This inverts the trust
  direction entirely: nothing external ever needs cluster-write
  credentials — the agent already living *inside* the cluster (with
  RBAC, Chapter 24, scoped to exactly what it needs) is the only thing
  that ever calls the API server to apply changes, and it does so by
  polling/watching git, the exact same "watch a source of truth and
  reconcile" controller pattern from Chapter 1/2 — now applied to your
  *entire deployment process*, not just individual objects.
- Either way, the actual mechanics that make a deploy *safe* are things
  you've already learned: readiness probes (Chapter 17) gating a rolling
  update (Chapter 8), `kubectl rollout status`/`helm history` (Chapters
  8, 26) for automated rollback triggers, and `--dry-run`/`kubectl diff`
  (Chapters 4, 6) for pre-apply validation steps in the pipeline itself.
- **Image tagging strategy matters mechanically here**: never deploy
  `:latest` — a pipeline needs a *specific, immutable* tag (commonly the
  git commit SHA) so "what's actually running" is always traceable back
  to an exact commit, and so a rollback (Chapter 8's `rollout undo`) is
  reverting to a genuinely previous, known-good image, not an ambiguous
  moving target.

## YAML/Config Walkthrough

**Push-based, via GitHub Actions:**
```yaml
# .github/workflows/deploy.yml
name: build-and-deploy
on:
  push:
    branches: [main]
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Build and push image
        run: |
          docker build -t myregistry/whoami:${{ github.sha }} .
          docker push myregistry/whoami:${{ github.sha }}
      - name: Configure kubeconfig
        run: echo "${{ secrets.KUBE_CONFIG }}" | base64 -d > kubeconfig
      - name: Deploy
        run: |
          kubectl --kubeconfig=kubeconfig set image deployment/whoami \
            whoami=myregistry/whoami:${{ github.sha }}
          kubectl --kubeconfig=kubeconfig rollout status deployment/whoami --timeout=120s
```
- `docker build ... ${{ github.sha }}` — tags with the exact commit SHA,
  never `latest` — precisely the immutable-tagging point above.
- `secrets.KUBE_CONFIG` — a Kubernetes Secret-adjacent concept, but note:
  this is a **CI system secret** (GitHub Actions' own secret store,
  distinct from a Kubernetes Secret, Chapter 13), holding live
  cluster-write credentials — this line *is* the push-based model's
  security tradeoff, made concrete.
- `kubectl rollout status ... --timeout=120s` — the pipeline fails loudly
  if the rollout doesn't succeed within the timeout, exactly per Chapter
  8's readiness-probe-gated rollout mechanics — a broken deploy blocks
  the pipeline rather than silently leaving a half-rolled-out state.

**Pull-based / GitOps, via ArgoCD:**
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: whoami
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/you/whoami-manifests.git
    targetRevision: main
    path: k8s/whoami
  destination:
    server: https://kubernetes.default.svc
    namespace: default
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```
- `spec.source` — the git repo/path ArgoCD watches; your CI pipeline's
  *only* job in this model is to update the image tag in a file at this
  path and push — it never touches the cluster.
- `spec.syncPolicy.automated.selfHeal: true` — this is the controller
  pattern from Chapter 1 made explicit and literal: if someone manually
  `kubectl edit`s a live object away from what's in git, ArgoCD's
  reconciliation loop notices the drift and reverts it — git, not
  whatever's live in the cluster, is the actual source of truth.
- `prune: true` — objects removed from git are actually deleted from the
  cluster too, keeping the cluster's state a true mirror of the repo, not
  just an additive one.

## Hands-on Example

**Push-based, end-to-end, using tools already installed this course:**
```bash
# simulate what a pipeline step does, by hand, to see the exact mechanics:
docker build -t localhost:5000/whoami:$(git rev-parse --short HEAD) .
# (a local registry setup is a real prerequisite for pushing to Kind — for this lab,
#  reuse the public traefik/whoami image and just SIMULATE a new "version" via a label change)

kubectl set image deployment/whoami whoami=traefik/whoami:v1.10.4
kubectl rollout status deployment/whoami --timeout=60s
echo "Exit code: $?"    # a real pipeline gates on this exact exit code
```

**Simulate a bad deploy and prove the pipeline would catch it:**
```bash
kubectl set image deployment/whoami whoami=traefik/whoami:this-tag-does-not-exist
kubectl rollout status deployment/whoami --timeout=30s
echo "Exit code: $?"    # non-zero — a real CI job would fail HERE, before declaring success
kubectl rollout undo deployment/whoami
```
This is the entire safety mechanism of Chapter 8, now framed explicitly
as a CI/CD gate: `rollout status`'s exit code *is* your pipeline's
pass/fail signal.

**Install ArgoCD and see GitOps reconciliation directly:**
```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl wait --for=condition=available deployment/argocd-server -n argocd --timeout=180s
kubectl port-forward svc/argocd-server -n argocd 8080:443
```
Point an `Application` object (as above) at a real public git repo of
plain Kubernetes YAML, and watch ArgoCD's UI show it sync automatically.

**Prove `selfHeal` directly** — the most convincing demonstration in this
whole chapter:
```bash
kubectl scale deployment whoami --replicas=10    # manual, out-of-band change
```
Watch ArgoCD's UI flag the Application as `OutOfSync`, then
automatically revert `replicas` back to whatever git actually says —
proving, hands-on, that git (not the live cluster) is the real source of
truth under GitOps.

Cleanup:
```bash
kubectl delete namespace argocd
```

## Debugging Techniques

- **Pipeline reports "success" but the app is visibly broken** — check
  whether the pipeline actually gates on `kubectl rollout status`'s exit
  code (or Helm/ArgoCD's equivalent health check) rather than just
  `kubectl apply`'s exit code, which succeeds the instant the API server
  *accepts* the object, regardless of whether it ever actually becomes
  healthy.
- **GitOps agent shows `OutOfSync` and won't resolve** — check for a
  genuine manifest error (invalid YAML, RBAC denial on the agent's own
  ServiceAccount, Chapter 24) versus expected, intentional drift being
  actively corrected by `selfHeal`.
- **Deploy "worked" locally/in staging, fails in prod pipeline
  specifically** — check registry authentication (`imagePullSecrets`,
  Chapter 13) and RBAC scope (Chapter 24) for whichever identity/
  ServiceAccount the pipeline or GitOps agent uses in that specific
  environment — environment-specific credentials are a very common
  divergence point.
- **Rollback via CI doesn't actually fix the issue** — confirm the
  pipeline rolled back the *image tag* specifically, not just re-ran the
  same broken deploy — this is why immutable, commit-SHA tagging matters
  mechanically, not just as hygiene.

## Best Practices

- Never deploy `:latest` — always an immutable, traceable tag (git commit
  SHA is the standard choice) so any running Pod's image can be traced
  back to an exact commit, and rollback means something concrete.
- Prefer GitOps (pull-based) for anything production-grade — it removes
  standing cluster-write credentials from your CI system entirely, a
  meaningfully smaller attack surface, and gives you git's own history/
  review process as your deployment audit log for free.
- Gate every pipeline step on the actual health signal (`rollout status`,
  Helm's `--wait`, ArgoCD's sync/health status) — never just on whether
  `kubectl apply`/`helm upgrade` was *accepted*, which says nothing about
  whether it actually became healthy.
- Keep environment-specific config (Chapter 26's per-environment
  `values.yaml` files) in git alongside the application manifests, so the
  GitOps agent's "source of truth" genuinely captures the full desired
  state for each environment, not just the base template.

## Interview Questions

1. **What's the fundamental difference between push-based CI/CD and
   GitOps?** Push-based has the CI pipeline itself apply changes to the
   cluster using credentials it holds; GitOps has an in-cluster agent
   pull/watch a git repo and apply changes itself — inverting which side
   needs cluster-write credentials, and making git the actual source of
   truth reconciled against continuously, not just an initial trigger.
2. **Why is `selfHeal` in a GitOps tool like ArgoCD significant?**
   It proves git, not the live cluster, is the actual desired state —
   any manual out-of-band change to the cluster gets automatically
   reverted, which is the controller/reconciliation-loop pattern
   (Chapter 1/2) applied to your entire deployment process.
3. **Why should you never deploy using the `:latest` image tag?**
   `:latest` is not a stable reference to a specific build — you lose the
   ability to know exactly what's running or to roll back to a
   genuinely previous, known-good version; an immutable tag (commit SHA)
   keeps deployments traceable and rollbacks meaningful.
4. **What should a CI/CD pipeline actually gate its success on, and
   why?** The rollout's actual health signal (`kubectl rollout status`
   exit code, Helm's `--wait`, or the GitOps tool's health status) — not
   merely whether the API server accepted the manifest, which happens
   instantly regardless of whether the new Pods ever become healthy.
5. **What's a concrete security advantage of GitOps over traditional
   push-based deployment?** No external system (CI pipeline) needs
   standing write credentials to the cluster — only an in-cluster agent,
   scoped via RBAC (Chapter 24) to exactly what it needs, ever calls the
   API server to make changes.

## Mini Assignment

Set up the ArgoCD lab from the hands-on section pointed at a small public
git repo of your own containing a Deployment + Service (reuse
`whoami-deploy.yaml`/`whoami-svc.yaml`). Confirm automatic sync on a git
push (change the replica count in the repo, push, watch ArgoCD apply it
within its polling interval), then deliberately break the Mini
Assignment's own safety net: push a manifest with an invalid `apiVersion`
(Chapter 6) and observe exactly how ArgoCD reports the sync failure,
without ever silently applying a broken state.

## Lesson Summary

- CI/CD for Kubernetes connects "a new image exists" to "the cluster's
  desired state now reflects it" — either by the pipeline pushing changes
  directly, or (increasingly standard) an in-cluster GitOps agent pulling
  changes from git.
- GitOps inverts the trust model: no external system needs cluster-write
  credentials, and `selfHeal` proves git — not the live cluster — is the
  actual source of truth, the controller pattern applied to your whole
  deployment process.
- Every safety mechanism a pipeline relies on (readiness-gated rollouts,
  rollback, dry-run validation) is a mechanism you already learned in
  earlier chapters, now wired into automation rather than run by hand.
- Immutable image tagging (commit SHA, never `:latest`) is what makes
  "what's running" traceable and rollback meaningful.

---

### Before Chapter 30 (Production Best Practices) — tell me:

1. In GitOps, which side (CI pipeline or in-cluster agent) actually calls
   the Kubernetes API to apply changes, and why does that matter for
   security?
2. Why is gating a pipeline on `kubectl apply`'s exit code insufficient,
   specifically?
3. From the Mini Assignment — how did ArgoCD report the deliberately
   broken manifest, and did it apply any part of it before failing?
