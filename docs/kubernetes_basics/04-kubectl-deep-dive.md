# Chapter 4 — kubectl Deep Dive

## Lesson Objectives

By the end of this chapter you will:

- Understand `kubectl` as nothing more than an HTTP client for the API
  server — demystifying every command that follows.
- Be fluent in the verbs you'll use dozens of times a day:
  `get`, `describe`, `logs`, `exec`, `apply`, `delete`, `edit`.
- Know the imperative-vs-declarative split within `kubectl` itself, and
  which one production teams actually use.
- Be able to debug "why isn't kubectl working" independent of anything
  Kubernetes-object-specific.

---

## Theory

You've used `docker` as a CLI that both *builds instructions* and *talks
to a daemon*. `kubectl` only does the second half — it never builds
anything. Every `kubectl` command does exactly one thing: serialize your
intent into an HTTP request, send it to the API server (Chapter 2), and
render the JSON response back as readable output. That's it. There's no
local state, no daemon on your machine — `kubectl` is closer in spirit to
`curl` with very good manners than to `dockerd`'s CLI frontend.

This matters practically: if `kubectl` misbehaves, the bug is in one of
exactly three places — (1) your local kubeconfig pointing at the wrong
cluster/context, (2) network/auth between you and the API server, or (3)
the object/state you asked about. There is no fourth category ("kubectl's
own internal state got corrupted") the way there sometimes is with
stateful CLIs.

### Imperative vs. declarative kubectl

- **Imperative commands** — `kubectl run`, `kubectl create deployment`,
  `kubectl scale` — issue a one-time instruction, similar in spirit to
  `docker run`. Great for quick experiments (as you did in Chapter 3),
  bad for anything you want to track in git or reproduce reliably.
- **Declarative `kubectl apply -f file.yaml`** — reads a YAML manifest
  describing desired state and reconciles the live object to match it,
  computing a diff against what's already there (a three-way merge
  against the last-applied config, actually stored as an annotation on the
  object). This is what production teams use almost exclusively — the
  YAML file is checked into git, and `kubectl apply` (usually via CI/CD,
  Chapter 29) is the *only* way changes reach the cluster. This is what
  "GitOps" is built on.

---

## Architecture Diagram

```
   your terminal
   ┌─────────────┐        HTTPS (auth via kubeconfig:
   │   kubectl     │ ─────▶  token/cert + cluster URL)
   └─────────────┘                    │
                                       ▼
                            ┌───────────────────┐
                            │   kube-apiserver     │
                            └───────────────────┘
                                       │
                                       ▼
                                     etcd

  ~/.kube/config tells kubectl WHICH cluster (context), WHO you are
  (user/cert/token), and WHICH namespace to default to.
```

---

## Docker Comparison

| Concern | `docker` CLI | `kubectl` |
|---|---|---|
| Talks to | Local Docker Engine daemon (usually a Unix socket) | Remote API server (HTTPS, possibly hundreds of miles away) |
| Builds images | Yes (`docker build`) | No — never |
| State awareness | Daemon has full local state | Stateless client; every command is a fresh API call |
| Config location | Docker context (`~/.docker/`) | Kubeconfig (`~/.kube/config`), supports many clusters/contexts at once |
| Primary interaction style teams use | Mostly imperative (`docker run`) | Declarative (`kubectl apply -f`), usually via CI/CD not by hand |

---

## Internal Working

- `~/.kube/config` (the **kubeconfig**) holds three linked lists:
  **clusters** (API server URL + CA cert), **users** (how you
  authenticate — client cert, token, or exec plugin like `aws eks
  get-token`), and **contexts** (a named pairing of one cluster + one user
  + a default namespace). `kubectl config use-context X` just changes
  which pairing is "current" — nothing on the cluster changes.
- `kubectl apply` computes a **three-way merge**: it compares (a) the
  manifest you're applying, (b) the object's live state, and (c) the
  *last applied* manifest (stored verbatim in the
  `kubectl.kubernetes.io/last-applied-configuration` annotation on the
  object). This three-way diff is precisely how `apply` can tell "you
  removed this field on purpose" apart from "someone else changed this
  field out-of-band, leave it alone" — a plain two-way diff couldn't
  distinguish those.
- `kubectl get -o wide` / `-o yaml` / `-o json` aren't different data —
  they're different serializations of the exact same object the API
  server returned; useful to remember when you want to pipe output into
  `jq` or diff two objects precisely.

---

## Hands-on Lab

Using the `masterclass` cluster from Chapter 3:

**1. Inspect your kubeconfig:**
```bash
kubectl config view
kubectl config get-contexts
kubectl config current-context
```
`view` prints the full merged config (with secrets redacted by default);
`get-contexts` lists every cluster/user pairing available; `current-context`
shows which one commands run against right now.

**2. Core read commands — do these against `kube-system`, which already
has real objects (the control-plane components from Chapter 2), so
there's something to look at immediately:**
```bash
kubectl get pods -n kube-system
kubectl get pods -n kube-system -o wide
kubectl describe pod <a-pod-name> -n kube-system
```
`get` lists objects, one line each. `-n` selects the namespace (default
namespace is used if omitted — Chapter 11 covers this properly). `-o wide`
adds columns (node, IP) without changing what's fetched — same API call,
richer rendering. `describe` fetches the full object *plus* recent
**Events** related to it — this events list is where 90% of debugging
information actually lives; get comfortable reading it now.

**3. Logs and exec — your `docker logs`/`docker exec` equivalents:**
```bash
kubectl logs <pod-name> -n kube-system
kubectl logs <pod-name> -n kube-system -f       # follow, like docker logs -f
kubectl exec -it <pod-name> -n kube-system -- sh
```
Identical mental model to Docker: `logs` streams the container's stdout/
stderr as captured by the container runtime; `exec -it` opens an
interactive shell inside the running container. The only new piece:
since a Pod can hold more than one container (Chapter 5), you'll
sometimes need `-c <container-name>` to disambiguate which one.

