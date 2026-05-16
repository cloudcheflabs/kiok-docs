# Distributed Architecture

kiok separates the control/coordination plane from the execution plane so the two scale independently.

## Masters (Control Plane)

- Terminate the admin protocol and serve the Admin UI / REST API on a dedicated port.
- Hold and replicate cluster state — DAG definitions, run history, IAM, KMS, connections — in an embedded KMS-encrypted RocksDB store. Only the leader writes; followers receive every change.
- Run the scheduler (cron firing, manual triggers) and the task coordinator (run assignment, progress tracking, driver failover).

## Workers (Execution Plane)

- Drive DAG runs: walk the graph, compute ready tasks, dispatch them, report state.
- Execute task bodies — shell, python, http — and stream stdout/stderr back to the leader.
- Hold no authoritative state. A lost Worker loses only the runs it was driving, and those are reassigned.

## Cluster coordination

- A cluster opens to API traffic only when every Master and at least one Worker is ready. Until then, requests are rejected with `503`. There is no timeout — a cluster started without any Worker simply stays closed.
- The previous leader tends to win re-election after a restart (sticky leadership), so cluster state moves as little as possible.
- All state changes run on the leader. Admin requests that land on a follower are automatically routed to the leader, so a single source of truth is preserved.
- Default `admin/admin` credentials are created only on the very first cluster bootstrap. After that the cluster never recreates defaults — protecting against any local issue silently overwriting your real IAM.

## Two surfaces

- **Admin HTTP** (default 8080): the Admin UI, REST API, and health probes. The only HTTP surface in the cluster.
- **Internal protocol** (default 19999 for Masters, 19998 for Workers): a custom NIO channel carrying all inter-node traffic — state fan-out, run assignment, status, and log streaming. Separated from the admin port so cluster traffic never contends with the API. See [Network &amp; Internal Communication](network.md).
