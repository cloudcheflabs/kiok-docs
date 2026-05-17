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

> Whatever format you author the DAG in, you write the **literal reference string** `${conn...}` / `${secret...}`. Do not interpolate it with your language's own string substitution — kiok resolves it at run time.

## Examples

The same DAG below — a DB export that then calls an API — is shown in all three authoring formats. Each references the `analytics_db` connection and the `api_token` secret registered above.

### YAML

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
      psql -h "${conn.analytics_db.host}" -p "${conn.analytics_db.port}" \
           -U "${conn.analytics_db.username}" \
           -c "\copy report TO '/tmp/report.csv' CSV"
  - id: notify
    type: http
    requires: [dump]
    config:
      url: https://api.example.com/ingest
      method: POST
      body: '{"token":"${secret.api_token}","status":"done"}'
```

### Python

```python
from kiok import Dag

dag = Dag("db_export", schedule="0 4 * * *")

dag.task("dump",
         script='#!/bin/bash\n'
                'export PGPASSWORD="${conn.analytics_db.password}"\n'
                'psql -h "${conn.analytics_db.host}" -p "${conn.analytics_db.port}" '
                '-U "${conn.analytics_db.username}" '
                '-c "\\copy report TO \'/tmp/report.csv\' CSV"')

# The ${...} text is a literal kiok reference — not a Python f-string.
dag.task("notify", task_type="python", requires=["dump"],
         script='import urllib.request, json\n'
                'req = urllib.request.Request(\n'
                '    "https://api.example.com/ingest",\n'
                '    data=json.dumps({"status": "done"}).encode(),\n'
                '    headers={"Authorization": "Bearer ${secret.api_token}"})\n'
                'print(urllib.request.urlopen(req).status)')
```

### Java

```java
public class DbExportDag implements KiokDag {
    @Override
    public Dag define() {
        Dag dag = new Dag("db_export").schedule("0 4 * * *");

        dag.task("dump").shell(
            "#!/bin/bash\n" +
            "export PGPASSWORD=\"${conn.analytics_db.password}\"\n" +
            "psql -h \"${conn.analytics_db.host}\" -p \"${conn.analytics_db.port}\" \\\n" +
            "     -U \"${conn.analytics_db.username}\" \\\n" +
            "     -c \"\\copy report TO '/tmp/report.csv' CSV\"");

        dag.task("notify").requires("dump")
           .http("https://api.example.com/ingest")
           .config("method", "POST")
           .config("body", "{\"token\":\"${secret.api_token}\",\"status\":\"done\"}");

        return dag;
    }
}
```

In every case the registered DAG — the `DagSpec` stored in kiok — contains only the literal `${conn.analytics_db.password}` and `${secret.api_token}` strings. The credentials never leave the encrypted connection store until a worker resolves them for that single task execution.
