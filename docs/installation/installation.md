# Download &amp; Install

This page walks through downloading the kiok distribution and bringing up an evaluation cluster on a single machine. For a guided first-DAG workflow, continue to [Getting Started](getting-started.md). For ongoing operations of an existing cluster, see [Cluster Operations](../operations/operations.md).

## Prerequisites

- **Java 17** — kiok is written in Java and ships as a tarball; Java 17 (or newer) must be on `PATH`.
- **Python 3** — required on every worker to run `python` task bodies, and used by git-sync to compile Python-authored DAGs. `python3` must be on `PATH`.
- **A 32+ character master key** — used to derive the cluster's root encryption key. Set as the environment variable `KIOK_MASTER_KEY` before starting any node.
- **macOS or Linux** — the supplied start scripts are bash and assume a POSIX shell.

## Download the Release Archive

```bash
curl -L -O https://github.com/cloudcheflabs/kiok-pack/releases/download/kiok-archive/kiok-1.0.0.tar.gz
```

Extract the archive and change into the resulting directory:

```bash
tar zxvf kiok-1.0.0.tar.gz
cd kiok-1.0.0
```

The archive ships with the layout below:

| Path | Contents |
|---|---|
| `bin/` | Start/stop scripts for ZooKeeper, Master, and Worker; the `submit.sh` CLI |
| `lib/` | All JARs needed to run the Master and Worker; `lib/python/` holds the kiok Python SDK |
| `conf/` | `jvm.conf` and the embedded-ZooKeeper config under `conf/zk/` |
| `admin-ui/` | Built admin SPA, served by the Master's admin port |
| `data/` | Default data directory (RocksDB stores, git checkouts, job-log buffers) |
| `logs/` | Default log destination |

## Start an Evaluation Cluster

A single-machine evaluation cluster is one ZooKeeper, one Master, and one Worker. Export the master key once, then start the three roles:

```bash
export KIOK_MASTER_KEY="kiok-evaluation-master-key-0123456789"

bin/start-zk.sh
bin/start-master.sh
bin/start-worker.sh
```

> The master key must be at least 32 characters. The **same value** must be supplied to every kiok process (Master or Worker) in a real deployment — losing it means losing access to all encrypted state.

After a few seconds the cluster is ready. The Master listens on:

| Port | Purpose |
|---|---|
| `8080` | Admin UI and REST API — point browsers and the SDKs here |
| `19999` | Internal cluster communication |

The Worker uses internal port `19998`. All inter-node traffic uses the internal ports; only `8080` serves HTTP.

To stop the evaluation cluster, reverse the order:

```bash
bin/stop-worker.sh
bin/stop-master.sh
bin/stop-zk.sh
```

## Running Individual Roles

For a real deployment you start each role separately and point them at an external ZooKeeper. Each script reads the bundled defaults and accepts `-D` overrides on the command line. Configuration resolves in the order **system property (`-D`) → environment variable → bundled defaults**; see [Configuration Management](../features/configuration.md).

### ZooKeeper

```bash
bin/start-zk.sh
```

Production deployments usually point at a separately-managed ensemble. Set the connect string via `kiok.zk.serverList` (or the environment variable `KIOK_ZK_SERVERLIST`).

### Master

```bash
bin/start-master.sh \
  -Dkiok.master.admin.port=8080 \
  -Dkiok.master.internal.port=19999 \
  -Dkiok.zk.serverList=zk-host:2181
```

Run two or more Masters for high availability — point them at the same ZooKeeper and master key. The cluster automatically elects exactly one as leader.

### Worker

```bash
bin/start-worker.sh \
  -Dkiok.worker.internal.port=19998 \
  -Dkiok.worker.task.slots=8 \
  -Dkiok.zk.serverList=zk-host:2181
```

Each Worker advertises a fixed number of task slots (default 8). Add Workers to run more tasks in parallel — see [Worker-Driver Execution](../features/execution.md).

### Stopping individual roles

Each `start-*.sh` writes a PID file under `bin/`, and a matching `stop-*.sh` reads it to shut the process down gracefully. Use these in your service manager (systemd, k8s lifecycle hooks, etc.) instead of sending raw signals. `bin/status.sh` reports which roles are running.

## Verifying the Cluster

Once nodes are up, the Master admin port exposes health and readiness endpoints:

```bash
curl http://localhost:8080/healthz
curl http://localhost:8080/readyz
```

`/healthz` reports liveness; `/readyz` returns `200` only once every Master and at least one Worker is ready. While the cluster is still starting, the REST API rejects requests with `503` — health and login endpoints stay reachable throughout.

For the next step — signing in, registering a DAG, and triggering a run — continue to [Getting Started](getting-started.md).
