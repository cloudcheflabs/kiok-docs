# Scheduler &amp; Triggers

kiok decides *when* a DAG runs. A DAG can fire on a cron schedule, be triggered on demand, or both.

## Cron schedules

A DAG carries an optional 5-field cron expression:

```yaml
dag:
  id: daily_etl
  schedule: "0 2 * * *"   # every day at 02:00
```

A leader-only scheduler loop ticks at a short interval (`kiok.scheduler.tick.interval.ms`, default 1s) and, for each DAG whose next fire time is due, creates a new run. Because scheduling runs only on the leader, a DAG never double-fires when the cluster has multiple Masters.

At most one run is created per DAG per tick, and the tick advances the DAG's watermark whether or not
it fired — which is why missed intervals are skipped rather than accumulated. See below.

Omit `schedule` for a **manual-only** DAG — it never fires on its own and runs only when triggered.

## Manual triggers

Any DAG — scheduled or not — can be triggered on demand from the admin UI's **Run** button, the REST API, or the CLI:

```bash
bin/submit.sh trigger daily_etl --token <jwt>
bin/submit.sh trigger daily_etl --args date=2026-05-16 --token <jwt>
```

Trigger arguments are passed into the run and made available to its tasks.

## Missed fire times are skipped

If the cluster is down across one or more scheduled fire times — or the schedule comes due while a
[maintenance window](cluster-maintenance.md) is open — **those intervals are not run, and are not
replayed afterwards**. The DAG resumes from the next upcoming fire time.

The same applies when a DAG is first registered: it arms from that moment and does not reach back
over its own history.

!!! warning "`catchup: true` is accepted but does nothing"
    The `catchup` field parses and is stored on the DAG, and the SDK exposes a setter for it, but
    **no code reads it** — there is no backfill path in the scheduler. Setting `catchup: true` does
    not create runs for missed intervals, and a DAG carrying it behaves exactly like one that does
    not.

    Do not rely on it to cover an outage. For intervals you cannot afford to lose, trigger them by
    hand once the cluster is back, or give the task an argument-driven date range
    (see [Date Placeholders](date-placeholders.md)) so one run can cover the gap.

Firings skipped because of a maintenance window are logged at `WARN` on the leader, so there is at
least a record of what was missed. Firings skipped because the cluster was down are not — the
process that would have logged them was not running.

## Concurrency

`kiok.scheduler.max.global.concurrency` caps how many task assignments are in flight cluster-wide, so a burst of scheduled runs cannot overwhelm the Workers. Tasks beyond the cap wait until a slot frees.

## Pausing the scheduler

Opening a [maintenance window](cluster-maintenance.md) stops the scheduler creating runs, stops
pending runs being dispatched to Workers, and refuses manual triggers — without killing runs already
executing. It is the supported way to hold the cluster still while restoring a backup or rotating
keys.

## Run lifecycle

A run moves through `PENDING → RUNNING → SUCCESS` / `FAILED` (or `CANCELLED`). Each task within it has its own state. The leader's task coordinator hands the run to a Worker, which drives it to completion — see [Worker-Driver Execution](execution.md).
