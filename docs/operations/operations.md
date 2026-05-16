# Cluster Operations

This page covers running a kiok cluster in steady state: starting and stopping roles, observing the cluster, and the day-two procedures exposed via the admin UI. For first-time installation see [Installation](../installation/installation.md); for the full property reference see [Configuration Management](../features/configuration.md).

## Process Layout

A kiok deployment is composed of three process types:

- **ZooKeeper** — provides cluster coordination, membership, and discovery. The release archive ships an embedded single-node ZooKeeper for evaluation; production deployments use a dedicated ensemble.
- **Master** — terminates the admin protocol, holds and replicates cluster state (DAGs, runs, IAM, KMS), runs the scheduler and the task coordinator. Two or more Masters run for high availability; one is elected leader.
- **Worker** — drives DAG runs and executes task bodies. Add Workers to raise execution capacity.

All three are managed by shell scripts under `bin/` in the release archive.

## Starting and Stopping

### Master

```bash
bin/start-master.sh
bin/stop-master.sh
```

The Master listens on the admin port (default 8080) and an internal cluster port (default 19999). `bin/status.sh` shows which roles are running on the host.

### Worker

```bash
bin/start-worker.sh
bin/stop-worker.sh
```

`stop-worker.sh` drains in-flight tasks before exiting (up to `kiok.worker.shutdown.drain.timeout.seconds`).

### ZooKeeper

```bash
bin/start-zk.sh
bin/stop-zk.sh
```

The bundled embedded ZooKeeper is for evaluation only. In production, point all roles at a managed ensemble via `kiok.zk.serverList`.

## Cluster Bootstrap and Restart

Bootstrap is fully automatic and deterministic:

1. The cluster elects a leader from the Master tier. After a restart, the previous leader tends to win re-election (sticky leadership) so cluster state moves as little as possible.
2. The leader initialises its own state. Default `admin/admin` credentials are created **only on the very first cluster bootstrap** — never again, even if the leader's local state somehow looks empty.
3. Followers and Workers wait until the leader is ready, then synchronise DAGs, IAM, and KMS from the leader (the single source of truth) and report themselves ready.
4. The leader waits until every Master is ready **and at least one Worker is registered and ready**, then opens the cluster to API traffic.

Until the cluster is open, REST API requests are rejected with `503`. The wait has no timeout — a cluster brought up without any Worker simply stays closed until one comes up. Health and login endpoints stay reachable throughout so operators can probe and sign in.

If the leader restarts, surviving followers re-elect (with sticky-leader bias) and the cycle repeats. Clients never see a half-initialised cluster.

### Runtime synchronisation

After bootstrap, every state change runs on the leader and is pushed to all other nodes immediately. Admin requests that mutate state (DAG register/delete, IAM, KMS, connections, git-repo config) automatically route to the leader if they land on a follower.

## Observability

### Admin UI

The admin port serves the UI at `/`. The **Settings → Topology** page shows the cluster snapshot — leader identity, registered Masters and Workers, per-worker task slots, and readiness. The **Dashboard** aggregates run metrics.

### Health & Readiness

- `/healthz` — liveness; returns 200 while the process is up.
- `/readyz` — returns 200 only when the node is fully ready, 503 otherwise. Suitable for container readiness probes.

### Metrics & Logs

The admin UI surfaces aggregated run and cluster metrics; see [Monitoring &amp; Metrics](../features/monitoring.md). Each role writes a rolling log file under `logs/`. Per-task and per-run logs are streamed live in the UI and archived — see [Job Logs](../features/job-logs.md).

## Day-Two Procedures

### DAG sources

DAGs reach the cluster three ways, all manageable from the admin UI:

- **Manual** — register a YAML/Python/Java DAG directly.
- **Git Sync** — the leader periodically pulls DAGs from configured git repositories. See [Git Sync](../features/git-sync.md).
- **Bundles** — upload a zip of DAG definitions for air-gapped clusters. See [DAG Bundles](../features/bundles.md).

### Backup & Restore

Enable **Backup** from the admin UI to ship cluster state — metadata (DAGs, runs, bundles, git configs), KMS, IAM, connections, and job logs — to an S3 target. Restore re-imports a chosen snapshot. See [Backup &amp; Restore](../features/backup.md).

### Driver failover

If a Worker driving a run dies, the leader's task coordinator detects the lost worker and reassigns the run to a healthy Worker. No operator action is required — see [Worker-Driver Execution](../features/execution.md).

## Scaling

- **Adding a Master** — start a new node pointing at the same ZooKeeper and master key. It joins as a follower, synchronises state from the leader, and is immediately available to serve the API and take over on leader loss. No maintenance window required.
- **Adding a Worker** — start a new Worker pointing at the same ZooKeeper. It registers its task slots and immediately becomes eligible to drive new runs and execute tasks.
- **Removing a node** — stop the process. A removed Worker's in-flight runs are reassigned by driver failover; a removed Master triggers re-election if it was the leader.

## Upgrades

kiok supports rolling upgrades for compatible versions. The recommended sequence:

1. Upgrade Workers one at a time, draining each before stopping it.
2. Upgrade Master followers one at a time.
3. Upgrade the leader last; it steps down, a follower takes over, and the upgraded process rejoins as a follower.

Across the upgrade, the cluster briefly closes during each leader transition; clients see short `503` windows, never inconsistent state.
