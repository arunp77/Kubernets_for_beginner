# Chapter 19 — CronJobs

## What it is

A **CronJob** creates a **Job** (Chapter 18) on a recurring schedule,
specified with standard cron syntax. It's a thin controller layered
directly on top of Job — the same relationship Deployment has to
ReplicaSet (Chapter 8 built on Chapter 7).

## Why it exists

Chapter 18 covered one-off finite tasks; plenty of real tasks are finite
*and recurring* — nightly database backups, hourly report generation, a
daily cache-warming job. You know the Linux equivalent, `cron` — the same
idea, cluster-native: instead of a crontab entry on one specific machine
(back to the single-host assumption Chapter 1 opened with), Kubernetes'
CronJob controller creates a fresh Job (which creates fresh Pods) on
schedule, on whatever node the scheduler picks, with the exact same
retry/parallelism machinery Job already provides.

## When to use it

Anything you'd otherwise put in a Linux crontab, but that should run
inside your cluster with access to cluster resources (ConfigMaps,
Secrets, internal Services) rather than on some specific host you have to
maintain separately.

## Internal architecture

- The CronJob controller watches the wall clock (compared against
  `spec.schedule`) and, at each scheduled tick, creates a new **Job**
  object from `spec.jobTemplate` — that Job then behaves exactly as
  Chapter 18 described, completely independently of the CronJob once
  created.
- **`concurrencyPolicy`** governs what happens if a previous run's Job
  hasn't finished by the time the next scheduled tick arrives:
  `Allow` (default — let them run concurrently), `Forbid` (skip the new
  tick entirely if the previous Job is still running), `Replace` (kill
  the still-running Job and start the new one). This matters enormously
  for tasks that aren't safe to run concurrently with themselves (e.g., a
  backup job that would corrupt output if two instances wrote to the same
  location at once).
- **`startingDeadlineSeconds`** — if the CronJob controller itself was
  down (cluster maintenance, control-plane issue) and misses a scheduled
  tick, this caps how late a missed run is allowed to start before being
  skipped entirely, rather than firing a large backlog of overdue runs
  all at once when the controller comes back.
- `successfulJobsHistoryLimit` / `failedJobsHistoryLimit` — how many
  completed/failed Job objects (and their Pods) are kept around for
  inspection before being garbage collected — the CronJob-level version of
  Chapter 18's `ttlSecondsAfterFinished` cleanup concern.

## YAML Definition

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: nightly-backup
spec:
  schedule: "0 2 * * *"
  concurrencyPolicy: Forbid
  startingDeadlineSeconds: 300
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 3
  jobTemplate:
    spec:
      backoffLimit: 2
      activeDeadlineSeconds: 1800
      template:
        spec:
          restartPolicy: OnFailure
          containers:
            - name: backup
              image: busybox
              command: ["sh", "-c", "echo backing up at $(date); sleep 5"]
```
- `apiVersion: batch/v1` — same API group as Job (Chapter 18).
- `spec.schedule: "0 2 * * *"` — standard 5-field cron syntax
  (minute hour day-of-month month day-of-week) — `0 2 * * *` means
  "02:00 every day," in the **cluster's configured time zone** (worth
  double-checking explicitly — a common source of "ran at the wrong
  time" surprises across UTC vs. local-time assumptions).
- `spec.concurrencyPolicy: Forbid` — appropriate here since two
  concurrent backup runs writing to the same destination would be
  actively harmful, not just wasteful.
- `spec.jobTemplate.spec` — literally a full Job spec (everything from
  Chapter 18: `backoffLimit`, `activeDeadlineSeconds`, the Pod
  `template`), nested one level deeper — because a CronJob's entire
  purpose is stamping out Job objects shaped exactly like this.

## Hands-on Example

For a fast feedback loop instead of waiting for `2am`, use a schedule
that fires every minute during the lab:

```yaml
# same as above but: schedule: "*/1 * * * *"
```
```bash
kubectl apply -f nightly-backup-cronjob.yaml
kubectl get cronjob nightly-backup
```
`LAST SCHEDULE` starts empty, then updates once the first tick fires.
Wait a couple of minutes:
```bash
kubectl get jobs -l app.kubernetes.io/created-by=cronjob-controller 2>/dev/null || kubectl get jobs
kubectl get pods
```
Watch a new Job (and its Pod) appear roughly once a minute, each named
`nightly-backup-<timestamp-hash>` — a fresh Job object per scheduled run,
each behaving exactly per Chapter 18.

**Prove `concurrencyPolicy: Forbid` actually prevents overlap** — patch
the container to sleep longer than the schedule interval, so a run would
still be in progress when the next tick fires:
```bash
kubectl patch cronjob nightly-backup --type='json' \
  -p='[{"op":"replace","path":"/spec/jobTemplate/spec/template/spec/containers/0/command","value":["sh","-c","sleep 90"]}]'
