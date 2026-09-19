# Cluster Maintenance Mode

A cluster-wide switch that stops kiok starting work, so an operator can restore a backup, rotate keys, or replace Workers without a run beginning underneath them.

## Why this one is different

In a request-driven service, closing the API closes the service. kiok is not request-driven. Four loops run on the leader and create work with nobody asking:

| Loop | What it does on its own |
| --- | --- |
| Cron scheduler | creates runs when a DAG's schedule comes due |
| Task coordinator | hands pending runs to Workers |
| Git sync | pulls a repository and registers DAG definitions |
| Backup scheduler | ships cluster state to S3 |

A maintenance mode that only refused HTTP requests would be decorative — the 2am cron would fire in the middle of your restore regardless. So this switch is read by **the loops first**, and the REST gate is the smaller half of it.

## What stops, and what does not

| Stopped | Keeps running |
| --- | --- |
| The cron scheduler — no runs are created | Runs **already executing** |
| Dispatch — pending runs stay queued, Workers get nothing new | `POST /runs/{id}/cancel` |
| Git sync — no DAG definitions are registered | Driver-failure detection and reassignment |
| `POST /dags/{id}/runs` — manual triggers get `503` | The backup scheduler |
| | DAG registration and deletion, IAM, KMS, connections |
| | Every read: DAG list, run state, task logs, topology, metrics |

Three of those deserve their reasoning, because a blunter switch would get them wrong.

**Cancel stays open.** The first thing an operator does after opening a window is stop the runs still executing. A switch that blocked `cancel` would be blocking the work it exists to enable.

**Runs already executing are not killed.** Opening a window is not an emergency stop. Killing live runs would turn a routine maintenance window into a pile of failed runs and half-written outputs; cancel the ones you need stopped, deliberately.

**The backup scheduler keeps running.** Taking a backup during a maintenance window is usually the *point*, not something to prevent.

## Missed cron firings are skipped, not queued

A schedule that comes due inside the window **does not fire, and is not replayed afterwards**.

This is a deliberate choice and the alternative is worse: a window left open overnight would release twelve hours of accumulated runs the instant it closed, which is an outage of its own — a stampede against the same Workers and the same downstream systems the window was protecting.

Every skipped firing is logged at `WARN` on the leader:

```
WARN  Scheduler - DAG daily_etl was due at 2026-09-19T02:00:00Z but the cluster is
      in maintenance mode -- skipping this firing
```

That log exists because a silently missed schedule is the kind of thing an operator discovers weeks later, from the data being wrong. Check it after closing a window and re-trigger by hand what mattered.

`catchup: true` does **not** bring these back either. Catch-up exists for an unplanned outage, where
the cluster was down and nobody chose to miss anything; a maintenance window is a deliberate operator
action, and replaying it on exit is the stampede this design avoids. See
[Scheduler &amp; Triggers](scheduler.md#what-catchup-does-not-cover).

!!! warning "Plan windows around your schedules"
    Because firings are dropped, a long window across a busy schedule loses those runs permanently. Prefer a window between fire times, and for anything that must not be missed, trigger it manually once the window closes.

## Pending runs are frozen, not dropped

Runs that were already triggered when the window opened stay `PENDING` and dispatch when it closes.

The queue cannot grow meanwhile — the scheduler creates nothing and manual triggers are refused — so what drains afterwards is whatever existed at the moment the window opened. Bounded, and legitimately owed an execution.

## Where the setting lives

In the cluster metadata store — the encrypted RocksDB config store — under `cluster.maintenance.mode`, so it rides the same snapshot replication as DAG definitions and run records.

ZooKeeper would have been the other candidate and is the wrong one: **ZooKeeper holds node state** (membership, leadership, readiness) while **settings live in RocksDB**. That split is the same across every Cloud Chef Labs product.

Storing it is what makes it hold:

- A leader restarted mid-window reloads it on boot and comes back with the loops still paused.
- A follower that **wins an election** mid-window starts as a leader that is already paused, rather than one that begins scheduling because it was just elected. This is the case that a memory-only flag would get catastrophically wrong.

Each master caches the value and re-reads it after every snapshot import, including after a backup restore — which replaces every config key, this one among them, so the window ends up however the backup had it. Check the banner after a restore rather than assuming the window you opened is still open.

## Turning it on and off

From the admin UI: **Topology → Enter Maintenance**. A confirmation appears first — it says what stops and what does not — and a banner stays on screen while the window is open.

Or over REST:

```bash
TOKEN=$(curl -sf -X POST http://localhost:8080/api/v1/auth/login \
    -H 'Content-Type: application/json' \
    -d '{"user":"admin","password":"…"}' | jq -r .accessToken)

# status — answered by whichever master receives it, from its own cache
curl -sf http://localhost:8080/api/v1/admin/maintenance -H "Authorization: Bearer $TOKEN"

# on
curl -sf -X POST http://localhost:8080/api/v1/admin/maintenance \
    -H "Authorization: Bearer $TOKEN" -H 'Content-Type: application/json' \
    -d '{"enabled":true}'

# off
curl -sf -X POST http://localhost:8080/api/v1/admin/maintenance \
    -H "Authorization: Bearer $TOKEN" -H 'Content-Type: application/json' \
    -d '{"enabled":false}'
```

`POST` requires the `admin:Maintenance` action on `cluster` and is forwarded to the leader, which owns the store. `GET` is answered locally by any master, so you can confirm the window reached every master rather than only the one that was told to open it.

## What a refused trigger looks like

```
HTTP/1.1 503 Service Unavailable
Retry-After: 30
Content-Type: application/json

{"error":"cluster is in maintenance mode; new runs are not accepted",
 "maintenanceMode":true,"retryAfterSeconds":30}
```

`503` with `Retry-After` rather than `403`: the caller is not forbidden, the cluster is closed for now, and every HTTP client already knows to back off on that pair.

```properties
# How long a client is told to wait before retrying a refused trigger (seconds).
kiok.cluster.maintenance.retry.after.seconds=30
```

The SDK and CLI paths are refused too — the check sits in the scheduler itself as well as at the REST layer, because a window that one entry point honours and another does not is worse than none.

## When to use it

| Use it for | Don't use it for |
| --- | --- |
| Restoring a backup over live DAG and run state | Adding a Master or Worker (neither needs a window) |
| KMS rotation | A rolling upgrade (leader transitions are already handled) |
| Replacing or draining Workers as a group | A single Worker restart (driver failover covers it) |
| Investigating a bad run without new ones piling in | Any window long enough to drop schedules you need |

## Verifying it

`tests/maintenance-mode-e2e.sh` in the product repository runs the whole cycle against a compose cluster: a DAG on a one-minute cron **fires normally first**, the window opens and no run is created across a full cron period, a manual trigger is refused with `503` + `Retry-After`, cancel and reads and DAG registration keep working, the skipped firing is present in the log, the leader is restarted and comes back still paused, and then the window closes and the cron fires again.

Proving the cron fires *before* opening the window is the part that makes the rest mean anything — without it, "no runs appeared" would pass against a scheduler that was broken all along. Proving it fires *again* afterwards matters for the mirror reason.

## See also

- [Scheduler &amp; Triggers](scheduler.md) — how firing works normally, and what happens to missed intervals.
- [Backup &amp; Restore](backup.md) — the operation a window is most often opened for.
- [Worker-Driver Execution](execution.md) — why a run already executing is unaffected.
