# Tutorials

End-to-end walkthroughs that exercise kiok against a real workload. Each tutorial uses concrete commands, real files, and authors **the same DAG three times** — once in YAML, once with the Python SDK, once with the Java SDK — so you can see how the three authoring formats map to the same internal `DagSpec` and end up running identically.

## Available tutorials

- [Spark + Iceberg via SSH-to-Master](spark-iceberg-via-ssh.md) — submit a PySpark client-mode job and a Java uberjar cluster-mode job from a kiok DAG, by `ssh`-ing into a Spark master that lives on a different host. Covers credential injection via the `Conn` helper, the `src/main/{script,python,java}/` source layout, git-sync from a real GitHub repo, and the bundle-upload alternative.
- [Spark via Apache Livy](spark-via-livy.md) — drive Spark batches through Livy's REST API: `POST /batches` to submit, poll state + tail log by batch id, surface the full Spark stdout (TaskSetManager progress, "Pi is roughly …", SparkContext shutdown) into kiok's task-log view. Includes a self-contained `docker compose` stack (kiok + Livy + embedded Spark), the YAML / Python / Java authoring of the same task, and a **worker-restart resume** demo that proves the in-flight Spark job is not abandoned when the kiok worker JVM dies.

## Prerequisites common to every tutorial

- A running kiok cluster — at least one Master and one Worker. Follow [Installation](../installation/installation.md) if you don't have one yet.
- The kiok master's admin endpoint reachable from where you run the `curl` examples below. The cluster's admin port and credentials are described in [Getting Started](../installation/getting-started.md).
- A scratch shell — `bash`, `jq`, and `git` on the operator's machine.

Set these once and reuse for every tutorial:

```bash
export KIOK=http://<kiok-master-host>:18400      # kiok admin endpoint
export KTOK=$(curl -sS -X POST $KIOK/api/v1/auth/login \
  -H 'Content-Type: application/json' \
  -d '{"user":"admin","password":"<your-strong-pw>"}' | jq -r .accessToken)
```