kubectl get jobs -w
```
Watch: no second Job starts while the first is still running — confirm
by checking `kubectl get events | grep -i forbid` for a
`JobAlreadyActive`-type skip message.

**Compare against `Allow`:**
```bash
kubectl patch cronjob nightly-backup -p '{"spec":{"concurrencyPolicy":"Allow"}}'
```
Now watch overlapping Jobs genuinely run side by side.

**Manually trigger an ad-hoc run** (useful for testing/backfills without
waiting for the schedule):
```bash
kubectl create job manual-backup-test --from=cronjob/nightly-backup
kubectl get pods -l job-name=manual-backup-test
```

Cleanup:
```bash
kubectl delete cronjob nightly-backup
kubectl delete job manual-backup-test
```

## Debugging Techniques

- **CronJob never fires (`LAST SCHEDULE` stays empty)** — check the
  cron expression itself first (an off-by-one in cron syntax is
  extremely common — verify with any cron-expression checker before
  assuming Kubernetes is broken), then check the CronJob controller
  itself is healthy (`kubectl get pods -n kube-system` for
  controller-manager health).
- **Job ran but at the "wrong" time** — check the cluster/node's
  configured time zone against your assumption; `spec.timeZone` (a
  more recent CronJob field) can pin this explicitly rather than
  relying on cluster-default assumptions.
- **Runs piling up unexpectedly** — check `concurrencyPolicy`; `Allow`
  (the default!) is easy to leave unset by accident for a task that
  actually needed `Forbid`.
- **Old Jobs/Pods accumulating** — tune
  `successfulJobsHistoryLimit`/`failedJobsHistoryLimit` down if history
  isn't being reviewed, or up temporarily while actively debugging a
  flaky scheduled task.

## Best Practices

- Set `concurrencyPolicy` deliberately, not by accepting the `Allow`
  default unconsidered — ask explicitly "is it safe for two runs of this
  task to overlap?"
- Always set `startingDeadlineSeconds` for time-sensitive tasks (e.g.,
  "must run within business hours") so a missed tick during a control
  -plane outage doesn't silently fire hours late once things recover.
- Keep the actual work idempotent/safe-to-rerun, exactly per Chapter 18 —
  CronJobs inherit every one of Job's retry semantics, including creating
  fresh Pods on failure.
- Alert on CronJob failures explicitly (Chapter 27/28's monitoring and
  logging) — a silently-failing nightly backup is a classic "nobody
  noticed for three months" incident category.

## Interview Questions

1. **What's the relationship between a CronJob, a Job, and Pods?**
   CronJob creates a new Job object on each scheduled tick per its cron
   expression; that Job then creates and manages Pods exactly as Chapter
   18 describes — CronJob never touches Pods directly.
2. **What does `concurrencyPolicy: Forbid` do, and when would you use
   it?** Skips a new scheduled run entirely if the previous run's Job is
   still active — use it whenever concurrent runs of the same task would
   be harmful (e.g., writing to the same output location).
3. **What happens if the CronJob controller is down when a scheduled tick
   was supposed to fire?** The tick is missed; `startingDeadlineSeconds`
   determines how late a catch-up run is still allowed to start once the
   controller recovers, versus being skipped as too-late.
4. **How would you manually trigger an ad-hoc run of a CronJob's task
   without waiting for its schedule?** `kubectl create job
   <name> --from=cronjob/<cronjob-name>` — creates a one-off Job from the
   same template immediately.
5. **What's the cluster-native equivalent problem CronJob solves versus a
   traditional Linux crontab entry?** Running scheduled tasks tied to one
   specific host's crontab versus letting the scheduler place them on any
   healthy node, with cluster resource access (ConfigMaps/Secrets/
   Services) and Job's retry semantics built in.

## Mini Assignment

Create a CronJob that runs every minute with `concurrencyPolicy: Forbid`
and a task that takes about 90 seconds. Watch for at least 4-5 scheduled
minutes and document, from `kubectl get jobs` timestamps and `kubectl get
events`, exactly which ticks were skipped due to the previous run still
being active, confirming the concurrency policy's effect with real
timestamps rather than assuming it worked.

## Lesson Summary

- A CronJob is a thin scheduler layered on Job — at each cron tick it
  creates a fresh Job object, which behaves exactly as Chapter 18
  describes from that point on.
- `concurrencyPolicy` (Allow/Forbid/Replace) is the setting most worth
  choosing deliberately, since the default (`Allow`) is wrong for any
  task unsafe to run concurrently with itself.
- `startingDeadlineSeconds` prevents a backlog of missed runs from all
  firing at once after a control-plane outage.
- `kubectl create job --from=cronjob/<name>` gives you an ad-hoc,
  on-demand run using the same template, without touching the schedule.

---

### Before Chapter 20 (DaemonSets) — tell me:

1. What creates the actual Pods for a CronJob's scheduled work — the
   CronJob controller directly, or something else?
2. Why would `concurrencyPolicy: Allow` be actively dangerous for a
   backup job, specifically?
3. From the Mini Assignment — which scheduled ticks got skipped, and does
   the timing line up with your task's ~90-second runtime against a
   1-minute schedule?
