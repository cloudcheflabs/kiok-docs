# Backup &amp; Restore

kiok can ship the entire cluster's state to an S3-compatible target, and restore it later. Backup is opt-in (off by default) and runs in two trigger modes that share the same code path:

- **Manual** — *Back up now* button in the admin UI, or `POST /api/v1/admin/backup`.
- **Cron** — a 5-field UNIX cron expression evaluated by a leader-only scheduler. Set it from the admin UI or `PUT /api/v1/admin/backup/cron`.

The destination is any AWS-compatible S3 endpoint: AWS S3, MinIO, ShannonStore, or similar. Pointing it at a separate system gives you a true offsite copy.

## What Gets Backed Up

A backup run captures five stores under one shared backup id, so a single restore reassembles a consistent point in time:

- **metadata** — DAG definitions, run history, bundle metadata, and git-repo configuration.
- **kms** — the KMS keystore.
- **iam** — users, groups, policies, and access keys.
- **connections** — the encrypted connection store.
- **joblog** — per-task and per-run job logs.

Job logs are included because, in the default no-S3-archival configuration, they live only in the leader's RocksDB — without backup, a leader disk loss would lose every completed run's log.

## What Is Encrypted

Nothing secret reaches the bucket in the clear. Each backup id is a folder with up to six objects:

| Object | Contents | Sealed with |
|---|---|---|
| `kms.bin` | the KMS keystore | a key derived from the **master key** (`KIOK_MASTER_KEY`, PBKDF2) — the same sealing the keystore has on disk |
| `metadata.bin`, `iam.bin`, `connections.bin`, `joblog.bin` | DAGs and runs, users / password hashes / access keys / user tokens, connection credentials, job logs | AES-256-GCM envelope encryption; each data key is wrapped with the backup key (`kiok.backup.kms.key.id`, default `backup-encryption`) **from the keystore in `kms.bin`** |
| `manifest.json` | backup id, time, `"format": 2`, and per store its SHA-256 and which backup id holds its bytes | not encrypted — it holds no secrets, and must be readable before any key is |

So a backup opens with the master key alone: the master key opens `kms.bin`, and the keys in it open the
other stores. Neither the cluster that made the backup nor its KMS directory is needed.

**The KMS must be on.** With `kiok.kms.enabled=false` there is no key to seal anything with, so a backup
is refused with `503` and the reason (`backups are encrypted with the KMS, and the KMS is off ...`) rather
than writing passwords, access keys and connection credentials to S3 in the clear. Nothing is written.

## Configuration

The **Backup &amp; Restore** page in the admin UI exposes:

- **Enabled** — master switch (default: off). The live toggle is persisted in the metadata store.
- **Backup S3 target** — either the default (the job-log S3 configuration) or a stored S3 [connection](connections.md) selected by id.
- **Automatic backup schedule** — a 5-field UNIX cron expression (e.g. `0 2 * * *` for daily at 02:00). Same grammar as the DAG [scheduler](scheduler.md). Leave empty to disable.
- **Retention** — keep the newest *N* backups and/or delete backups older than *D* days. Both default to `0`, which means no limit: nothing is ever deleted until an operator sets one. See [Retention](#retention).

The page also has a **Back up now** button that triggers an immediate backup.

These settings are read from the replicated metadata store each time they are used, so every Master
answers with the same values, and a Master that becomes leader after a failover keeps running the
schedule the previous leader had — it does not fall back to what it read when it started. Each part of
the page loads on its own: an S3 error listing backups is shown next to the other settings instead of
hiding them. Every backup route, including the reads, is administrator-only.

### Cron scheduling

The cron expression lives on the leader as a `BackupScheduler` daemon that ticks once per scheduler interval (the same cadence as the DAG cron scheduler) and fires `backupNow()` once when the cron has a fire time in `(lastChecked, now]`. The schedule is persisted under `backup.cron` in the metadata store, so a leader handoff or master restart re-arms the same cron.

The first tick after the cron is set, or after a leader change, **arms from "now"** — kiok does not back-fire missed cron times on startup. Recovering a missed nightly backup three days late tends to surprise more than skipping it.

The admin UI shows the next scheduled fire time next to an active cron, in the browser's time zone (the cron itself is evaluated in the leader's zone). The two REST endpoints:

```bash
# Read the current cron and its next-fire timestamp
curl -H "Authorization: Bearer $TOKEN" \
  http://kiok-master:8080/api/v1/admin/backup/cron

# Set or clear (null / empty)
curl -X PUT -H "Authorization: Bearer $TOKEN" -H 'Content-Type: application/json' \
  -d '{"cron": "0 2 * * *"}' \
  http://kiok-master:8080/api/v1/admin/backup/cron
```

