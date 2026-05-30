# Ontul Operator

The `ontul` task submits a job to an [ontul](https://cloudcheflabs.github.io/ontul-docs) cluster over its REST API, then tails the job's status and log into kiok's per-task log view until the job reaches a terminal state. One kiok DAG can therefore orchestrate ontul SQL, Java, and Python jobs alongside `shell` / `python` / `http` / `livy` / `trino` tasks, with the same scheduling, retries, dependencies, and credential handling.

This page is the in-depth reference for the operator. For the one-paragraph summary alongside the other task types, see [Task Types](task-types.md#ontul).

## Job shapes

`ontul.jobType` selects one of four shapes ontul accepts on `POST /v1/api/job/submit`:

| `jobType` | Body field kiok fills | Carries |
|---|---|---|
| `BATCH` (default) | `sql` | the task `script` (a SQL statement) |
| `STREAMING` | `sql` | the task `script` (a streaming SQL statement) |
| `CLASS` | `className` + `deps` + `args` | a Java entrypoint in a JAR |
| `PYTHON` | `scriptPath` + `deps` + `args` | a Python entrypoint script |

The worker then polls `GET /v1/api/job/status/{jobId}` and `GET /v1/api/job/{jobId}/log?from=N`, streaming new log lines as they arrive. The task succeeds only when ontul reports `COMPLETED`; `FAILED` or `KILLED` fail the task. Cancelling the kiok task issues `POST /v1/api/job/kill/{jobId}`.

!!! note "Worker-restart resume"
    Like the `livy` and `trino` operators, an `ontul` task records ontul's `jobId` on `TaskRun.externalId`. If the kiok worker driving the task restarts, it **resumes** polling the same ontul job instead of submitting a duplicate. If the ontul master dropped the job (e.g. it too restarted), the resume fails fast rather than polling a 404 forever.

## Where the JAR / Python lives

This is the operator's central decision. `ontul.deps` (CLASS/PYTHON) and `ontul.scriptPath` (PYTHON) each accept **either** an already-remote URI **or** a local path on the kiok worker:

```
s3:// jar (already published)   ─▶  kiok submits the s3:// URI as-is, no upload
local jar + ontul.depsConnectionId ─▶ kiok PUTs it to S3, then submits the s3:// URI
local jar, no depsConnectionId   ─▶  error: nowhere to stage it to
```

A path is treated as **remote** (passed through untouched) when it starts with `s3://`, `s3a://`, `hdfs://`, `gs://`, `http://`, or `https://`. Anything else is a local file.

### A) Artifacts already on S3 (recommended)

When your build/CI already publishes the jar and script to S3, just reference the `s3://` URIs. kiok forwards them verbatim; **`ontul.depsConnectionId` is not set**, and ontul's own deps loader fetches them at submit time.

```yaml
dag:
  id: ontul_s3_jobs
  schedule: "0 4 * * *"
  default_timeout: 1h
tasks:
  # BATCH — SQL is the job body; no artifact.
  - id: rollup_sql
    type: ontul
    timeout: 20m
    config:
      ontul.url:     "https://ontul-master.example:9090"
      ontul.token:   "${secret.ontulToken}"
      ontul.jobType: "BATCH"
    script: |
      INSERT INTO warehouse.daily_rollup
      SELECT date(event_ts) AS day, customer_id, sum(amount) AS total
      FROM warehouse.events
      WHERE date(event_ts) = DATE '#{ nowMinusFormatted(0, 0, 1, 0, 0, 0, "yyyy-MM-dd") }'
      GROUP BY 1, 2

  # CLASS — Java entrypoint; jars hosted on S3, referenced as-is.
  - id: enrichment_class
    type: ontul
    requires: [rollup_sql]
    timeout: 40m
    config:
      ontul.url:       "https://ontul-master.example:9090"
      ontul.token:     "${secret.ontulToken}"
      ontul.jobType:   "CLASS"
      ontul.className: "com.example.RollupEnrichment"
      ontul.deps:
        - "s3://ontul-artifacts/jobs/rollup-enrichment-1.0.jar"
        - "s3://ontul-artifacts/libs/common-utils-2.1.jar"
      ontul.args:
        date:        "#{ nowMinusFormatted(0, 0, 1, 0, 0, 0, \"yyyy-MM-dd\") }"
        targetTable: "warehouse.daily_rollup_enriched"
      ontul.jobConfig:
        ontul.job.driver.mode: "worker"

  # PYTHON — Python entrypoint hosted on S3, referenced as-is.
  - id: feature_python
    type: ontul
    requires: [enrichment_class]
    timeout: 40m
    config:
      ontul.url:        "https://ontul-master.example:9090"
      ontul.token:      "${secret.ontulToken}"
      ontul.jobType:    "PYTHON"
      ontul.scriptPath: "s3://ontul-artifacts/jobs/build_features.py"
      ontul.deps:
        - "s3://ontul-artifacts/libs/feature_lib.zip"
      ontul.args:
        date:        "#{ nowMinusFormatted(0, 0, 1, 0, 0, 0, \"yyyy-MM-dd\") }"
        outputTable: "warehouse.daily_features"
      ontul.jobConfig:
        ontul.job.driver.mode: "worker"
```

### B) Local files staged Spark-style

When the jar/script sits on the kiok worker (a CI checkout, a mounted volume), set `ontul.depsConnectionId` to a kiok [connection store](connections.md) S3 connection. The worker uploads each local file under a content-addressed key (`<prefix>/<sha256>-<filename>`), HEAD-first so an unchanged jar costs one HEAD instead of one PUT, and submits the resulting `s3://` URI. `s3://` paths in the same list still pass through untouched, so mixed lists are fine.

```yaml
- id: enrichment_class
  type: ontul
  config:
    ontul.url:        "https://ontul-master.example:9090"
    ontul.token:      "${secret.ontulToken}"
    ontul.jobType:    "CLASS"
    ontul.className:  "com.example.RollupEnrichment"
    ontul.deps:
      - "/home/ci/build/libs/rollup-enrichment-1.0.jar"   # local → staged
      - "s3://shared-deps/common-utils-2.1.jar"           # remote → passthrough
    ontul.depsConnectionId: "ss-staging"                  # kiok-side S3 connection
    ontul.depsS3Prefix:     "staging/ontul-deps"          # or s3://other-bucket/prefix
    # ontul's worker needs the SAME connection id to fetch the staged s3:// URI:
    ontul.jobConfig:
      ontul.deps.s3.connectionId: "ss-staging"
```

!!! warning "Two stores, one connection id"
    kiok handles the **upload**; ontul handles the **download**. For B), the same connection id (`ss-staging` above) must exist in *both* the kiok connection store and the ontul connection store with the same endpoint/credentials. For A) — artifacts already on S3 — only ontul needs the credentials to read them.

