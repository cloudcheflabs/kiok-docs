# Spark + Iceberg via SSH-to-Master

This tutorial drives a kiok cluster end-to-end against a real workload:

- A **PySpark** job in **client deploy mode** that writes 3 rows into an Iceberg table through a Polaris REST catalog backed by S3-compatible storage.
- A **Java uberjar** in **cluster deploy mode** that does the same write into a different table.

Both submits run on a Spark cluster that lives on a **different host** from any kiok Worker. The kiok Worker does not carry a `SPARK_HOME` of its own — it `ssh`es into the Spark master, runs `spark-submit` there, and streams the result back. Every credential the job touches (S3 access key, Polaris OAuth secret) is pulled at task-exec time from a kiok [Connection](../features/connections.md) — the DAG source itself stays free of secrets and is safe to commit to a public repo.

You will author the same DAG three ways and verify all three behave identically:

| Format | File | DAG name |
|---|---|---|
| YAML | `src/main/script/spark_iceberg_dag.yaml` | `spark-iceberg-yaml` |
| Python SDK | `src/main/python/spark_iceberg_dag.py` | `spark-iceberg-python` |
| Java SDK | `src/main/java/com/cloudcheflabs/dags/SparkIcebergJavaDag.java` | `spark-iceberg-java` |

Then you will deploy the same source tree two ways — by [Git Sync](../features/git-sync.md) and by a [Bundle](../features/bundles.md) zip upload — and confirm both paths register the same three DAGs.

## What you need before starting

