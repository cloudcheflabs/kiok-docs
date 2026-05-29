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

Job logs are included because, in the default no-S3-archival configuration, they live only in the leader's RocksDB — without backup, a leader disk loss would lose every completed run's log. Every store is KMS envelope-encrypted before it leaves the cluster.

## Configuration

The **Backup &amp; Restore** page in the admin UI exposes:

- **Enabled** — master switch (default: off). The live toggle is persisted in the metadata store.
- **Backup S3 target** — either the default (the job-log S3 configuration) or a stored S3 [connection](connections.md) selected by id.
- **Automatic backup schedule** — a 5-field UNIX cron expression (e.g. `0 2 * * *` for daily at 02:00). Same grammar as the DAG [scheduler](scheduler.md). Leave empty to disable.

The page also has a **Back up now** button that triggers an immediate backup.

### Cron scheduling

The cron expression lives on the leader as a `BackupScheduler` daemon that ticks once per scheduler interval (the same cadence as the DAG cron scheduler) and fires `backupNow()` once when the cron has a fire time in `(lastChecked, now]`. The schedule is persisted under `backup.cron` in the metadata store, so a leader handoff or master restart re-arms the same cron.

The first tick after the cron is set, or after a leader change, **arms from "now"** — kiok does not back-fire missed cron times on startup. Recovering a missed nightly backup three days late tends to surprise more than skipping it.

The admin UI shows the next scheduled fire time next to an active cron. The two REST endpoints:

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

## Restore

The same admin page lists every backup in S3, newest first. Each row has a **Restore** button.

> Restore is **destructive** — it overwrites current cluster state. The UI asks for confirmation first.

The flow:

1. The operator clicks **Restore** on a backup row and confirms.
2. The leader reads that backup's `manifest.json`.
3. For each of the five stores, it fetches the bytes (following the incremental pointer to whichever backup id actually holds them) and imports them — overwriting the live store.
4. The restore completes; the cluster is now at the chosen snapshot.

Because the `joblog` store is restored too, completed-run logs come back along with DAGs, runs, IAM, KMS, and connections.

> The **Restore** button is per-backup-row — it appears only once at least one backup exists in S3. On a cluster with no backups yet, the list shows "No backups" and there is nothing to restore. Enable backup and run **Back up now** first.

## Disaster Recovery

The intended recovery scenario: a Master loses its local disk. Bring up a fresh Master pointed at the same ZooKeeper and master key, open the **Backup &amp; Restore** page, and restore the latest backup from S3 — DAGs, run history, IAM, KMS, connections, and job logs are all reinstated.

Keeping the master key safe is essential: every backed-up store is envelope-encrypted, and without `KIOK_MASTER_KEY` the backup cannot be decrypted.
