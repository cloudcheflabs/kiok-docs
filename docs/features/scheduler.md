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

Omit `schedule` for a **manual-only** DAG — it never fires on its own and runs only when triggered.

## Manual triggers

Any DAG — scheduled or not — can be triggered on demand from the admin UI's **Run** button, the REST API, or the CLI:

```bash
bin/submit.sh trigger daily_etl --token <jwt>
bin/submit.sh trigger daily_etl --args date=2026-05-16 --token <jwt>
```

Trigger arguments are passed into the run and made available to its tasks.

## Catchup

When a DAG is registered (or the cluster was down across one or more scheduled fire times), `catchup` decides what happens to the missed intervals:

- `catchup: false` (default) — skip missed intervals; the DAG resumes from the next upcoming fire time.
- `catchup: true` — create a run for each missed interval so the schedule is backfilled.

## Concurrency

`kiok.scheduler.max.global.concurrency` caps how many task assignments are in flight cluster-wide, so a burst of scheduled runs cannot overwhelm the Workers. Tasks beyond the cap wait until a slot frees.

## Run lifecycle

A run moves through `PENDING → RUNNING → SUCCESS` / `FAILED` (or `CANCELLED`). Each task within it has its own state. The leader's task coordinator hands the run to a Worker, which drives it to completion — see [Worker-Driver Execution](execution.md).
