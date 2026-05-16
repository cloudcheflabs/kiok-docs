# Monitoring &amp; Metrics

kiok surfaces the state of the cluster and of every workflow run through the admin UI.

- **Dashboard** — aggregate run metrics: counts by state (success / failed / running), throughput over time, and recent activity.
- **Topology** — the live cluster snapshot: the leader Master, every registered Master and Worker, per-Worker task slots, and readiness.
- **Run views** — per run, a graph view (each task coloured by state), a job list (per-task worker, timing, exit code), and a timeline.
- **Health &amp; readiness endpoints** — `/healthz` (liveness) and `/readyz` (returns 200 only when fully ready) for container probes and external monitoring.
- **Logs** — each role writes a rolling log file under `logs/`; per-task and per-run logs are streamed live and retained — see [Job Logs](job-logs.md).
