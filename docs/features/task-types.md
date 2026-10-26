# Task Types

A task is one node in a DAG. kiok runs seven task types (`shell`, `python`, `python_callable`, `http`, `livy`, `trino`, `ontul`); all of them are authored the same way in YAML, Python, or Java.

## `shell`

Runs an inline shell script. The script body is written to a file, made executable, and run by the Worker; stdout and stderr are captured and streamed back.

```yaml
- id: extract
  type: shell
  script: |
    #!/bin/bash
    set -e
    echo "extracting"
```

### One-file shorthand: a bare `.sh` DAG

For the common single-task case, drop a plain `.sh` file under the git repo (or inside a bundle zip) and kiok registers it as a one-task DAG automatically — no YAML / Python / Java wrapper needed. The filename stem (lowercased, non-alphanumeric → `-`) becomes the DAG id; the file body becomes the script of the single task named `run`. Manual-trigger only; if you need a schedule, author the same workload as YAML with a `schedule:` instead.

## `python`

Runs an inline Python script with the Worker's `python3` (`kiok.python.path`). Useful for data manipulation without a shell wrapper.

```yaml
- id: transform
  type: python
  requires: [extract]
  script: |
    print("rows:", sum(range(100)))
```

`python_callable` imports a module on the Worker and calls one function in it
(`config.module`, `config.callable`) — e.g. `module: mypkg.jobs`, `callable: run`.

### Environment of shell and python tasks

