# Chapter 18 — Jobs

## What it is

A **Job** is a controller that runs one or more Pods **to completion** —
unlike every controller you've met so far (ReplicaSet, Deployment), whose
entire purpose is keeping Pods running *indefinitely*. A Job's Pods are
expected to **exit successfully** and stay stopped; the Job's job is to
make sure that happens, retrying on failure, up to a limit.

## Why it exists

Everything since Chapter 7 has assumed a long-running server process —
that's the majority of what you deploy, but not all of it. Database
migrations, batch data processing, one-off report generation, sending a
bulk email campaign — these are fundamentally **finite** tasks. Running
them as a Deployment would be actively wrong: the moment the task's
container process exits (successfully!), a Deployment's ReplicaSet would
see "actual replicas dropped below desired" and restart it — an infinite
loop of a task that already succeeded. Kubernetes needed a genuinely
different controller whose reconciliation target is "N successful
completions," not "N always-running replicas."

## When to use it

Any run-once-to-completion workload: DB migrations run as part of a
deploy, data backfills, report generation, sending notifications, CI
build/test steps run inside the cluster. If the same task needs to run on
a schedule repeatedly, you want a **CronJob** (Chapter 19), which is
built directly on top of Job.

## Internal architecture

- A Job creates Pods with `restartPolicy: Never` or `OnFailure` (never
  `Always`, which is only valid for long-running controllers) — this is
  enforced by the API server, a direct signal that this Pod is expected
  to terminate.
- `spec.completions` — how many **successful** Pod completions are needed
  total before the Job itself is considered `Complete`.
- `spec.parallelism` — how many Pods may run **concurrently** while
  working toward that completion count — e.g., `completions: 10,
  parallelism: 3` processes 10 units of work, 3 at a time.
- `spec.backoffLimit` — how many Pod **failures** are tolerated (with
  exponential backoff between retries) before the Job gives up entirely
  and marks itself `Failed`.
- On Pod failure (non-zero exit, or `OnFailure` restartPolicy triggering a
  container-level retry), the Job controller creates a **replacement
  Pod** — same reconciliation-loop shape as everything else in this
  course, just counting "successes needed" instead of "replicas needed."
- `spec.template.spec.restartPolicy: OnFailure` restarts the *same*
  container in place on failure (kubelet-level, cheap); `Never` instead
  lets the *Job controller* create a brand-new replacement Pod on failure
  — different retry granularity, worth choosing deliberately.

## YAML Definition

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: data-migration
spec:
  completions: 5
  parallelism: 2
  backoffLimit: 3
  activeDeadlineSeconds: 300
  template:
    spec:
      restartPolicy: OnFailure
      containers:
        - name: migrator
          image: busybox
          command: ["sh", "-c", "echo migrating batch $RANDOM; sleep 5; exit 0"]
