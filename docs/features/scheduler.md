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

## Catchup — recovering intervals missed during an outage

`catchup` decides what happens to fire times that passed while the cluster was down.

```yaml
dag:
  id: daily_etl
  schedule: "0 2 * * *"
  catchup: true          # default false
```

- **`catchup: false` (default)** — missed intervals are skipped. The DAG resumes from the next
  upcoming fire time and never looks backwards.
- **`catchup: true`** — on the first scheduler evaluation after a restart, the DAG creates one run
  per interval that passed since its **last run**, each tagged `trigger=catchup` and stamped with
  the interval it stands for (not with the moment it was created), so a backfilled run carries the
  time its data belongs to and sorts into history where it belongs.

### Where it resumes from

The resume point is the DAG's **most recent run**, not a separately persisted watermark. That is the
only definition that stays true across a restart: the runs are already durable, and "the last
interval that actually produced a run" is exactly what an operator means by *where did we leave off*.

A DAG that has **never run** arms from now even with `catchup: true`. Without a start date,
backfilling a freshly registered DAG would have to reach back to an arbitrary point, so it does not
try.

### The backfill is capped

A gap can be arbitrarily long — a cluster down for a week with an hourly DAG is 168 intervals — and
releasing all of them at once is an outage of its own. `kiok.scheduler.catchup.max.runs` (default
`24`) bounds it. When the gap is larger, the **most recent** intervals are run and the older ones are
dropped with a `WARN` naming the range that was let go:

```
WARN  Scheduler - Catch-up capped at 24 run(s); dropping 144 older missed interval(s)
      from 2026-09-12T01:00:00Z to 2026-09-18T00:00:00Z. Trigger them by hand if they matter.
```

Fresher data is worth more than stale data, which is why the recent end is kept. If the older
intervals matter, trigger them by hand or give the task an argument-driven date range (see
[Date Placeholders](date-placeholders.md)) so one run can cover the gap.

### What catchup does not cover

**A maintenance window.** Intervals that come due while a
[maintenance window](cluster-maintenance.md) is open are skipped and never revisited, `catchup` or
not. That is a deliberate operator action, not the unplanned outage catchup exists for — and a window
kept open overnight would otherwise release twelve hours of runs the instant it closed. Those skipped
firings are logged at `WARN` on the leader so there is a record of them.

**Runs that were already created.** If a run existed but had not finished when the leader died, it is
not a missed interval — the task coordinator recovers it on the next leadership change and reassigns
it to a healthy worker. That path is independent of `catchup` and applies to every DAG. See
[Worker-Driver Execution](execution.md).

The distinction is whether a run record exists:

| What happened | Recovered by |
| --- | --- |
| A run was created, then the leader died mid-execution | Run recovery — always, regardless of `catchup` |
| The cluster was down, so no run was ever created | `catchup: true` only |
| A maintenance window was open | Neither — skipped by design, logged at `WARN` |

## Concurrency

`kiok.scheduler.max.global.concurrency` caps how many task assignments are in flight cluster-wide, so a burst of scheduled runs cannot overwhelm the Workers. Tasks beyond the cap wait until a slot frees.

## Pausing the scheduler

Opening a [maintenance window](cluster-maintenance.md) stops the scheduler creating runs, stops
pending runs being dispatched to Workers, and refuses manual triggers — without killing runs already
executing. It is the supported way to hold the cluster still while restoring a backup or rotating
keys.

## Run lifecycle

A run moves through `PENDING → RUNNING → SUCCESS` / `FAILED` (or `CANCELLED`). Each task within it has its own state. The leader's task coordinator hands the run to a Worker, which drives it to completion — see [Worker-Driver Execution](execution.md).
