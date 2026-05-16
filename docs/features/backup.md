# Backup &amp; Restore

kiok can ship the entire cluster's state to an S3-compatible target, and restore it later. Backup is **manual-only** and opt-in — you enable it and trigger runs from the admin UI.

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

The page also has a **Back up now** button that triggers an immediate backup.

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