**4. Imperative vs. declarative, side by side:**
```bash
# imperative — quick, not reproducible, not tracked
kubectl create deployment demo --image=nginx

# capture the SAME desired state as YAML instead
kubectl get deployment demo -o yaml > demo.yaml
kubectl delete deployment demo

# now do it the way production actually works
kubectl apply -f demo.yaml
```
`kubectl get -o yaml > file` is a genuinely useful trick: let an
imperative command scaffold a starting YAML, then throw the imperative
object away and manage the same thing declaratively from then on.

**5. Edit and diff:**
```bash
kubectl edit deployment demo          # opens live object in $EDITOR, applies on save
kubectl diff -f demo.yaml             # shows what apply WOULD change, without changing it
```
`edit` is fine for emergency live tweaks; `diff` is what you should run
habitually before `apply` in anything resembling production, to see the
blast radius of a change before committing to it.

**Cleanup:**
```bash
kubectl delete -f demo.yaml
```

---

## YAML Walkthrough

Not applicable — Chapter 6 is dedicated to YAML structure itself. This
chapter used YAML only as a pass-through payload for `apply`/`get -o
yaml`, without explaining its fields yet.

---

## Troubleshooting

- **`The connection to the server ... was refused`** — kubeconfig points
  at a cluster that's down or the wrong context is selected; check
  `kubectl config current-context` and `kubectl cluster-info` first,
  always, before assuming anything about the object you're working with.
- **`Error from server (Forbidden)`** — you reached the API server fine,
  but RBAC (Chapter 24) denies you this action; not a connectivity
  problem, an authorization one — different fix entirely.
- **`No resources found`** — often just means you forgot `-n <namespace>`
  and the object lives elsewhere, not that it doesn't exist. Try `kubectl
  get <type> -A` (all namespaces) before concluding it's missing.
- **`kubectl apply` says "field is immutable"** — some fields (like a
  Pod's `spec.containers[].image` isn't immutable, but a Service's
  `clusterIP` is, for example) can't be changed on an existing object;
  you'll need to delete and recreate, which `apply` deliberately won't do
  for you silently.

---

## Best Practices

- Default to `kubectl apply -f`, never `kubectl create`/`kubectl run`,
  for anything you intend to keep — imperative commands leave no
  git-trackable record and don't merge cleanly with later changes.
- Always run `kubectl diff -f` before `apply` in any shared/production
  context — never apply changes you haven't previewed.
- Learn `kubectl explain <resource>.<field>` early — e.g. `kubectl explain
  pod.spec.containers` prints the *built-in, authoritative* field
  documentation straight from the API server's schema. It's faster and
  more trustworthy than searching docs, and it's how you'll answer "what
  fields exist here" for the rest of the course.
- Set up shell autocompletion (`kubectl completion bash`) immediately —
  Kubernetes object/field names are long, and typo-driven errors are the
  single most common source of early frustration.

---

## Interview Questions

1. **What does `kubectl` actually do when you run a command?**
   Serializes the command into an HTTPS request to the API server per
   your kubeconfig's current context, and renders the JSON response —
   nothing more; it holds no cluster state itself.
2. **What's the difference between `kubectl apply` and `kubectl create`?**
   `create` issues a one-time imperative creation and errors if the object
   already exists. `apply` is declarative: it computes a three-way diff
   (desired manifest vs. live state vs. last-applied-config) and
   reconciles, and is safe to re-run repeatedly (idempotent).
3. **How does `kubectl apply` know which fields you removed on purpose
   versus fields someone else changed out-of-band?**
   It compares against the `last-applied-configuration` annotation stored
   from the previous `apply`, not just the current live object — a
   genuine three-way merge, not a two-way diff.
4. **What's in a kubeconfig file?**
   Clusters (API server endpoint + CA cert), users (auth credentials or a
   plugin to fetch them), and contexts (named cluster+user+namespace
   combinations) — plus which context is "current."
5. **How would you debug a `kubectl` command that hangs?**
   Check `kubectl cluster-info` / `kubectl config current-context` first
   to rule out connectivity/context issues before assuming the problem is
   with the object or command itself.

---

## Mini Assignment

Create an nginx Deployment imperatively with `kubectl create deployment`,
then use `kubectl get -o yaml` to capture it, delete the imperative
version, hand-clean the captured YAML (remove server-generated fields like
`status`, `uid`, `resourceVersion`, `creationTimestamp` — you'll learn
exactly which fields matter in Chapter 6), and `apply` the cleaned file.
Then change the replica count in the file and run `kubectl diff -f` before
`kubectl apply -f` — confirm the diff shows exactly what you expect before
it's applied.

---

## Lesson Summary

- `kubectl` is a stateless HTTP client for the API server — every bug
  traces back to kubeconfig/context, network/auth, or the object/state
  itself, never to hidden local client state.
- Imperative commands (`create`, `run`, `scale`) are fine for quick
  exploration; declarative `apply -f` (three-way merge against
  last-applied-config) is what real teams use for anything durable.
- `describe`, `logs`, and `exec` map almost one-to-one onto Docker
  equivalents you already know.
- `kubectl explain` and `kubectl diff` are two habits worth building now
  that will pay off for the rest of the course.

---

### Before Lesson 5 (Pods) — tell me:

1. In your own words, what three categories does every `kubectl` failure
   fall into?
2. Why can't a simple two-way diff safely implement what `kubectl apply`
   does?
3. What did `kubectl diff` show you in the Mini Assignment before you
   applied the replica-count change?