## Same DAG, three authoring forms

A kiok DAG can be written as YAML, Java, or Python; all compile to the same `DagSpec`. The Java/Python SDKs expose typed `ontul…` helpers:

=== "Java SDK"

    ```java
    import com.cloudcheflabs.kiok.sdk.Dag;
    import com.cloudcheflabs.kiok.sdk.KiokDag;
    import java.util.List;
    import java.util.Map;

    public class OntulS3JobsDag implements KiokDag {
        @Override
        public Dag define() {
            Dag dag = new Dag("ontul_s3_jobs")
                    .schedule("0 4 * * *")
                    .defaultTimeoutMs(60 * 60 * 1000L);

            String url = "https://ontul-master.example:9090";
            String token = "${secret.ontulToken}";
            String yesterday = "#{ nowMinusFormatted(0, 0, 1, 0, 0, 0, \"yyyy-MM-dd\") }";

            dag.task("enrichment_class")
               .ontulClass(
                   "com.example.RollupEnrichment",
                   List.of("s3://ontul-artifacts/jobs/rollup-enrichment-1.0.jar",
                           "s3://ontul-artifacts/libs/common-utils-2.1.jar"),
                   Map.of("date", yesterday,
                          "targetTable", "warehouse.daily_rollup_enriched"))
               .ontulUrl(url)
               .ontulToken(token)
               .ontulJobConfig(Map.of("ontul.job.driver.mode", "worker"));

            dag.task("feature_python")
               .ontulPython(
                   "s3://ontul-artifacts/jobs/build_features.py",
                   List.of("s3://ontul-artifacts/libs/feature_lib.zip"),
                   Map.of("date", yesterday,
                          "outputTable", "warehouse.daily_features"))
               .ontulUrl(url)
               .ontulToken(token)
               .requires("enrichment_class");

            return dag;
        }
    }
    ```

