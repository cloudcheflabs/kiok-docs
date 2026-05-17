# Connections

A **connection** is a named, encrypted bag of credentials and endpoint settings. Connections keep secrets out of DAG definitions, git-sync configuration, and the backup configuration — so a DAG file stays safe to commit to a public repository.

## What connections hold

Each connection has an id, a type label, and a set of key/value properties. Typical uses:

- **Git credentials** — an SSH `privateKey`, or `username` + `password`/token, for a private repository synced by [Git Sync](git-sync.md).
- **S3 credentials** — `endpoint`, `region`, `bucket`, `accessKey`, `secretKey`, `pathStyle` — for a [Backup &amp; Restore](backup.md) destination.
- **Database / API credentials** — anything a task needs to reach an external system.

## Registering a connection

Open **Settings → Connections** in the admin UI and fill in **Add / update connection**:

| Field | Meaning |
|---|---|
| Connection id | Unique id you reference from DAGs (e.g. `analytics_db`) |
| Type | A label — `s3` / `git` / `secret` / `jdbc` / `kafka` / `generic`. Organisational only; it does not change behaviour |
| Description | Free text |
| Properties | One `key=value` per line |

**Example — a database connection** `analytics_db`:

```
host=db.internal
port=5432
username=reporter
password=p4ssw0rd-...
```

**Example — a standalone secret** `api_token` (type `secret`) — give it a single property named `value`:

```
value=ghp_xxxxxxxxxxxxxxxx
```

Connections are stored in a dedicated **KMS-encrypted RocksDB store** on the leader and synchronised to followers. The API never returns the values in plaintext.

## Referencing a connection from a DAG

A DAG references a connection **indirectly** — it carries only the reference, never the secret. Two forms:

| Reference | Resolves to |
|---|---|
| `${conn.<id>.<key>}` | One property of a connection — e.g. `${conn.analytics_db.password}` |
| `${secret.<name>}` | The `value` property of connection `<name>` — `${secret.api_token}` is shorthand for `${conn.api_token.value}` |

References work **inside a task's `script` body and inside any task `config` value**. A worker substitutes them in-memory **immediately before running the task** — the real value exists only transiently in the worker process, never in stored metadata, git, or the admin UI. A reference to an unknown connection or property fails the task.

> The reference is resolved at **task-execution time**, not when the DAG is authored or compiled. You cannot read a credential's actual value from YAML or from Python/Java SDK code — by design, so the secret never enters the stored DAG definition.

## SDK reference helpers

When authoring a DAG in **Python or Java code**, build the reference with the SDK helper instead of hand-typing the `${...}` literal — it is typo-safe and self-documenting. The helper only constructs the reference string; it does not read the value.

| | Python (`kiok` SDK) | Java (`kiok` SDK) |
|---|---|---|
| Connection property | `conn("analytics_db", "password")` | `Conn.ref("analytics_db", "password")` |
| Standalone secret | `secret("api_token")` | `Conn.secret("api_token")` |

Both return the same literal kiok resolves at run time — e.g. `${conn.analytics_db.password}`.

## Examples

The same DAG below — a DB export that then calls an API — is shown in all three authoring formats. Each references the `analytics_db` connection and the `api_token` secret registered above.

### YAML

YAML carries the `${...}` reference literally:

```yaml
dag:
  id: db_export
  schedule: "0 4 * * *"
tasks:
  - id: dump
    type: shell
    script: |
      #!/bin/bash
      export PGPASSWORD="${conn.analytics_db.password}"
      psql -h "${conn.analytics_db.host}" -U "${conn.analytics_db.username}" \
           -c "\copy report TO STDOUT CSV"
  - id: notify
    type: http
    requires: [dump]
    config:
      url: https://api.example.com/ingest
      method: POST
      body: '{"token":"${secret.api_token}","status":"done"}'
```

### Python

Use the `conn` / `secret` helpers from the `kiok` SDK:

```python
from kiok import Dag, conn, secret

dag = Dag("db_export", schedule="0 4 * * *")

dag.task("dump", script=f"""#!/bin/bash
export PGPASSWORD="{conn('analytics_db', 'password')}"
psql -h "{conn('analytics_db', 'host')}" \\
     -U "{conn('analytics_db', 'username')}" \\
     -c "\\copy report TO STDOUT CSV"
""")

dag.task("notify", task_type="python", requires=["dump"], script=f"""
import urllib.request
req = urllib.request.Request(
    "https://api.example.com/ingest",
    headers={{"Authorization": "Bearer {secret('api_token')}"}})
print(urllib.request.urlopen(req).status)
""")
```

### Java

Use the `Conn` helper from the `kiok` SDK:

```java
import com.cloudcheflabs.kiok.sdk.Conn;
import com.cloudcheflabs.kiok.sdk.Dag;
import com.cloudcheflabs.kiok.sdk.KiokDag;

public class DbExportDag implements KiokDag {
    @Override
    public Dag define() {
        Dag dag = new Dag("db_export").schedule("0 4 * * *");

        dag.task("dump").shell(
            "#!/bin/bash\n" +
            "export PGPASSWORD=\"" + Conn.ref("analytics_db", "password") + "\"\n" +
            "psql -h \"" + Conn.ref("analytics_db", "host") + "\" \\\n" +
            "     -U \"" + Conn.ref("analytics_db", "username") + "\" \\\n" +
            "     -c \"\\copy report TO STDOUT CSV\"");

        dag.task("notify").requires("dump")
           .http("https://api.example.com/ingest")
           .config("method", "POST")
           .config("body", "{\"token\":\"" + Conn.secret("api_token") + "\"}");

        return dag;
    }
}
```

In every case the registered DAG — the `DagSpec` stored in kiok — contains only the literal `${conn.analytics_db.password}` and `${secret.api_token}` strings. The credentials never leave the encrypted connection store until a worker resolves them for that single task execution.