- A kiok cluster — Master + at least one Worker. `$KIOK` and `$KTOK` from [Tutorials index](index.md#prerequisites-common-to-every-tutorial) work.
- A Spark **standalone** cluster reachable from the kiok Worker host. The tutorial assumes:
  - Spark master URL: `spark://<spark-host>:7077` (use your real port).
  - Spark distribution unpacked at `/opt/spark` on the Spark master, with `bin/spark-submit` runnable as some Unix user (we use the user `spark`).
- An S3-compatible object store and a pre-existing Iceberg catalog. Any AWS S3 or compatible (MinIO, Ceph RGW, ShannonStore, …) works. You need:
  - S3 endpoint URL, region, access key, secret key.
  - An Iceberg [REST catalog](https://iceberg.apache.org/concepts/catalog/#rest-catalog) URL + OAuth client credentials.
- A namespace and a Spark user that is allowed to `CREATE TABLE` / `INSERT` against tables under it. The tutorial uses `iceberg.test.*`.
- One pre-built Java uberjar uploaded to S3 (you'll only use it for the cluster-mode task). Class: `com.cloudcheflabs.spark.IcebergSmoke`. Object location: `s3a://uberjar-upload/iceberg-smoke-java.jar`.

The cluster topology this tutorial assumes:

```mermaid
flowchart LR
  subgraph kiok-host[kiok Worker host]
    kw[kiok Worker]
  end
  subgraph spark-host[Spark master host]
    sm[Spark master]
    sw[Spark worker]
    bin[/opt/spark/bin/spark-submit]
  end
  s3[(S3 bucket<br/>warehouse + pyspark scripts + uberjar)]
  cat[Iceberg REST catalog]
  kw -- ssh --> spark-host
  bin --> sm --> sw
  sm -. read .-> s3
  sw -. read/write .-> s3
  sm -. table metadata .-> cat
  cat -. data files .-> s3
```

## Step 1 — Passwordless SSH from kiok Worker → Spark master

The kiok Worker shell-tasks will run `ssh spark@<spark-host>` with no password and no key argument — so set up a one-shot ed25519 key now.

```bash
# On the kiok Worker host, as the user kiok runs as (usually `kiok`):
sudo -u kiok ssh-keygen -t ed25519 -N "" -f /home/kiok/.ssh/id_ed25519 -C "kiok@worker"
sudo -u kiok cat /home/kiok/.ssh/id_ed25519.pub

# On the Spark master host, append that public key to spark's authorized_keys:
sudo mkdir -p /home/spark/.ssh && sudo chmod 700 /home/spark/.ssh
echo '<paste pubkey here>' | sudo tee -a /home/spark/.ssh/authorized_keys
sudo chmod 600 /home/spark/.ssh/authorized_keys
sudo chown -R spark:spark /home/spark/.ssh

# Smoke test (from the kiok Worker host):
sudo -u kiok ssh -o StrictHostKeyChecking=no spark@<spark-host> hostname
# → <spark-host>
```

The kiok DAG script uses these SSH options so an unknown-host first connection does not stall the run:

```bash
ssh -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null \
    -o LogLevel=ERROR spark@<spark-host> "<remote command>"
```

## Step 2 — Stash credentials in kiok Connections

Two Connections cover everything the DAG references:

```bash
# S3 (lakehouse warehouse + the pre-built uberjar live here):
curl -sS -X POST $KIOK/api/v1/connections \
  -H "Authorization: Bearer $KTOK" -H 'Content-Type: application/json' \
  -d '{
    "connectionId": "s3-shannon",
    "type": "S3",
    "description": "Object storage for the lakehouse warehouse + uberjar",
    "properties": {
      "endpoint":  "http://<s3-host>:8080",
      "region":    "us-east-1",
      "accessKey": "<S3 access key>",
      "secretKey": "<S3 secret key>",
      "pathStyleAccess": "true"
    }
  }'

# Polaris REST catalog (OAuth 2.0 client credentials):
curl -sS -X POST $KIOK/api/v1/connections \
  -H "Authorization: Bearer $KTOK" -H 'Content-Type: application/json' \
  -d '{
    "connectionId": "polaris-rest",
    "type": "GENERIC",
    "description": "Polaris REST catalog (Iceberg)",
    "properties": {
      "uri":              "http://<polaris-host>:30000/api/catalog",
      "warehouse":        "lakehouse",
      "oauthCredential":  "<client-id>:<client-secret>",
      "scope":            "PRINCIPAL_ROLE:ALL"
    }
  }'
```

Both stores are KMS-encrypted at rest — the credentials never appear in any DAG file. Inside a task script you reference them like this:

- YAML: `${conn.s3-shannon.accessKey}`
- Python (SDK): `conn('s3-shannon', 'accessKey')`
- Java (SDK): `Conn.ref("s3-shannon", "accessKey")`

All three produce the literal `${conn.s3-shannon.accessKey}` reference string, which the Worker's `ConnectionResolver` substitutes with the real value just before exec.

## Step 3 — Pre-upload the PySpark script to S3

For both client-mode and cluster-mode submits we want `spark-submit` to **fetch** the workload from S3 rather than read a local path — the same way it fetches the uberjar. Stage the PySpark body once:

```python
# /tmp/yaml_pyspark.py — the workload (used by the YAML DAG; copy to
# py_pyspark.py and java_pyspark.py for the other two DAGs, changing only
# the table name so each DAG writes to a distinct iceberg.test.* table).
from pyspark.sql import SparkSession
import os

C, N, T = "iceberg", "test", "yaml_dag_smoke"
spark = (SparkSession.builder.appName("kiok-yaml-pyspark")
    .config(f"spark.sql.catalog.{C}",              "org.apache.iceberg.spark.SparkCatalog")
    .config(f"spark.sql.catalog.{C}.catalog-impl", "org.apache.iceberg.rest.RESTCatalog")
    .config(f"spark.sql.catalog.{C}.uri",          os.environ["POLARIS_URI"])
    .config(f"spark.sql.catalog.{C}.warehouse",    "lakehouse")
    .config(f"spark.sql.catalog.{C}.credential",   os.environ["POLARIS_CRED"])
    .config(f"spark.sql.catalog.{C}.scope",        "PRINCIPAL_ROLE:ALL")
    .config(f"spark.sql.catalog.{C}.io-impl",      "org.apache.iceberg.aws.s3.S3FileIO")
    .config(f"spark.sql.catalog.{C}.s3.endpoint",         os.environ["S3_ENDPOINT"])
    .config(f"spark.sql.catalog.{C}.s3.access-key-id",    os.environ["S3_AK"])
    .config(f"spark.sql.catalog.{C}.s3.secret-access-key",os.environ["S3_SK"])
    .config(f"spark.sql.catalog.{C}.s3.path-style-access","true")
    .config(f"spark.sql.catalog.{C}.s3.region",           "us-east-1")
    .config("spark.sql.defaultCatalog", C).getOrCreate())

spark.sql(f"CREATE NAMESPACE IF NOT EXISTS {C}.{N}")
spark.sql(f"DROP TABLE IF EXISTS {C}.{N}.{T}")
spark.sql(f"CREATE TABLE {C}.{N}.{T} (id INT, label STRING) USING iceberg")
spark.sql(f"INSERT INTO {C}.{N}.{T} VALUES (1, 'yaml-1'), (2, 'yaml-2'), (3, 'yaml-3')")
n = spark.sql(f"SELECT COUNT(*) AS n FROM {C}.{N}.{T}").collect()[0]["n"]
print(f"=== yaml-dag-pyspark wrote {n} rows ===")
spark.stop()
```

Upload it once with whatever S3 client you have. A tiny PySpark uploader (run anywhere with the Spark distribution available, including the Spark master itself) does the job without installing a separate CLI:

```python
# /tmp/s3_upload.py
from pyspark.sql import SparkSession
spark = SparkSession.builder.appName("s3-uploader").getOrCreate()
hconf = spark.sparkContext._jsc.hadoopConfiguration()
jvm   = spark._jvm
URI, Path, FS = jvm.java.net.URI, jvm.org.apache.hadoop.fs.Path, jvm.org.apache.hadoop.fs.FileSystem
fs = FS.get(URI("s3a://pyspark-scripts/"), hconf)
fs.mkdirs(Path("s3a://pyspark-scripts/"))
for name in ("yaml_pyspark.py", "py_pyspark.py", "java_pyspark.py"):
    src = Path("file:///tmp/" + name)
    dst = Path("s3a://pyspark-scripts/" + name)
    if fs.exists(dst): fs.delete(dst, False)
    fs.copyFromLocalFile(False, True, src, dst)
    print("uploaded s3a://pyspark-scripts/" + name)
spark.stop()
```

```bash
# Spark master's spark-defaults.conf already carries S3 endpoint + AK/SK
# for the eventLog directory, so the uploader needs no extra config:
sudo -u spark $SPARK_HOME/bin/spark-submit --master 'local[2]' \
  --conf spark.eventLog.enabled=false /tmp/s3_upload.py
# → uploaded s3a://pyspark-scripts/yaml_pyspark.py
# → uploaded s3a://pyspark-scripts/py_pyspark.py
# → uploaded s3a://pyspark-scripts/java_pyspark.py
```

## Step 4 — Build the DAG repository

Lay out a repo with the recommended source-tree convention (`src/main/{script,python,java}/`). The repo carries **source only** — kiok's leader compiles every `.java` in-process at sync time, so there is no build step on the operator side.

```
chango-dags/
└── src
    └── main
        ├── script
        │   └── spark_iceberg_dag.yaml
        ├── python
        │   └── spark_iceberg_dag.py
        └── java
            └── com/cloudcheflabs/dags
                └── SparkIcebergJavaDag.java
```

### 4a — The YAML DAG

`src/main/script/spark_iceberg_dag.yaml`:

```yaml
# Kiok DAG (YAML) — submit a PySpark client-mode write and a Java uberjar
# cluster-mode write to the spark cluster. The kiok worker doesn't need a
# local SPARK_HOME — each task SSHes (passwordless, kiok@worker → spark@master)
# into the spark master and runs spark-submit there. Credentials only via
# ${conn.<id>.<key>} refs the worker's ConnectionResolver substitutes before
# bash runs — this YAML never carries a plaintext secret.
dag:
  id: spark-iceberg-yaml
  name: spark-iceberg-yaml
  catchup: false
  defaultTimeoutMs: 600000

tasks:
  - id: pyspark_client_mode_write
    type: shell
    script: |
      #!/bin/bash
      set -euo pipefail
      SSH="ssh -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null -o LogLevel=ERROR spark@<spark-host>"
      $SSH "
        export JAVA_HOME=/opt/openlogic-openjdk-17.0.7+7-linux-x64
        export POLARIS_URI='http://<polaris-host>:30000/api/catalog'
        export POLARIS_CRED='${conn.polaris-rest.oauthCredential}'
        export S3_ENDPOINT='${conn.s3-shannon.endpoint}'
        export S3_AK='${conn.s3-shannon.accessKey}'
        export S3_SK='${conn.s3-shannon.secretKey}'
        /opt/spark/bin/spark-submit \
          --master spark://<spark-host>:7077 \
          --deploy-mode client --conf spark.eventLog.enabled=false \
          --conf spark.hadoop.fs.s3a.endpoint='${conn.s3-shannon.endpoint}' \
          --conf spark.hadoop.fs.s3a.access.key='${conn.s3-shannon.accessKey}' \
          --conf spark.hadoop.fs.s3a.secret.key='${conn.s3-shannon.secretKey}' \
          --conf spark.hadoop.fs.s3a.path.style.access=true \
          s3a://pyspark-scripts/yaml_pyspark.py
      "

  - id: java_spark_cluster_mode_write
    type: shell
    requires: [pyspark_client_mode_write]
    script: |
      #!/bin/bash
      set -euo pipefail
      SSH="ssh -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null -o LogLevel=ERROR spark@<spark-host>"
      $SSH "
        export JAVA_HOME=/opt/openlogic-openjdk-17.0.7+7-linux-x64
        /opt/spark/bin/spark-submit \
          --master spark://<spark-host>:7077 \
          --deploy-mode cluster --conf spark.eventLog.enabled=false \
          --class com.cloudcheflabs.spark.IcebergSmoke \
          --conf spark.hadoop.fs.s3a.endpoint='${conn.s3-shannon.endpoint}' \
          --conf spark.hadoop.fs.s3a.access.key='${conn.s3-shannon.accessKey}' \
          --conf spark.hadoop.fs.s3a.secret.key='${conn.s3-shannon.secretKey}' \
          --conf spark.hadoop.fs.s3a.path.style.access=true \
          s3a://uberjar-upload/iceberg-smoke-java.jar
      "
```

Key shape:

- Two tasks. The cluster-mode task depends on the client-mode task (`requires: [pyspark_client_mode_write]`).
- Each task is a single `shell` task that runs one `ssh ... "<remote spark-submit>"`. The kiok Worker never directly invokes `spark-submit`.
- `${conn.X.Y}` reference strings are substituted by `ConnectionResolver` **before** bash parses the line — so the resolved (decrypted) credential exists only inside the (encrypted) SSH stream and the remote env vars; the YAML file itself stays plaintext-safe.

### 4b — The Python SDK DAG

`src/main/python/spark_iceberg_dag.py`:

```python
"""Kiok DAG (Python SDK) — same workload as the YAML twin.
One ssh-then-spark-submit per task; pyspark script + java uberjar both live
in S3. Credentials only via conn(...).
"""
from kiok import Dag, conn

SSH = ('ssh -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null '
       '-o LogLevel=ERROR spark@<spark-host>')

# Pre-build the credential refs so the f-string inside the script body stays
# regex-friendly (the inner conn() call uses its own single quotes).
POLARIS_CRED = conn('polaris-rest', 'oauthCredential')
S3_ENDPOINT  = conn('s3-shannon',   'endpoint')
S3_AK        = conn('s3-shannon',   'accessKey')
S3_SK        = conn('s3-shannon',   'secretKey')

CLIENT_SCRIPT = f"""#!/bin/bash
set -euo pipefail
SSH="{SSH}"
$SSH "
  export JAVA_HOME=/opt/openlogic-openjdk-17.0.7+7-linux-x64
  export POLARIS_URI='http://<polaris-host>:30000/api/catalog'
  export POLARIS_CRED='{POLARIS_CRED}'
  export S3_ENDPOINT='{S3_ENDPOINT}'
  export S3_AK='{S3_AK}'
  export S3_SK='{S3_SK}'
  /opt/spark/bin/spark-submit \\
    --master spark://<spark-host>:7077 \\
    --deploy-mode client --conf spark.eventLog.enabled=false \\
    --conf spark.hadoop.fs.s3a.endpoint='{S3_ENDPOINT}' \\
    --conf spark.hadoop.fs.s3a.access.key='{S3_AK}' \\
    --conf spark.hadoop.fs.s3a.secret.key='{S3_SK}' \\
    --conf spark.hadoop.fs.s3a.path.style.access=true \\
    s3a://pyspark-scripts/py_pyspark.py
"
"""

CLUSTER_SCRIPT = f"""#!/bin/bash
set -euo pipefail
SSH="{SSH}"
$SSH "
  export JAVA_HOME=/opt/openlogic-openjdk-17.0.7+7-linux-x64
  /opt/spark/bin/spark-submit \\
    --master spark://<spark-host>:7077 \\
    --deploy-mode cluster --conf spark.eventLog.enabled=false \\
    --class com.cloudcheflabs.spark.IcebergSmoke \\
    --conf spark.hadoop.fs.s3a.endpoint='{S3_ENDPOINT}' \\
    --conf spark.hadoop.fs.s3a.access.key='{S3_AK}' \\
    --conf spark.hadoop.fs.s3a.secret.key='{S3_SK}' \\
    --conf spark.hadoop.fs.s3a.path.style.access=true \\
    s3a://uberjar-upload/iceberg-smoke-java.jar
"
"""

dag = Dag("spark-iceberg-python", catchup=False, default_timeout="10m")
dag.task("pyspark_client_mode_write",     script=CLIENT_SCRIPT)
dag.task("java_spark_cluster_mode_write", script=CLUSTER_SCRIPT,
         requires=["pyspark_client_mode_write"])
```

Things to notice:

- The module-level `dag` variable is automatically picked up by `kiok.compile`. To define multiple DAGs in one file, just assign more module-level `Dag` instances.
- `conn('s3-shannon', 'accessKey')` returns the literal string `${conn.s3-shannon.accessKey}` — the f-string just splices the reference into the script body. The actual credential is resolved at task-exec time, not at compile time.

### 4c — The Java SDK DAG

`src/main/java/com/cloudcheflabs/dags/SparkIcebergJavaDag.java`:

```java
package com.cloudcheflabs.dags;

import com.cloudcheflabs.kiok.sdk.Conn;
import com.cloudcheflabs.kiok.sdk.Dag;
import com.cloudcheflabs.kiok.sdk.KiokDag;

public class SparkIcebergJavaDag implements KiokDag {

    private static final String SSH =
            "ssh -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null "
          + "-o LogLevel=ERROR spark@<spark-host>";

    private static final String POLARIS_CRED = Conn.ref("polaris-rest", "oauthCredential");
    private static final String S3_ENDPOINT  = Conn.ref("s3-shannon",   "endpoint");
    private static final String S3_AK        = Conn.ref("s3-shannon",   "accessKey");
    private static final String S3_SK        = Conn.ref("s3-shannon",   "secretKey");

    private static final String CLIENT_SCRIPT = """
            #!/bin/bash
            set -euo pipefail
            SSH="%s"
            $SSH "
              export JAVA_HOME=/opt/openlogic-openjdk-17.0.7+7-linux-x64
              export POLARIS_URI='http://<polaris-host>:30000/api/catalog'
              export POLARIS_CRED='%s'
              export S3_ENDPOINT='%s'
              export S3_AK='%s'
              export S3_SK='%s'
              /opt/spark/bin/spark-submit \\
                --master spark://<spark-host>:7077 \\
                --deploy-mode client --conf spark.eventLog.enabled=false \\
                --conf spark.hadoop.fs.s3a.endpoint='%s' \\
                --conf spark.hadoop.fs.s3a.access.key='%s' \\
                --conf spark.hadoop.fs.s3a.secret.key='%s' \\
                --conf spark.hadoop.fs.s3a.path.style.access=true \\
                s3a://pyspark-scripts/java_pyspark.py
            "
            """.formatted(SSH,
                    POLARIS_CRED, S3_ENDPOINT, S3_AK, S3_SK,
                    S3_ENDPOINT, S3_AK, S3_SK);

    private static final String CLUSTER_SCRIPT = """
            #!/bin/bash
            set -euo pipefail
            SSH="%s"
            $SSH "
              export JAVA_HOME=/opt/openlogic-openjdk-17.0.7+7-linux-x64
              /opt/spark/bin/spark-submit \\
                --master spark://<spark-host>:7077 \\
                --deploy-mode cluster --conf spark.eventLog.enabled=false \\
                --class com.cloudcheflabs.spark.IcebergSmoke \\
                --conf spark.hadoop.fs.s3a.endpoint='%s' \\
                --conf spark.hadoop.fs.s3a.access.key='%s' \\
                --conf spark.hadoop.fs.s3a.secret.key='%s' \\
                --conf spark.hadoop.fs.s3a.path.style.access=true \\
                s3a://uberjar-upload/iceberg-smoke-java.jar
            "
            """.formatted(SSH, S3_ENDPOINT, S3_AK, S3_SK);

    @Override
    public Dag define() {
        Dag dag = new Dag("spark-iceberg-java")
                .catchup(false)
                .defaultTimeoutMs(600_000);
        dag.task("pyspark_client_mode_write").shell(CLIENT_SCRIPT);
        dag.task("java_spark_cluster_mode_write")
           .requires("pyspark_client_mode_write")
           .shell(CLUSTER_SCRIPT);
        return dag;
    }
}
```

There is **no `pom.xml`, no `build.gradle`, no `mvn package`**. The file goes into the repo as a plain `.java` source. The kiok leader runs `javax.tools.JavaCompiler` in-process at sync time with kiok-sdk on the master's classpath, so the `Conn`, `Dag`, and `KiokDag` imports resolve automatically. The original source text is what the admin UI's **Source** tab shows.

## Step 5 — Deploy via Git Sync

Push the source tree to any git repository the kiok master can reach.

```bash
cd ~/work/chango-dags
git init -b main && git add -A
git commit -m "spark-iceberg DAGs (yaml/python/java)"
git remote add origin https://github.com/<your-org>/<your-repo>.git
git push -u origin main
```

Register the repo with kiok:

```bash
curl -sS -X POST $KIOK/api/v1/admin/gitsync/repos \
  -H "Authorization: Bearer $KTOK" -H 'Content-Type: application/json' \
  -d '{
    "name":     "chango-dags",
    "url":      "https://github.com/<your-org>/<your-repo>.git",
    "ref":      "main",
    "tag":      false,
    "subpath":  ""
  }'
# → {"saved":"chango-dags"}
```

The leader pulls the repo on the next sync tick. Trigger a sync now so you don't have to wait:

```bash
curl -sS -X POST $KIOK/api/v1/admin/gitsync/sync \
  -H "Authorization: Bearer $KTOK"
# → {"status":"sync triggered"}
```

Confirm all three DAGs registered:

```bash
curl -sS -H "Authorization: Bearer $KTOK" $KIOK/api/v1/dags \
  | jq '[.[] | select(.origin == "git") | {id, name, sourcePath}]'
```

Expected:

```json
[
  {"id":"chango-dags_src_main_script_spark_iceberg_dag",
   "name":"spark-iceberg-yaml",
   "sourcePath":"src/main/script/spark_iceberg_dag.yaml"},
  {"id":"chango-dags_src_main_python_spark_iceberg_dag.py_spark-iceberg-python",
   "name":"spark-iceberg-python",
   "sourcePath":"src/main/python/spark_iceberg_dag.py"},
  {"id":"chango-dags_src_main_java_src/main/java/com/cloudcheflabs/dags/SparkIcebergJavaDag.java::spark-iceberg-java",
   "name":"spark-iceberg-java",
   "sourcePath":"src/main/java/com/cloudcheflabs/dags/SparkIcebergJavaDag.java"}]
```

Open one of them in the admin UI's **DAG → Source** tab to verify the **Original source** matches the file you pushed and the **Compiled DAG spec** shows the two tasks with the right `requires`. For the Java DAG the original source is the `.java` text you wrote — not a "compiled from N files" marker.

## Step 6 — Deploy via a Bundle zip (alternative)

For an air-gapped cluster, package the same source tree as a zip and upload it. The compile flow is identical to git-sync, just driven by a one-shot HTTP upload.

```bash
cd ~/work/chango-dags
zip -r /tmp/spark-iceberg-bundle.zip src

curl -sS -X POST "$KIOK/api/v1/admin/bundles/spark-iceberg-bundle" \
  -H "Authorization: Bearer $KTOK" \
  -H 'Content-Type: application/zip' \
  --data-binary @/tmp/spark-iceberg-bundle.zip
# → {"uploaded":"spark-iceberg-bundle"}
```

Verify:

```bash
curl -sS -H "Authorization: Bearer $KTOK" $KIOK/api/v1/dags \
  | jq '[.[] | select(.origin == "bundle") | {id, name, sourcePath}]'
```

You should see three more DAGs, prefixed with `spark-iceberg-bundle_…`. Re-uploading the same bundle name overwrites the previous DAGs in-place but preserves their run history. Deleting the bundle removes the DAGs but again leaves the history intact — see [DAG Bundles](../features/bundles.md) for the durability story.

## Step 7 — Run the DAGs and read the logs

Trigger any of the three DAGs:

```bash
curl -sS -X POST \
  "$KIOK/api/v1/dags/chango-dags_src_main_script_spark_iceberg_dag/runs" \
  -H "Authorization: Bearer $KTOK" -H 'Content-Type: application/json' -d '{}'
# → {"runId":"chango-dags_src_main_script_spark_iceberg_dag-<ts>-N", ...}
```

Poll until the run reaches a terminal state:

```bash
RID=<runId from above>
for i in $(seq 1 40); do
  R=$(curl -sS -H "Authorization: Bearer $KTOK" $KIOK/api/v1/runs/$RID)
  S=$(echo "$R" | jq -r .state)
  T=$(echo "$R" | jq -c '.tasks | with_entries(.value |= .state)')
  echo "[$i] $S $T"
  [ "$S" = "SUCCESS" ] || [ "$S" = "FAILED" ] && break
  sleep 10
done
```

Successful run:

```
[1] PENDING {}
[2] RUNNING {"pyspark_client_mode_write":"RUNNING","java_spark_cluster_mode_write":"PENDING"}
[3] RUNNING {"pyspark_client_mode_write":"SUCCESS","java_spark_cluster_mode_write":"RUNNING"}
[4] SUCCESS {"pyspark_client_mode_write":"SUCCESS","java_spark_cluster_mode_write":"SUCCESS"}
```

Verify the writes from the Spark master (or any host with the Spark distribution):

```bash
sudo -u spark $SPARK_HOME/bin/spark-shell \
  --conf spark.sql.catalog.iceberg=org.apache.iceberg.spark.SparkCatalog \
  --conf spark.sql.catalog.iceberg.catalog-impl=org.apache.iceberg.rest.RESTCatalog \
  --conf spark.sql.catalog.iceberg.uri=http://<polaris-host>:30000/api/catalog \
  ...
spark.sql("SELECT * FROM iceberg.test.yaml_dag_smoke").show()
```

### Reading the logs in the admin UI

Open the run in the admin UI's **Run Log** tab. It shows **one combined view** that streams the run's driver-orchestration events at the top followed by each task's full stdout/stderr below — every section with its own per-source byte offset, so re-rendering does not duplicate already-seen lines, and section headers stay stuck to the top while you scroll. For an active task the section is `LIVE` and polls every 2 seconds; once a task hits a terminal state the section stops polling but stays visible.

This is what surfaces a `spark-submit`'s long stack of `INFO`/`WARN`/`ERROR` lines, Iceberg metrics, S3A I/O — everything the remote `spark-submit` printed over the SSH stream — in one continuous tail-style view, instead of forcing you to click into each task.

## What you have built

- A kiok cluster that runs the **same workload three times**, authored once in YAML, once in Python, once in Java — with no jar / classpath / build step on the operator side. Every Java DAG was compiled in-process by the kiok leader at sync/upload time.
- A `src/main/{script,python,java}/` source layout that maps cleanly to git-sync **and** bundle-upload deploys.
- Credentials never written down in any DAG file — every secret reference uses kiok's connection store, decrypted only in-memory inside the Worker that's about to exec the task.
- A kiok Worker host that doesn't carry a `SPARK_HOME` of its own — the heavy `spark-submit` runs on the Spark master via passwordless SSH, and the Worker just streams the result.