=== "Python SDK"

    ```python
    from kiok.dag import Dag

    URL = "https://ontul-master.example:9090"
    TOKEN = "${secret.ontulToken}"
    YESTERDAY = '#{ nowMinusFormatted(0, 0, 1, 0, 0, 0, "yyyy-MM-dd") }'

    dag = Dag("ontul_s3_jobs", schedule="0 4 * * *", default_timeout="1h")

    dag.ontul("enrichment_class", URL, token=TOKEN, job_type="CLASS",
              class_name="com.example.RollupEnrichment",
              deps=["s3://ontul-artifacts/jobs/rollup-enrichment-1.0.jar",
                    "s3://ontul-artifacts/libs/common-utils-2.1.jar"],
              args={"date": YESTERDAY,
                    "targetTable": "warehouse.daily_rollup_enriched"},
              job_config={"ontul.job.driver.mode": "worker"})

    dag.ontul("feature_python", URL, token=TOKEN, job_type="PYTHON",
              script_path="s3://ontul-artifacts/jobs/build_features.py",
              deps=["s3://ontul-artifacts/libs/feature_lib.zip"],
              args={"date": YESTERDAY, "outputTable": "warehouse.daily_features"},
              requires=["enrichment_class"])
    ```

!!! danger "Use `#{ … }` for the run date — even in Java/Python"
    Note the Java and Python examples above embed the **`#{ … }` placeholder string**, not a code-computed date such as `LocalDate.now().minusDays(1)`. A kiok DAG is compiled to a static `DagSpec` **once** (at git-sync / registration); the scheduler then fires runs from that stored spec. Code in `define()` therefore runs at compile time, so a code-computed date freezes to whenever the leader last compiled the source — every daily run would reuse it. Only `#{ … }` is re-expanded by the worker on **each** run. This is kiok's analog of Airflow's `{{ ds }}`. See [Date Placeholders](date-placeholders.md).

## The ontul job sources

The DAG above only references S3 URIs. The actual job sources — built and published to S3 by CI, not by kiok — are ordinary ontul SDK programs. Their `main` / module entrypoint receives the `ontul.args` map as `key=value` arguments.

=== "RollupEnrichment.java"

    ```java
    package com.example;

    import com.cloudcheflabs.ontul.sdk.OntulSession;
    import com.cloudcheflabs.ontul.sdk.Source;
    import java.util.HashMap;
    import java.util.Map;

    public class RollupEnrichment {
        public static void main(String[] args) throws Exception {
            Map<String, String> p = new HashMap<>();
            for (String a : args) {
                String[] kv = a.split("=", 2);
                if (kv.length == 2) p.put(kv[0], kv[1]);
            }
            String day = p.getOrDefault("date", "1970-01-01");
            String target = p.getOrDefault("targetTable", "warehouse.daily_rollup_enriched");

            OntulSession session = OntulSession.builder()
                    .master("ontul-master", 47470).build();
            try {
                String enriched =
                    "SELECT r.day, r.customer_id, r.total, c.c_name AS customer_name, " +
                    "CASE WHEN r.total >= 10000 THEN 'PLATINUM' " +
                    "     WHEN r.total >= 1000  THEN 'GOLD' ELSE 'STANDARD' END AS tier " +
                    "FROM warehouse.daily_rollup r " +
                    "JOIN tpch.tiny.customer c ON c.c_custkey = r.customer_id " +
                    "WHERE r.day = DATE '" + day + "'";
                session.execute("CREATE TABLE IF NOT EXISTS " + target + " AS " + enriched + " LIMIT 0");
                session.execute("INSERT INTO " + target + " " + enriched);
                long rows = session.source(Source.sql(
                        "SELECT * FROM " + target + " WHERE day = DATE '" + day + "'")).count();
                System.out.println("enriched rows for " + day + " = " + rows);
            } finally {
                session.close();
            }
        }
    }
    ```

    Build & publish (CI): `javac -cp ontul-sdk.jar … && jar cf rollup-enrichment-1.0.jar … && aws s3 cp rollup-enrichment-1.0.jar s3://ontul-artifacts/jobs/`

