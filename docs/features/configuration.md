# Configuration Management

kiok resolves configuration through a priority chain, so the same distribution runs unchanged across laptops, containers, and production hosts. The bundled `kiok.properties` ships every key with its default and an explanatory comment; this page mirrors that file as a complete reference.

## Priority order

Every key is resolved highest-to-lowest:

1. **System property** — a `-D` flag on the command line, e.g. `-Dkiok.master.admin.port=8080`.
2. **Environment variable** — the property name uppercased with dots replaced by underscores: `kiok.master.admin.port` → `KIOK_MASTER_ADMIN_PORT`.
3. **Properties file** — the `kiok.properties` defaults shipped inside the distribution (classpath), then any external `--config` file.
4. **Hard-coded default** — the fallback baked into `KiokConfig`.

A value set at a higher level overrides the same key set lower down.

## Variable substitution

Properties may reference other properties with `${...}`. Most on-disk stores derive from a single base directory, and `${user.dir}` expands to the process working directory:

```properties
kiok.base.data.dir=./data
kiok.metadata.rocksdb.path=${kiok.base.data.dir}/metadata
kiok.log.path=${user.dir}/logs
```

!!! note "Optional keys"
    Keys marked **optional** below are commented out (or blank) in the shipped file. They take effect only when you set them via the properties file, an environment variable, or a `-D` system property. The `kiok.user.*` client credentials are supplied by **clients** (the Java/Python SDKs and `submit.sh`) at submit time and are never read from the server-side file.

## Base

| Property | Default | Description |
|---|---|---|
| `kiok.base.data.dir` | `./data` | Root directory for all local on-disk node state. Most paths below default to a subdirectory of this. Point it at a persistent volume — ephemeral storage loses all node state on restart. |

## ZooKeeper

ZooKeeper backs master leader election and the node registry. Every master and worker in a cluster must point at the same ensemble.

| Property | Default | Description |
|---|---|---|
| `kiok.zk.serverList` | `localhost:2181` | Comma-separated ZooKeeper connect string (`host:port,...`). |
| `kiok.zk.rootPath` | `/kiok` | ZooKeeper namespace (chroot) for all kiok znodes, so multiple clusters can share one ensemble. |
| `kiok.zk.sessionTimeoutMs` | `30000` | Session timeout (ms). On expiry, ephemeral znodes (e.g. leadership) drop and re-election triggers. Larger tolerates GC/network pauses but detects failures slower. |
| `kiok.zk.connectionTimeoutMs` | `10000` | Maximum time (ms) to wait when first establishing the TCP connection to ZooKeeper. |

## Master

| Property | Default | Description |
|---|---|---|
| `kiok.master.host` | `0.0.0.0` | Bind address for the master's listeners. `0.0.0.0` listens on all interfaces; set a specific IP to restrict exposure. |
| `kiok.master.admin.port` | `8080` | TCP port for the admin HTTP server (REST API + admin UI + auth). Operators and the admin UI connect here. |
| `kiok.master.admin.context.path` | `/` | HTTP context-path prefix the admin server is mounted under. Set e.g. `/kiok` to serve behind a reverse-proxy path prefix. |
| `kiok.master.internal.port` | `19999` | TCP port for the internal node-to-node binary NIO protocol. Workers and other masters connect here; keep it firewalled to the cluster. |
| `kiok.master.admin.worker.threads` | `8` | Thread-pool size serving admin HTTP requests. Raise for higher concurrent admin/API load. |
| `kiok.master.admin.ui.static.path` | `./admin-ui` | Filesystem path to the static admin UI assets served by the admin HTTP server. |
| `kiok.master.broadcast.timeout.ms` | `5000` | Timeout (ms) for a leader's fan-out RPC to all registered nodes. |
| `kiok.master.leader.deference.window.ms` | `3000` | Sticky-incumbent election: on restart a former leader defers joining the election this long if a prior leader still appears alive, so the incumbent reclaims leadership without a disruptive toggle. `0` disables. |
| `kiok.master.startup.leader.timeout.ms` | `30000` | Startup gate (ms): how long a master waits at boot for leadership to resolve before proceeding (logs a warning and continues on timeout). |
| `kiok.master.startup.follower.timeout.ms` | `60000` | Startup gate (ms): how long a follower master waits for the leader to become ready before fetching cluster state. Should exceed the leader's own startup time. |
| `kiok.master.startup.poll.interval.ms` | `100` | Poll interval (ms) inside master startup wait-loops. Lower wakes faster but burns more idle CPU. Floored to 1 ms. |

