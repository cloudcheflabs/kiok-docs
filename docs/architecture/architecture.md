# Architecture

kiok is a workflow orchestrator built for high availability and horizontal scale. A cluster is composed of three logical tiers — a control & coordination tier, an execution tier, and a membership tier — each scalable independently.

## kiok Architecture

### Control & Coordination Layer (Masters)

Masters are the "brain" of kiok. They terminate the admin protocol, hold cluster state, and decide *what* should run *when*.

- **Admin HTTP** — the only HTTP surface in the cluster. A Netty server hosts the admin UI, the JWT-authenticated REST API, and the `/healthz` / `/readyz` probes. All other traffic uses the internal protocol.
- **Scheduler** — a leader-only loop that fires DAGs on their cron schedule and accepts manual run triggers. Catchup of missed intervals is opt-in per DAG.
- **Task coordinator** — when a run is due, the leader assigns the whole run to one worker (its *driver*) and tracks progress. If that worker dies mid-run, the coordinator detects it and reassigns the run.
- **Metadata store** — DAG definitions, run history, git-repo configs, and bundle metadata live in an embedded RocksDB store, KMS envelope-encrypted at rest. Only the leader writes; followers and workers receive snapshots.
- **State fan-out** — every change on the leader (DAGs, IAM, KMS, connections) is pushed to followers immediately, and re-synchronised on every restart.

### Execution Layer (Workers)

Workers are the "hands" of the cluster. A worker is both a *driver* and an *executor*.

- **Driver** — the worker assigned a run walks the DAG: it computes which tasks are ready (all upstream dependencies satisfied), dispatches them, and reports task and run state back to the leader.
- **Executor** — the same worker runs the task bodies: shell scripts, inline Python, and HTTP calls. Task stdout/stderr is streamed back to the leader as it is produced.
- Workers hold no authoritative state. Losing a worker loses at most the runs it was driving, and those are reassigned to a survivor.

### Coordination Layer (ZooKeeper)

ZooKeeper provides cluster membership, leader election, and the readiness signal that gates client traffic.

- **Service discovery** — every master and worker registers itself and its readiness state.
- **Sticky leader election** — exactly one master is elected leader; on restart, the previous leader tends to win re-election, so cluster state moves as little as possible.
- **Cluster readiness guarantees**:
    - The admin API accepts requests only after every master and at least one worker is ready. Until then requests are rejected with `503`. There is no timeout — a cluster started without any worker simply stays closed.
    - Default `admin/admin` credentials are created only on the very first bootstrap, never again. A leader whose local state looks empty for any reason refuses to recreate defaults — your real IAM cannot be silently overwritten.

### Architectural Properties

- **Independent scalability** — scheduling/coordination capacity (masters) and execution capacity (workers) scale separately. Add workers to run more tasks in parallel; add masters for HA.
- **Fault tolerance** — worker failure triggers driver failover: the leader reassigns in-flight runs to a healthy worker. Master failure triggers leader re-election; a follower takes over with the fanned-out state already in hand.
- **Single source of truth** — DAGs, run history, IAM, and KMS state always live on the leader. Followers and workers synchronise from the leader on every restart and on every change at runtime.
- **Encrypted at rest** — metadata, KMS keystore, IAM, connection credentials, and job logs are all KMS envelope-encrypted on disk.
