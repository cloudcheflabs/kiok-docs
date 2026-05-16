# Job Logs

Every task's stdout/stderr, and every run's driver orchestration log, is captured, streamed live to the admin UI, and retained for later inspection.

## Capture and streaming

As a Worker runs a task, it batches the task's output and pushes it to the leader over the internal protocol (`kiok.joblog.worker.batch.*` controls batch size and interval). The leader stores it and serves it back to the admin UI.

Each run also has a **run log** — the driver's orchestration log (which task was dispatched, when, with what result) — stored under a reserved task id. The UI streams both per-task logs and the run log incrementally: an offset-based tail fetches only newly-appended bytes, with a **Live / Paused** toggle.

## Where logs live

Job logs are stored in a dedicated leader-side RocksDB store, **separate from the metadata store**, and KMS envelope-encrypted at rest. There are two retention modes:

- **No S3 archival (default)** — the leader RocksDB store is the *permanent home*. A completed task's log stays there indefinitely.
- **S3 archival configured** — a periodic flush (`kiok.joblog.flush.interval.ms`, default 10s) and a final flush on task completion copy each log to S3, after which the RocksDB buffer is dropped. The periodic flush means even a long task that dies still has its log up to the last interval.

Configure the S3 destination with `kiok.joblog.s3.*` (endpoint, region, bucket, prefix, credentials).

## Reading a completed log

A log read checks the leader RocksDB store first; if the buffer is empty **and** S3 archival is on, it falls back to the S3 archive. So a completed task whose RocksDB buffer was dropped after archival is still served transparently — the UI and CLI never need to know which tier holds the bytes.

## Durability

In the default no-S3 configuration, job logs live only in the leader's RocksDB. To protect them against a leader disk loss, enable [Backup &amp; Restore](backup.md) — job logs are one of the backed-up stores, so a restore brings completed-run logs back along with the rest of cluster state.