## Worker

| Property | Default | Description |
|---|---|---|
| `kiok.worker.host` | `0.0.0.0` | Bind address for the worker's internal NIO listener. `0.0.0.0` = all interfaces. |
| `kiok.worker.internal.port` | `19998` | TCP port for the worker's internal node-to-node binary NIO protocol. The leader connects here to assign DAG runs and push state. |
| `kiok.worker.shutdown.drain.timeout.seconds` | `60` | Graceful-shutdown drain window (seconds): on shutdown the worker stops accepting new work and waits up to this long for in-flight tasks before forcing exit. |
| `kiok.worker.task.slots` | `8` | Maximum tasks this worker runs concurrently (its task-slot capacity). Raising it increases per-worker throughput at the cost of CPU/memory/IO contention. |
| `kiok.worker.startup.leader.timeout.ms` | `120000` | Startup gate (ms): how long a worker waits at boot for the master leader to become ready before fetching cluster state (logs a warning and continues on timeout). |
| `kiok.worker.startup.poll.interval.ms` | `1000` | Poll interval (ms) inside the worker's await-leader-ready loop. Floored to 1 ms. |

## Internal Protocol (custom NIO)

| Property | Default | Description |
|---|---|---|
| `kiok.internal.nio.worker.threads` | `4` | I/O worker threads for each node's internal NIO server (used by both master and worker), sizing the event-loop pool handling node-to-node RPC. |

## KMS

The built-in KMS performs envelope encryption of at-rest stores and, optionally, the internal protocol.

| Property | Default | Description |
|---|---|---|
| `kiok.kms.enabled` | `true` | Master switch for the built-in KMS. When `true`, a `KIOK_MASTER_KEY` must be available to unseal it. Disabling turns off encryption of metadata/IAM/connection/job-log/backup data. |
| `kiok.kms.rocksdb.path` | `${kiok.base.data.dir}/kms` | RocksDB directory where the KMS persists wrapped data-encryption keys. Must be persistent and protected. |
| `kiok.kms.master.key.env` | `KIOK_MASTER_KEY` | Name of the environment variable holding the KMS master-key passphrase (the var name, not the key). Stretched via PBKDF2 to derive the key that unseals the KMS. |
| `kiok.kms.pbkdf2.iterations` | `200000` | PBKDF2 iteration count for deriving the KMS root key. Higher is stronger but slows unseal at startup; changing it after keys are written may break unsealing of existing data. |
| `kiok.kms.encrypt.internal.protocol` | `true` | When `true`, internal node-to-node NIO payloads are KMS-encrypted in transit. Set `false` only on a fully trusted network. Effective only when KMS is enabled. |
| `kiok.kms.internal.protocol.key.id` | `internal-protocol` | KMS key id used to encrypt/decrypt internal-protocol payloads. All nodes must agree on this id. |
| `kiok.kms.fetch.max.retries` | `60` | Worker state-fetch retry budget: attempts a worker makes to fetch KMS/IAM/connection/metadata snapshots from a ready master at startup before giving up. |
| `kiok.kms.fetch.retry.interval.ms` | `1000` | Delay (ms) between worker state-fetch retry attempts. |

## IAM

| Property | Default | Description |
|---|---|---|
| `kiok.iam.rocksdb.path` | `${kiok.base.data.dir}/iam` | RocksDB directory for the IAM store (users, roles, credentials). Persistent; contents are envelope-encrypted by KMS. |
| `kiok.iam.admin.user` | `admin` | Bootstrap admin username created on first startup if no users exist. |
| `kiok.iam.admin.password` | `admin` | Bootstrap admin password seeded on first startup. **Change this** in any real deployment (or reset later via the admin recovery socket). |

## Metadata (DAGs, runs, schedules)

| Property | Default | Description |
|---|---|---|
| `kiok.metadata.rocksdb.path` | `${kiok.base.data.dir}/metadata` | RocksDB directory storing DAG definitions, run history, task states, and schedules. Persistent. |
| `kiok.metadata.kms.key.id` | `metadata-encryption` | KMS key id used to envelope-encrypt the metadata store at rest. Effective when KMS is enabled. |

