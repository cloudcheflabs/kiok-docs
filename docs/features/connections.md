# Connections

A **connection** is a named, encrypted bag of credentials and endpoint settings. Connections keep secrets out of DAG definitions, git-sync configuration, and the backup configuration — so DAG files stay safe to commit to a public repository.

## What connections hold

Each connection has an id and a set of key/value properties. Typical uses:

- **Git credentials** — an SSH `privateKey`, or `username` + `password`/token, for a private repository synced by [Git Sync](git-sync.md).
- **S3 credentials** — `endpoint`, `region`, `bucket`, `accessKey`, `secretKey`, `pathStyle` — for a [Backup &amp; Restore](backup.md) destination.
- Any other endpoint or credential a task needs to reach an external system.

## Storage

Connections live in a dedicated KMS-encrypted RocksDB store on the leader, synchronised to followers. Credentials are never written to config files or returned in plaintext by the API.

## Referencing a connection

Configuration and task bodies reference a connection **indirectly**, never by embedding the secret:

- `${conn.<id>.<key>}` — substitutes one property of a connection;
- `${secret.<name>}` — substitutes a named secret.

kiok resolves the placeholder at execution time. Because the DAG definition only ever contains the placeholder, the same DAG file is safe in a public git repo — the actual credential exists only in the encrypted connection store.

Manage connections from the admin UI's **Settings → Connections** page.