=== "build_features.py"

    ```python
    import sys
    from ontul.session import OntulSession

    def main():
        params = dict(a.split("=", 1) for a in sys.argv[1:] if "=" in a)
        day = params.get("date", "1970-01-01")
        out = params.get("outputTable", "warehouse.daily_features")

        session = OntulSession(host="ontul-master", port=47470)
        try:
            feature_sql = f"""
                SELECT e.customer_id, DATE '{day}' AS feature_date, e.tier,
                       e.total AS spend_today, AVG(h.total) AS spend_7d_avg
                FROM warehouse.daily_rollup_enriched e
                JOIN warehouse.daily_rollup h
                  ON h.customer_id = e.customer_id
                 AND h.day BETWEEN DATE '{day}' - INTERVAL '7' DAY AND DATE '{day}'
                WHERE e.day = DATE '{day}'
                GROUP BY e.customer_id, e.tier, e.total
            """
            session.execute(f"CREATE TABLE IF NOT EXISTS {out} AS {feature_sql} LIMIT 0")
            session.execute(f"INSERT INTO {out} {feature_sql}")
            cnt = session.source(f"SELECT count(*) AS cnt FROM {out} WHERE feature_date = DATE '{day}'")
            print("feature rows:", cnt.column("cnt")[0].as_py() if cnt.num_rows else 0)
        finally:
            session.close()

    if __name__ == "__main__":
        main()
    ```

    Publish (CI): `aws s3 cp build_features.py s3://ontul-artifacts/jobs/`

The packaged distribution ships all of these under `examples/` (`ontul_s3_jobs.yaml`, `ontul_s3_jobs.py`, `OntulS3JobsDag.java`, and `ontul-jobs/{RollupEnrichment.java,build_features.py}`).

## Configuration reference

| Key | Meaning |
|---|---|
| `ontul.url` | Ontul master base URL, e.g. `https://ontul-master:9090` (required) |
| `ontul.token` | Ontul **user token** (the long-lived `OTOK…` value issued with an ontul access key, i.e. `ONTUL_USER_TOKEN`); typically `${secret.ontulToken}`. Sent as `Authorization: Token <token>` — an opaque, non-expiring credential, **not** a short-lived JWT, so it needs no periodic refresh |
| `ontul.jobType` | `BATCH` (default) / `STREAMING` / `CLASS` / `PYTHON` |
| `ontul.jobName` | Display name for the ontul job (defaults to the task id) |
| `ontul.className` | Java entrypoint — required for `CLASS` |
| `ontul.scriptPath` | Python entrypoint path — required for `PYTHON` (local file staged when `depsConnectionId` is set) |
| `ontul.deps` | List of dependency paths — remote URIs pass through; locals staged when `depsConnectionId` is set |
| `ontul.args` | Map of program args; ontul flattens it to `key=value` argv for the entrypoint |
| `ontul.jobConfig` | Map forwarded verbatim as ontul's job-level `config` (e.g. `ontul.job.driver.mode`, `ontul.deps.s3.connectionId`) |
| `ontul.depsConnectionId` | kiok connection-store S3 connection id used to upload local deps. Omit when all paths are already remote |
| `ontul.depsS3Prefix` | Key prefix (default `staging/ontul-deps`) or a full `s3://other-bucket/prefix` URI |
| `ontul.pollIntervalMs` | status/log poll cadence; default 2000 |

## Credentials and date handling

- **Ontul user token** — never inline it. Use `${secret.ontulToken}` (or `${conn.<id>.<key>}`); kiok resolves it from the encrypted [connection store](connections.md) in the worker, just before submit. The stored `DagSpec`, git history, and admin UI Source view never hold the raw value. The value is the ontul user token (`OTOK…`) sent as `Authorization: Token <token>`; obtain it from ontul via `POST /admin/iam/keys` (the `token` field returned alongside `accessKeyId`/`secretAccessKey`).
- **Dates** — the `#{ … }` placeholders in `script`, in every `config` value, and inside the `ontul.args` map are resolved per run. See [Date Placeholders](date-placeholders.md).
