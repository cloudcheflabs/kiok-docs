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
- A password change sent to a follower is applied on the leader and answered only once that follower accepts the new password too. If it cannot catch up, the answer is `502` and says the old password still works on that master, so you never get a success while the old password is still valid there.
- A Master or Worker that stalls longer than its ZooKeeper session timeout (a long GC pause, a frozen VM, a network partition) loses its registration with the session. When it reconnects it registers again and fetches IAM, KMS, connections and metadata from the leader, so it is back in every later state push. Its log records `Re-registered … the previous session expired`.
- Default `admin/admin` credentials are created only on the very first cluster bootstrap. After that the cluster never recreates defaults — protecting against any local issue silently overwriting your real IAM.

## Two surfaces

- **Admin HTTP** (default 8080): the Admin UI, REST API, and health probes. The only HTTP surface in the cluster.
- **Internal protocol** (default 19999 for Masters, 19998 for Workers): a custom NIO channel carrying all inter-node traffic — state fan-out, run assignment, status, and log streaming. Separated from the admin port so cluster traffic never contends with the API. See [Network &amp; Internal Communication](network.md).
