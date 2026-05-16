# Configuration Management

kiok resolves configuration through a priority chain, so the same distribution runs unchanged across laptops, containers, and production hosts.

## Priority order

Highest to lowest:

1. **System properties** — `-D` flags passed on the command line, e.g. `-Dkiok.master.admin.port=8080`.
2. **Environment variables** — the property name uppercased with dots replaced by underscores: `kiok.master.admin.port` → `KIOK_MASTER_ADMIN_PORT`.
3. **Bundled defaults** — the `kiok.properties` defaults shipped inside the distribution.

A value set at a higher level overrides the same key set lower down.

## Variable substitution

Properties may reference other properties with `${...}` — for example the RocksDB stores all derive from one base directory:

```properties
kiok.base.data.dir=./data
kiok.metadata.rocksdb.path=${kiok.base.data.dir}/metadata
```

## Key settings

| Property | Purpose |
|---|---|
| `kiok.zk.serverList` | ZooKeeper connect string |
| `kiok.master.admin.port` | Admin UI / REST API port (default 8080) |
| `kiok.master.internal.port` | Master internal protocol port (default 19999) |
| `kiok.worker.internal.port` | Worker internal protocol port (default 19998) |
| `kiok.worker.task.slots` | Concurrent task slots per Worker (default 8) |
| `kiok.scheduler.tick.interval.ms` | Scheduler evaluation interval |
| `kiok.scheduler.default.task.timeout.ms` | Default per-task timeout |
| `kiok.kms.master.key.env` | Env var holding the master key (`KIOK_MASTER_KEY`) |
| `kiok.gitsync.interval.ms` | Git Sync pull interval |
| `kiok.joblog.s3.*` | Job-log S3 archival destination |
| `kiok.python.path` | Python binary for `python` tasks |

The bundled `kiok.properties` is the full reference — every key is listed there with its default and a comment.