## Connections (git creds, S3 creds, etc.)

| Property | Default | Description |
|---|---|---|
| `kiok.connection.rocksdb.path` | `${kiok.base.data.dir}/connections` | RocksDB directory for the connection store (named git/S3 credentials referenced by DAGs/operators by id). Persistent; contents are KMS-encrypted. |
| `kiok.connection.kms.key.id` | `connection-encryption` | KMS key id used to envelope-encrypt the connection store at rest. |

## RPC Timeouts

| Property | Default | Description |
|---|---|---|
| `kiok.master.rpc.timeout.ms` | `30000` | Default timeout (ms) for master-level RPCs. |
| `kiok.internal.rpc.timeout.ms` | `10000` | Timeout (ms) for each internal node-to-node NIO request/response (state fetches, run assignments, metrics, leader pushes). Raise on slow/loaded networks. |

## Worker Health Check

| Property | Default | Description |
|---|---|---|
| `kiok.worker.health.check.interval.ms` | `10000` | Interval (ms) at which worker health/liveness is probed. |
| `kiok.worker.health.check.timeout.ms` | `5000` | Per-probe timeout (ms) before a health check counts as a failure. |
| `kiok.worker.health.max.consecutive.failures` | `3` | Consecutive failed health checks after which a worker is treated as unhealthy/dead and its work can be reassigned. |

## Scheduler

| Property | Default | Description |
|---|---|---|
| `kiok.scheduler.tick.interval.ms` | `1000` | Scheduler-loop tick interval (ms): how often pending schedules are evaluated and the run driver advances state. Lower is snappier but uses more CPU. |
| `kiok.scheduler.max.global.concurrency` | `100` | Upper bound on globally concurrent task assignments cluster-wide, independent of per-worker slots. |
| `kiok.scheduler.default.task.timeout.ms` | `1800000` | Default per-task execution timeout (ms; 30 min) applied when a DAG/task sets no timeout of its own. |

## Git Sync

When enabled, the leader periodically pulls DAG repositories so DAG definitions can be managed in git.

| Property | Default | Description |
|---|---|---|
| `kiok.gitsync.enabled` | `false` | When `true`, the leader runs the git-sync loop pulling DAG repositories. |
| `kiok.gitsync.interval.ms` | `60000` | Interval (ms) between git pulls when git sync is enabled. |
| `kiok.gitsync.repos` | *(empty)* | **Optional.** Comma-separated repos, each `<name>|<url>|<branch>|<subpath>`. Credentials, if needed, are looked up in the connection store by `<name>`. |
| `kiok.gitsync.checkout.dir` | `${kiok.base.data.dir}/gitsync` | Local directory where synced repositories are checked out. |

## Job Logs

Per-task logs buffer on the worker and leader, then optionally archive to S3.

| Property | Default | Description |
|---|---|---|
| `kiok.joblog.temp.rocksdb.path` | `${kiok.base.data.dir}/joblog-temp` | RocksDB temp directory on the leader where per-task log bytes buffer before being flushed to S3. |
| `kiok.joblog.flush.interval.ms` | `10000` | Interval (ms) at which the leader flushes buffered log parts to S3 via multipart upload (a final flush also occurs on task completion). |
| `kiok.joblog.s3.endpoint` | *(blank)* | **Optional.** S3 endpoint for archived logs. Leave blank to disable S3 archival. |
| `kiok.joblog.s3.region` | `us-east-1` | **Optional.** S3 region for archived logs. |
| `kiok.joblog.s3.bucket` | `kiok-joblogs` | **Optional.** S3 bucket for archived logs. |
| `kiok.joblog.s3.prefix` | `joblogs` | Key prefix (folder) under the bucket where job logs are written. |
| `kiok.joblog.s3.path.style` | `false` | **Optional.** Use path-style addressing for non-AWS / MinIO-style endpoints. |
| `kiok.joblog.s3.access.key` | *(blank)* | **Optional.** S3 access key for log archival. |
| `kiok.joblog.s3.secret.key` | *(blank)* | **Optional.** S3 secret key for log archival. |
| `kiok.joblog.worker.batch.bytes` | `65536` | Worker-side max in-flight log bytes (64 KiB) accumulated for a task before force-pushing to the leader. |
| `kiok.joblog.worker.batch.interval.ms` | `2000` | Worker-side max time (ms) to hold buffered task log bytes before pushing to the leader even if the byte threshold is not met. |
| `kiok.joblog.kms.key.id` | `joblog-encryption` | KMS key id used to envelope-encrypt archived job logs at rest in S3. |