```
- `apiVersion: batch/v1` — Jobs (and CronJobs, Chapter 19) live in the
  `batch` API group.
- `spec.completions: 5` / `spec.parallelism: 2` — run 5 total successful
  Pods, at most 2 at once (so this Job takes roughly `ceil(5/2) = 3`
  rounds of ~2 Pods each).
- `spec.backoffLimit: 3` — give up after 3 total Pod failures across the
  whole Job.
- `spec.activeDeadlineSeconds` — a hard wall-clock cap on the Job's total
  runtime, regardless of retries remaining — a safety net against a Job
  that's technically still "retrying" but effectively hung forever.
- `spec.template.spec.restartPolicy: OnFailure` — required field for Job
  Pods; distinguishes retry-in-place from replacement-Pod semantics as
  above.

## Hands-on Example

```bash
kubectl apply -f data-migration-job.yaml
kubectl get job data-migration -w
```
Watch `COMPLETIONS` climb (e.g., `0/5` → `2/5` → ... → `5/5`), and
`kubectl get pods -l job-name=data-migration` show Pods appearing in
batches of 2 (your `parallelism`), each transitioning to `Completed`
(not `Running` forever — the key visual difference from every controller
so far).

```bash
kubectl logs -l job-name=data-migration --tail=-1
```
Aggregate logs across every Pod the Job created — useful since a Job can
span many Pods over its lifetime.

**Force a failure and watch retry behavior:**
```yaml
# same Job, but the container: command: ["sh", "-c", "exit 1"]
```
```bash
kubectl apply -f failing-job.yaml
kubectl get pods -l job-name=failing-job -w
```
Watch failed Pods accumulate (each one stays visible, marked `Error`, not
deleted automatically) up to `backoffLimit`, then the Job itself flips to
`Failed`:
```bash
kubectl get job failing-job -o jsonpath='{.status.conditions}'
```

**Prove Jobs and Deployments genuinely differ in intent** — try the same
"exit 0 immediately" container as a Deployment instead of a Job:
```bash
kubectl create deployment bad-idea --image=busybox -- sh -c "exit 0"
kubectl get pods -l app=bad-idea -w
```
Watch it `CrashLoopBackOff` forever — the ReplicaSet controller keeps
trying to maintain a "running" replica out of a container that, by
design, immediately exits successfully. This is exactly the wrong-tool
scenario Jobs exist to avoid.

Cleanup:
```bash
kubectl delete -f data-migration-job.yaml -f failing-job.yaml
kubectl delete deployment bad-idea
```

## Debugging Techniques

- **Job stuck, `COMPLETIONS` not increasing** — check `parallelism` isn't
  0 by mistake, and check individual Pods (`kubectl get pods -l
  job-name=<name>`) for `Pending` (scheduling issue, Chapter 2) or
  `CrashLoopBackOff` (application issue, Chapter 5) causes.
- **Job shows `Failed`** — `kubectl describe job <name>` shows the
  `BackoffLimitExceeded` condition; check the actual failed Pods' logs
  (`kubectl logs <pod> --previous` if needed) for the real root cause —
  the Job object itself only reports that retries were exhausted, not
  why each attempt failed.
- **Old completed/failed Pods piling up** — Jobs, unlike Deployments,
  don't clean up their Pods automatically by default; use
  `spec.ttlSecondsAfterFinished` to auto-delete a Job (and its Pods) some
  time after completion, or clean up manually — a very common
  "why is my cluster full of old Completed pods" surprise for people new
  to Jobs specifically.
- **`activeDeadlineSeconds` hit** — Job is marked `Failed` with reason
  `DeadlineExceeded`, even if `backoffLimit` hadn't been reached yet —
  don't confuse this with a normal backoff-exhaustion failure when
  reading incident logs.

## Best Practices

- Always set `backoffLimit` and (for anything with a reasonable expected
  runtime) `activeDeadlineSeconds` — an unbounded Job retrying forever
  against a permanently broken dependency is a real, quiet
  resource-consumption incident.
- Set `ttlSecondsAfterFinished` so completed Jobs and their Pods
  self-clean — don't rely on manual cleanup in production.
- Make Job workloads **idempotent** wherever possible — since retries
  create entirely new Pods (with `restartPolicy: Never`), a task that
  partially completes then fails and retries should be safe to run again
  from scratch, not double-apply its effects.
- Choose `parallelism` based on genuine downstream capacity (database
  connection limits, external API rate limits) — it's tempting to set it
  high for speed, but that can overwhelm the very systems the Job is
  writing to.

## Interview Questions

1. **Why can't you just run a one-off batch task as a Deployment?**
   A Deployment's ReplicaSet expects Pods to stay running; a task that
   exits successfully (even correctly) looks like "replica count dropped"
   to a ReplicaSet controller, which restarts it forever — the wrong
   reconciliation target entirely.
2. **What do `completions` and `parallelism` control, and how do they
   interact?** `completions` is the total number of successful Pod
   completions required; `parallelism` caps how many run concurrently
   while working toward that total — e.g., 10 completions at parallelism
   3 runs in ceil(10/3) batches.
3. **What's the difference between `restartPolicy: OnFailure` and
   `Never` for a Job's Pods?** `OnFailure` restarts the same container in
   place on failure (kubelet-level); `Never` lets the failed Pod stay as
   -is while the Job controller creates a brand-new replacement Pod —
   different retry granularity and visibility into failure history.
4. **What happens when a Job exceeds its `backoffLimit`?**
   The Job stops retrying and is marked `Failed`, with a
   `BackoffLimitExceeded` condition — no more Pods are created for it.
5. **Why do completed Job Pods stick around instead of being cleaned up
   automatically?** By default Kubernetes keeps them for post-mortem
   inspection (logs, exit codes); use `ttlSecondsAfterFinished` to
   opt into automatic cleanup after a delay you choose.

## Mini Assignment

Create a Job with `completions: 6`, `parallelism: 2`, where the container
succeeds on even-numbered runs and fails on odd ones (hint: use
`$RANDOM` or a counter file on an `emptyDir` to simulate this, since each
Pod starts fresh and doesn't know its own attempt number automatically).
Set `backoffLimit` generously enough for the Job to eventually reach
`6/6` despite some failures, and document how many total Pods (successes
+ failures) were created by the end.

## Lesson Summary

- A Job runs Pods **to completion**, tracking successful completions
  rather than maintaining always-running replicas — the correct
  controller for finite, one-off, or batch workloads.
- `completions`/`parallelism` control total work and concurrency;
  `backoffLimit`/`activeDeadlineSeconds` bound how much retrying/time is
  tolerated before giving up.
- Running a naturally-finite task as a Deployment instead causes an
  infinite, wrong-tool `CrashLoopBackOff` — you proved this directly.
- Job Pods aren't cleaned up automatically by default —
  `ttlSecondsAfterFinished` is the standard fix.

---

### Before Chapter 19 (CronJobs) — tell me:

1. Why does running a "run once and exit 0" container as a Deployment
   result in a crash loop, specifically?
2. What's the practical difference between `restartPolicy: OnFailure`
   and `Never` for a failing Job Pod?
3. From the Mini Assignment — how many total Pods did your Job create to
   reach 6 successful completions, given the induced failures?
