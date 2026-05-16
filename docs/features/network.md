# Network &amp; Internal Communication

kiok cleanly separates its two kinds of traffic onto two surfaces.

## Admin HTTP

A single HTTP port (default 8080) on the Master serves the Admin UI, the JWT-authenticated REST API, and the `/healthz` / `/readyz` probes. This is the **only** HTTP surface in the cluster — clients, the SDKs, and `submit.sh` all talk to it.

## Internal protocol

All inter-node traffic — leader state fan-out, run assignment, task status, and log streaming — runs over a custom **NIO protocol** on dedicated internal ports (default 19999 for Masters, 19998 for Workers), never over HTTP.

- Lightweight, concurrent request/reply between nodes, designed for low overhead at high fan-out.
- Optionally **KMS-encrypted** end to end (`kiok.kms.encrypt.internal.protocol`), so cluster traffic is protected on untrusted networks.
- Separated from the admin port so cluster coordination never contends with API or UI traffic.

## Tunables

RPC timeouts (`kiok.master.rpc.timeout.ms`, `kiok.internal.rpc.timeout.ms`) and Worker health-check intervals (`kiok.worker.health.*`) are configurable for throughput- or latency-sensitive deployments.