## Backup

Manual-only backup of metadata/KMS/IAM/connection state to S3, reusing the job-log S3 endpoint/bucket/credentials under a separate prefix.

| Property | Default | Description |
|---|---|---|
| `kiok.backup.enabled` | `false` | Initial default only — the live on/off toggle is persisted in the metadata store and changed at runtime from the admin UI, not by editing this file. |
| `kiok.backup.s3.prefix` | `backups` | Key prefix (folder) under the shared bucket where backup objects are written. |
| `kiok.backup.kms.key.id` | `backup-encryption` | KMS key id used to envelope-encrypt backup objects at rest. |

## Logging

| Property | Default | Description |
|---|---|---|
| `kiok.log.path` | `${user.dir}/logs` | Directory for log output. Also read directly as a `-Dkiok.log.path` system property when configuring logging. |
| `kiok.log.file.name` | `kiok.log` | Application log file name; must match what `logback.xml` expects. |
| `kiok.log.output.name` | *(unset)* | **Optional.** Per-role log file base name (e.g. `master` vs `worker`) so co-located processes write to distinct files. |

## Admin

| Property | Default | Description |
|---|---|---|
| `kiok.admin.token.access.expiry.ms` | `900000` | Lifetime (ms; 15 min) of issued JWT access tokens for the admin API. Shorter is more secure but refreshes more often. |
| `kiok.admin.token.refresh.expiry.ms` | `86400000` | Lifetime (ms; 24 h) of issued JWT refresh tokens — how long a session can be silently renewed before re-login. |
| `kiok.admin.http.max.content.size.bytes` | `10485760` | Maximum admin HTTP request body size (10 MiB). Raise to upload large DAG bundles via HTTP. |
| `kiok.admin.forward.timeout.ms` | `30000` | Per-request HTTP timeout (ms) when a follower master transparently forwards a write to the current leader. Must comfortably exceed the leader's processing time. |

## Metrics Collector

| Property | Default | Description |
|---|---|---|
| `kiok.metrics.collect.interval.seconds` | `10` | Period (seconds) at which the master snapshots local + remote node metrics over the internal protocol. |
| `kiok.metrics.retention.seconds` | `3600` | Retention window (seconds; 1 h) for the in-memory sliding metrics history. Larger keeps more history at more memory. |

## S3 Multipart

| Property | Default | Description |
|---|---|---|
| `kiok.s3.multipart.part.size.bytes` | `8388608` | Part size (bytes; 8 MiB) for S3 multipart uploads used by both job-log archival and backups. Must be ≥ 5 MiB (S3's minimum); smaller values are rejected by S3. |

## Task: HTTP runner

| Property | Default | Description |
|---|---|---|
| `kiok.task.http.connect.timeout.ms` | `10000` | TCP connect timeout (ms) for the HTTP task runner. Bounds only connection establishment; the overall request is still bounded by the DAG/task execution timeout. |

## Python (shell + python task execution)

| Property | Default | Description |
|---|---|---|
| `kiok.python.path` | `python3` | Path to the Python interpreter used to run python tasks and compile python DAGs. May be a bare command on `PATH` or an absolute path. |
| `kiok.python.lib.dir` | `lib/python` | Directory containing the bundled Python SDK used when compiling python DAGs to a DagSpec. |

## Client / SDK properties (`kiok.user.*`)

These carry the caller's credentials/identity and are supplied by **clients** — the Java SDK, Python SDK, and `submit.sh` — at submit time via `-D` system properties or environment variables. They are **not** read from the server-side `kiok.properties` and are listed here for reference only. All are **optional** (supply whichever your authentication method requires).

| Property | Default | Description |
|---|---|---|
| `kiok.user.accessKey` | *(unset)* | **Optional.** Client access key for access-key/secret-key authentication. |
| `kiok.user.secretKey` | *(unset)* | **Optional.** Client secret key paired with the access key. |
| `kiok.user.token` | *(unset)* | **Optional.** Client JWT bearer token. |
| `kiok.user.stsToken` | *(unset)* | **Optional.** Client STS / session token. |
