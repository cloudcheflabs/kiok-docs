# Monitoring &amp; Metrics

kiok surfaces the state of the cluster and of every workflow run through the admin UI.

- **Dashboard** — the number of Masters and Workers, and the last hour of CPU usage and JVM heap used per node (all nodes, masters only or workers only). Run states are on the **Executions** page. See the [Admin UI Guide](admin-ui.md#dashboard).
- **Topology** — the live cluster snapshot: the ZooKeeper ensemble (each server's role, status, version, connections, latency, znode count and zxid, plus the Master's session), the leader Master, every registered Master and Worker, and readiness, with a live log stream per node. See [Cluster Topology](topology.md).
- **Run views** — per run, a graph view (each task coloured by state), a job list (per-task worker, timing, exit code), a timeline and a combined run log — see the [Admin UI Guide](admin-ui.md#execution-detail).
- **Health &amp; readiness endpoints** — `/healthz` (liveness) and `/readyz` (returns 200 only when fully ready) for container probes and external monitoring.
- **Logs** — each role writes a rolling log file under `logs/`; per-task and per-run logs are streamed live and retained — see [Job Logs](job-logs.md).