A `shell`, `python` or `python_callable` task runs with the run's trigger arguments as
`KIOK_ARG_<NAME>` (upper-cased) and with `KIOK_DAG_ID`, `KIOK_RUN_ID` and `KIOK_TASK_ID` set — see
[Scheduler &amp; Triggers](scheduler.md#manual-triggers).

## `http`

Performs an HTTP call — useful for triggering external systems or webhooks. The URL and options come from the task config:

```yaml
- id: notify
  type: http
  requires: [transform]
  config:
    url: https://example.internal/webhook
    method: POST
    body: '{"status":"done"}'
```

| Config key | Meaning |
|---|---|
| `url` | Request URL (required) |
| `method` | HTTP method; defaults to `GET` |
| `body` | Optional request body |
| `ssl.trustStore` / `ssl.trustStorePassword` / `ssl.insecure` | TLS trust for an HTTPS URL with a self-signed certificate — see [HTTPS endpoints](#https-endpoints-with-a-self-signed-certificate) |
| `expectStatus` | Optional exact status code to require (e.g. `201`). When set, the task succeeds only on that status; otherwise **any 2xx** is a success and anything else fails the task |

The TCP connect timeout is bounded by `kiok.task.http.connect.timeout.ms` (default 10s); the overall request is bounded by the task/DAG timeout.

## `livy`

Submits a Spark batch via Apache Livy's REST API and tails the batch log until it terminates. Unlike `shell` / `python` / `http`, a `livy` task shepherds an **external** job whose identity (the Livy batch id) is recorded on the `TaskRun.externalId` field — which lets a kiok worker restart **resume** polling against the same Spark batch instead of submitting a duplicate.

```yaml
- id: nightly_spark
  type: livy
  config:
    livy.url:  "http://livy:8998"
    livy.file: "s3a://my-bucket/jobs/etl.jar"
    livy.className: "com.example.Etl"
    livy.args: ["--date", "2026-05-25"]
    livy.conf: { spark.sql.shuffle.partitions: "200" }
    livy.driverMemory:   "2g"
    livy.executorMemory: "4g"
    livy.numExecutors:   "8"
```

The full Spark stdout (`INFO TaskSetManager … (N/M)`, the workload's own `print(...)`, `SparkContext is stopping`, …) streams into kiok's per-task log view as it arrives. See the [Spark via Apache Livy](../tutorials/spark-via-livy.md) tutorial for a self-contained `docker compose` walkthrough, the YAML / Python / Java authoring of the same task, the full `livy.*` configuration reference, and a worker-restart resume demo.

| Key | Meaning |
|---|---|
| `livy.ssl.trustStore` | For an HTTPS Livy endpoint / gateway with a self-signed or private-CA certificate: a PEM, JKS or PKCS12 file **on the worker** holding the certificate or CA. Trusted in addition to the JVM's default CAs; the host name is still verified |
| `livy.ssl.trustStorePassword` | Password of a JKS / PKCS12 trust store (not needed for PEM); can be `${conn.<id>.password}` |
| `livy.ssl.trustStoreType` | `PEM` / `JKS` / `PKCS12`; detected when omitted |
| `livy.ssl.insecure` | `"true"` accepts any certificate and host name — development only; default `false` |

```yaml
- id: nightly_spark
  type: livy
  config:
    livy.url:  "https://livy-gw.example:8443"
    livy.file: "s3a://my-bucket/jobs/etl.jar"
    livy.className: "com.example.Etl"
    livy.ssl.trustStore: /etc/kiok/certs/gw.pem
```

See [HTTPS endpoints with a self-signed certificate](#https-endpoints-with-a-self-signed-certificate).

## `trino`

Runs the task's `script` as a SQL statement against any Trino coordinator — or a chango trino gateway protected by HTTP Basic auth. The worker uses the Trino REST `/v1/statement` protocol: POST the SQL, follow `nextUri` until terminal, stream rows + state transitions into the task log. Cancel issues a `DELETE /v1/query/{id}`.

```yaml
- id: daily_merge
  type: trino
  config:
    trino.url:      "https://trino-gw.chango.example/"
    trino.user:     "${conn.trinoGw.username}"
    trino.password: "${conn.trinoGw.password}"
    trino.catalog:  "iceberg"
    trino.schema:   "warehouse"
  script: |
    MERGE INTO warehouse.orders t
    USING (
      SELECT id, customer_id, total
      FROM staging.orders
      WHERE date(updated_at) = DATE '#{ nowMinusFormatted(0, 0, 1, 0, 0, 0, "yyyy-MM-dd") }'
    ) s
    ON t.id = s.id
    WHEN MATCHED THEN UPDATE SET total = s.total
    WHEN NOT MATCHED THEN INSERT (id, customer_id, total) VALUES (s.id, s.customer_id, s.total)
```

| Key | Meaning |
|---|---|
| `trino.url` | Base URL of the coordinator or gateway (required) |
| `trino.user` | `X-Trino-User` header; usually `${conn.<id>.username}` (required) |
| `trino.password` | Basic-auth password — typically `${conn.<id>.password}` for a chango gateway |
| `trino.catalog` / `trino.schema` | Default catalog / schema for the session |
| `trino.source` | `X-Trino-Source` identifier; defaults to `kiok` |
| `trino.pollIntervalMs` | nextUri poll cadence; default 500 |
| `trino.ssl.trustStore` | For an HTTPS gateway with a self-signed or private-CA certificate: a PEM, JKS or PKCS12 file **on the worker** holding the gateway's certificate or CA. Trusted in addition to the JVM's default CAs; the host name is still verified |
| `trino.ssl.trustStorePassword` | Password of a JKS / PKCS12 trust store (not needed for PEM); can be `${conn.<id>.password}` |
| `trino.ssl.trustStoreType` | `PEM` / `JKS` / `PKCS12`; detected when omitted |
| `trino.ssl.insecure` | `"true"` accepts any certificate and host name — development only; default `false` |

The `#{ nowMinusFormatted(...) }` token in the SQL is resolved at run time — see [Date placeholders](date-placeholders.md).

### HTTPS endpoints with a self-signed certificate

Without the `ssl.*` keys a task uses the JVM's default trust store, so a gateway with a self-signed
certificate fails with `PKIX path building failed`; the task log then names the keys to set. Set them per
task instead of `-Djavax.net.ssl.trustStore` on the kiok JVM, which would affect every TLS call in it and
drop the public CAs unless `cacerts` is copied in:

```yaml
- id: daily_merge
  type: trino
  config:
    trino.url:           "https://trino-gw.chango.example:443"
    trino.user:          "${conn.trinoGw.username}"
    trino.password:      "${conn.trinoGw.password}"
    trino.ssl.trustStore: /etc/kiok/certs/trino-gw.pem   # the gateway's certificate, on every worker
```

Every HTTPS task type takes the same four keys under its own prefix — they are not trino-only:

| Type | Keys |
|---|---|
| `trino` | `trino.ssl.trustStore`, `trino.ssl.trustStorePassword`, `trino.ssl.trustStoreType`, `trino.ssl.insecure` |
| `ontul` | `ontul.ssl.trustStore`, `ontul.ssl.trustStorePassword`, `ontul.ssl.trustStoreType`, `ontul.ssl.insecure` |
| `livy`  | `livy.ssl.trustStore`, `livy.ssl.trustStorePassword`, `livy.ssl.trustStoreType`, `livy.ssl.insecure` |
| `http`  | `ssl.trustStore`, `ssl.trustStorePassword`, `ssl.trustStoreType`, `ssl.insecure` (no prefix) |

The SDKs take them too: Python `dag.trino(...)` / `dag.ontul(...)` / `dag.livy(...)` with
`ssl_trust_store=..., ssl_trust_store_password=..., ssl_trust_store_type=..., ssl_insecure=...`; Java
`.trinoTrustStore(path, password)` / `.trinoTrustStoreType("PKCS12")` / `.trinoInsecure(true)` and the
`ontul…` / `livy…` variants (`.ontulTrustStore(...)`, `.livyTrustStore(...)`, …).

#### Set it once for every task

If all your gateways share one CA, set the trust store once on the **workers** instead of in every task
(`conf/kiok.properties`, or the matching environment variables):

```properties
kiok.task.ssl.trust.store=/etc/kiok/certs/company-ca.p12
kiok.task.ssl.trust.store.password=changeit
# kiok.task.ssl.trust.store.type=PKCS12   # detected when empty
```

It applies to `trino`, `ontul`, `livy` and `http` tasks that set neither `<prefix>.ssl.trustStore` nor
`<prefix>.ssl.insecure`. It is added to the JVM's default CAs (public endpoints keep working), host names
are still verified, and a task's own keys win. There is deliberately no worker-wide `insecure`. The
worker checks the file at startup and logs `HTTPS tasks trust … (kiok.task.ssl.trust.store)`, or an error
if it cannot be read; each task log says which trust store it used.

## `ontul`

Submits a job to an [ontul](https://cloudcheflabs.github.io/ontul-docs) cluster via its REST API and tails the job status + log until terminal. Four ontul job shapes are supported via `ontul.jobType`:

- **`BATCH`** (default) — `script` carries the SQL.
- **`STREAMING`** — `script` carries a streaming SQL statement (same body field as `BATCH`).
- **`CLASS`** — `ontul.className` plus a `ontul.deps` array of JAR paths.
- **`PYTHON`** — `ontul.scriptPath` plus optional `ontul.deps`.

!!! tip
    This section is a summary. For the full operator guide — S3-hosted vs locally-staged artifacts, the three authoring forms, the ontul job sources, and the complete config reference — see [Ontul Operator](ontul-operator.md).

```yaml
- id: nightly_rollup
  type: ontul
  config:
    ontul.url:     "https://ontul-master.example:9090"
    ontul.token:   "${secret.ontulToken}"
    ontul.jobType: "BATCH"
  script: |
    INSERT INTO warehouse.daily_rollup
    SELECT date(event_ts), customer_id, sum(amount)
    FROM warehouse.events
    WHERE date(event_ts) = DATE '#{ nowMinusFormatted(0, 0, 1, 0, 0, 0, "yyyy-MM-dd") }'
    GROUP BY 1, 2
```

| Key | Meaning |
|---|---|
| `ontul.ssl.trustStore` | For an HTTPS ontul endpoint / gateway with a self-signed or private-CA certificate: a PEM, JKS or PKCS12 file **on the worker** holding the certificate or CA. Trusted in addition to the JVM's default CAs; the host name is still verified |
| `ontul.ssl.trustStorePassword` | Password of a JKS / PKCS12 trust store (not needed for PEM); can be `${conn.<id>.password}` |
| `ontul.ssl.trustStoreType` | `PEM` / `JKS` / `PKCS12`; detected when omitted |
| `ontul.ssl.insecure` | `"true"` accepts any certificate and host name — development only; default `false` |

```yaml
- id: nightly_rollup
  type: ontul
  config:
    ontul.url:   "https://ontul-gw.example:9443"
    ontul.token: "${secret.ontulToken}"
    ontul.ssl.trustStore: /etc/kiok/certs/gw-trust.p12
    ontul.ssl.trustStorePassword: "${conn.gwTrust.password}"
  script: |
    SELECT 1
```

See [HTTPS endpoints with a self-signed certificate](#https-endpoints-with-a-self-signed-certificate).

### Spark-style S3 staging for CLASS / PYTHON deps

`ontul.deps` and `ontul.scriptPath` accept either pre-existing cluster paths or **local file paths on the kiok worker**. When you set `ontul.depsConnectionId`, the worker uploads the local files to S3 under a content-addressed key (`<sha256>-<filename>`) before submitting — HEAD-first so an unchanged jar costs one HEAD instead of one PUT. The S3 URI is what ontul sees; ontul's worker then downloads it via its own `S3DepFetcher`.

```yaml
- id: enrichment_class
  type: ontul
  config:
    ontul.url:        "https://ontul-master.example:9090"
    ontul.token:      "${secret.ontulToken}"
    ontul.jobType:    "CLASS"
    ontul.className:  "com.example.RollupEnrichment"
    # Mixed local + s3 paths — locals get staged, s3:// passes through.
    ontul.deps:
      - "/home/ci/build/libs/rollup-enrichment-1.0.jar"
      - "s3://shared-deps/common-utils-2.1.jar"
    ontul.depsConnectionId: "ss-staging"     # kiok-side S3 connection for upload
    ontul.depsS3Prefix:     "staging/ontul-deps"
    # The ontul worker needs the same connection id to fetch the s3:// URI.
    # ontul.jobConfig is forwarded verbatim as ontul's job-level `config`.
    ontul.jobConfig:
      ontul.deps.s3.connectionId: "ss-staging"
```

| Key | Meaning |
|---|---|
| `ontul.url` | Ontul master base URL (required) |
| `ontul.token` | Ontul user token (`OTOK…`, sent as `Authorization: Token`); typically `${secret.ontulToken}`. Opaque and long-lived — not a JWT |
| `ontul.jobType` | `BATCH` (default) / `STREAMING` / `CLASS` / `PYTHON` |
| `ontul.jobName` | Display name for the ontul job (defaults to the task id) |
| `ontul.className` | Java entrypoint (CLASS) |
| `ontul.scriptPath` | Python script path (PYTHON) |
| `ontul.deps` | JSON / YAML list of dependency paths — locals get staged when `depsConnectionId` is set |
| `ontul.args` | Map of program args forwarded to the ontul submit body |
| `ontul.jobConfig` | Map forwarded verbatim as ontul's `config` (use it for `ontul.deps.s3.connectionId`) |
| `ontul.depsConnectionId` | kiok ConnectionStore S3 connection id used to upload local deps |
| `ontul.depsS3Prefix` | Key prefix or `s3://other-bucket/prefix` (default `staging/ontul-deps`) |
| `ontul.pollIntervalMs` | status/log poll cadence; default 2000 |

The kiok side handles upload; the ontul side handles download. Both use the same content-addressed key, so re-running an unchanged jar is a no-op on both ends. For the ontul-side download to work, the *same* connection id must exist in the ontul Connection Store with the same S3 endpoint/credentials — see the [ontul docs](https://cloudcheflabs.github.io/ontul-docs/latest/features/backup-restore/) for setting up that store. The `ontul.jobType=CLASS` job ends with kiok task `SUCCESS` only after ontul's worker has successfully fetched the jar and run its `main(...)`.

## Common task fields

| Field | Meaning |
|---|---|
| `id` | Task identifier, unique within the DAG |
| `type` | `shell`, `python`, `python_callable`, `http`, `livy`, `trino`, or `ontul` |
| `script` | Inline body for `shell` / `python` tasks, SQL for `trino`, SQL for `ontul` BATCH/STREAMING |
| `requires` | List of upstream task ids that must succeed first |
| `timeout` | Per-task execution timeout; overrides the DAG / cluster default |
| `config` | Type-specific settings (e.g. `url`/`method`/`body` for `http`; `livy.*` / `trino.*` / `ontul.*` keys) |

## Referencing credentials

Task bodies must never embed secrets. Reference a stored credential indirectly — `${conn.<id>.<key>}` for a connection field, `${secret.<name>}` for a named secret — and kiok resolves it at execution time from the encrypted [connection store](connections.md). The DAG definition stays safe to commit to a public git repository.

## Date placeholders in script and config

The same resolution pass that swaps `${conn...}` / `${secret...}` also evaluates `#{ ... }` date functions — `nowFormatted`, `nowMinusFormatted`, `nowPlusInMillis`, etc. — so a daily-scheduled query targets the right partition on every run. Use these placeholders for run-relative dates **in all three authoring forms** (YAML, Java, Python): a kiok DAG is compiled once and the spec is re-evaluated per run, so a date computed in Java/Python `define()` would freeze at compile time. See [Date placeholders](date-placeholders.md) for the full function list, worked examples, and the one case where a code-computed date is correct.