An invalid cron expression is rejected with HTTP 400 — the parser's message is surfaced in the response body. Set `cron: null` or an empty string to clear.

## Incremental by Design

Each store is content-hashed (SHA-256). If a store's bytes are unchanged since the last backup, it is **not re-uploaded** — the new backup's manifest simply points at the backup id that already holds it. A backup taken with no cluster activity uploads almost nothing.

Each backup writes a `manifest.json` recording, per store, its hash and which backup id holds its bytes — so the incremental chain is always resolvable from any single manifest.

## Retention

Without a limit, every backup stays in the bucket forever. Retention deletes old backups automatically.
It has two limits, set together in the **Retention** card of the Backup &amp; Restore page:

| Field | API field | Meaning |
|---|---|---|
| **Keep the newest** | `keepLast` | Keep this many of the newest backups; older ones are deleted. `0` = no count limit. |
| **Delete older than (days)** | `maxAgeDays` | Delete backups taken more than this many days ago (the age comes from the backup id, which is its UTC creation time). `0` = no age limit. |

Both default to **`0` — no limit**, so a cluster that has never set retention keeps every backup, exactly as
before retention existed. When both are set, a backup is deleted if it falls outside **either** limit.

**When it runs.** Retention is applied by the leader right after every successful backup — **Back up now**,
`POST /api/v1/admin/backup`, and every cron-scheduled backup. Saving the limits deletes nothing by itself;
the page's **Apply now** button (with a confirmation dialog) or `POST /api/v1/admin/backup/retention`
applies them immediately. **Apply now** is disabled while there is no limit, or while the fields hold
unsaved changes. The response of a backup lists what retention deleted (`"pruned"`), and the page says so:
`Backup stored: 20261031T020000.123Z (retention deleted 1 old backup)`. If retention fails (for example
an S3 delete error), the backup is still reported as stored and the error is logged on the leader.

**What is always kept.**

- The **newest backup** is never deleted, whatever the limits say — even `maxAgeDays` on a cluster whose
  last backup is old. Retention never leaves the bucket without a backup to restore.
- Because backups are [incremental](#incremental-by-design), a kept backup's manifest can point at store
  objects that live in an older backup's folder. Retention deletes a dropped backup's `manifest.json`
  first — it disappears from **Backups in S3** at once — and deletes its store objects (`*.bin`) only
  when no kept backup's manifest points at them. Folders that keep such objects are not listed (they have
  no manifest) and are cleaned up by a later run, once nothing points at them any more.
- If any kept backup's manifest cannot be read (for example a backup in the older format sealed with a
  key this cluster lacks), retention still deletes the dropped manifests but leaves every store object in
  place, and logs a warning.
- Only objects shaped like a backup (`<prefix>/<backup id>/<file>`) are ever touched; other objects
  under the prefix are left alone.

**Where it is stored.** Like the schedule, the limits live in the replicated metadata store
(`backup.retention.keepLast`, `backup.retention.maxAgeDays`), not in `kiok.properties`, so every Master
answers with the same values and a new leader after a failover applies the same limits. Because they are
part of the metadata, a [restore](#restore) brings back the limits that were in effect when that backup was
taken — check the Retention card after a restore.

```bash
# Read the limits
curl -H "Authorization: Bearer $TOKEN" \
  http://kiok-master:8080/api/v1/admin/backup/retention
# {"keepLast":0,"maxAgeDays":0}

# Keep the newest 14 backups and nothing older than 30 days
curl -X PUT -H "Authorization: Bearer $TOKEN" -H 'Content-Type: application/json' \
  -d '{"keepLast": 14, "maxAgeDays": 30}' \
  http://kiok-master:8080/api/v1/admin/backup/retention

# Apply them now; returns the deleted backup ids
curl -X POST -H "Authorization: Bearer $TOKEN" \
  http://kiok-master:8080/api/v1/admin/backup/retention
# {"deleted":["20261001T020000.004Z","20260930T020000.011Z"]}
```

A negative or non-whole value is refused with `400` and the reason. A field left out of the `PUT` body is
set to `0`. `POST` answers `409` when no S3 target is configured.

## Restore

The same admin page lists the backups in S3, newest first, **10 per page** with previous / next and page
buttons under the table; the header shows the total count. Each row shows the time the backup was taken
(a backup id is its UTC creation time, e.g. `20261025T095700.401301630Z`, shown in the browser's zone) and
has a **Restore** button.

> Restore is **destructive** — it overwrites current cluster state. The UI asks for confirmation in a dialog first; **Cancel** restores nothing.

The flow:

1. The operator clicks **Restore** on a backup row and confirms.
2. The leader reads that backup's `manifest.json`.
3. For each of the five stores, it fetches the bytes, following the incremental pointer to whichever backup id actually holds them.
4. It opens `kms.bin` with its master key and decrypts **every** store with the keys in it — before anything is replaced. A backup that cannot be read leaves the cluster exactly as it was.
5. It imports the keystore first, then metadata, IAM, connections and job logs (re-encrypted on disk with the restored keys) — overwriting the live stores.
6. The leader sends the restored state (KMS first, then IAM, connections and metadata) to every other master and worker, so a user removed by the restore cannot still sign in through a follower.
7. The restore completes; the cluster is now at the chosen snapshot.

A restore that cannot run answers `500` with an `error` that says why, and the page shows it — no S3
target, a backup id that does not exist, a missing store, or a backup this cluster cannot decrypt:

- `the backup's keystore (kms.bin) cannot be opened with this cluster's master key` — the masters run
  with another `KIOK_MASTER_KEY` than the cluster that made the backup.
- `this backup is encrypted, and the KMS is off on this cluster` — start the masters with
  `kiok.kms.enabled=true`.
- `this backup was made in the older format ... this cluster does not have that key` — see below.

### Backups from earlier versions

Before this layout (backups without `"format": 2` in `manifest.json`), every object — the manifest and
the keystore included — was sealed with the backup key, which exists only inside the keystore of the
cluster that made it. Such a backup still restores **on that cluster**, as long as its KMS directory
(`data/kms`) is intact; on a new cluster it is refused with the reason above, and the master key alone
cannot open it. Take a new backup after upgrading so that a restorable one exists. Backups written by a
cluster running without the KMS are plain and restore anywhere.

Because the `joblog` store is restored too, completed-run logs come back along with DAGs, runs, IAM, KMS, and connections.

> The **Restore** button is per-backup-row — it appears only once at least one backup exists in S3. On a cluster with no backups yet, the list shows "No backups" and there is nothing to restore. Enable backup and run **Back up now** first.

## REST API

Every route is administrator-only. Writes sent to a follower Master are forwarded to the leader; reads
are answered by any Master from the replicated metadata store.

| Method and path | Body | Answer |
|---|---|---|
| `GET /api/v1/admin/backup` | — | `{"enabled": true}` |
| `PUT /api/v1/admin/backup` | `{"enabled": true}` | `{"enabled": true}` |
| `POST /api/v1/admin/backup` | — | `{"backupId": "...", "pruned": [ids]}`; `503` with `error` if the backup did not run |
| `GET /api/v1/admin/backup/list` | — | every backup id in S3, newest first (a JSON array; the admin UI pages it) |
| `POST /api/v1/admin/backup/restore` | `{"backupId": "..."}` | `{"restored": true}`; `500` with `error` if it did not |
| `GET` / `PUT /api/v1/admin/backup/connection` | `{"connectionId": "my-s3"}` (`null` = default) | `{"connectionId": ...}` |
| `GET` / `PUT /api/v1/admin/backup/cron` | `{"cron": "0 2 * * *"}` (`null` / `""` clears) | `{"cron": ..., "nextFireMillis": ...}`; `400` for a malformed cron |
| `GET` / `PUT /api/v1/admin/backup/retention` | `{"keepLast": 14, "maxAgeDays": 30}` (`0` = no limit) | `{"keepLast": ..., "maxAgeDays": ...}`; `400` for an invalid value |
| `POST /api/v1/admin/backup/retention` | — | `{"deleted": [ids]}`; `409` with no S3 target |

## Disaster Recovery

The intended recovery scenario: the cluster's state is gone — a Master lost its disk, or every Master
and ZooKeeper had to be rebuilt. What a restore onto a brand-new cluster needs:

1. **The same master key.** Start the Masters (and Workers) with the `KIOK_MASTER_KEY` of the cluster
   that made the backup. Nothing else from the old cluster is needed — not its KMS directory, not its
   ZooKeeper data.
2. **The KMS on** (`kiok.kms.enabled=true`, the default).
3. **The backup target.** Point the new cluster at the bucket: the default target is the job-log S3
   settings (`kiok.joblog.s3.*`) with the same `kiok.backup.s3.prefix`; or create an S3
   [connection](connections.md) to the bucket and select it as the **Backup S3 target**.
4. Sign in with the new cluster's admin account (rotate the factory password), turn **Enabled** on,
   pick the latest backup in **Backups in S3**, and **Restore**.

DAGs, run history, users and their passwords, access keys and user tokens, policies, connections, job
logs and the KMS keys come back, on every Master: sign in again with the **backed-up** admin password —
the new cluster's own one is replaced. Everything the new cluster wrote before the restore is replaced
as well, the backup settings included (they come from the backup's metadata).

Keeping the master key safe is essential: it is the one thing that opens a backup. Without the
`KIOK_MASTER_KEY` the cluster ran with, the backup cannot be decrypted — store it apart from the
bucket.
