# DAG Bundles

A **bundle** is a zip archive of DAG definitions uploaded directly to the cluster. Bundles are the deployment path for clusters that cannot reach a git server — fully air-gapped environments — and for one-off DAG sets that don't warrant a repository.

## What a bundle contains

A bundle zip holds any mix of:

- `*.yaml` / `*.yml` — YAML DAG definitions;
- `*.py` — Python-authored DAGs (compiled with `python3 -m kiok.compile`);
- `*.java` — Java-authored DAGs, batch-compiled in-process against kiok-sdk on the master's classpath;
- `*.sh` — shell-script DAGs (filename stem → DAG id, file body → single shell task named `run`).

The same recommended layout as [Git Sync](git-sync.md) — `src/main/{yaml,script,python,java}/` — applies inside the zip too. The scanner is location-agnostic (it walks the whole archive and picks files by extension), so the directory layout is operator-facing convention rather than a hard requirement.

A bundle therefore holds **source only** — there is no build step on the operator side; kiok's leader compiles every DAG in-process when it processes the upload, exactly as it does for git-sync.

Upload it from the admin UI's **Settings → DAG Bundles** page. Each compiled DAG is registered with `origin: bundle`, tagged with the bundle name and the source path inside the zip.

## Upload and overwrite

A bundle is identified by the name you give it on upload. Re-uploading under the **same name overwrites** that bundle — its previous DAGs are dropped and replaced by the new archive's contents. This makes a bundle a single versioned unit you can re-publish.

## Compile error reporting

Entries in the zip are compiled independently. If one file fails to compile — a YAML schema error, a Python syntax error, a bad dependency — the rest of the bundle still loads, and the failure is **recorded against the bundle**:

- the bundle metadata keeps an `errors` list, one entry per failed file with the reason;
- the **DAG Bundles** page shows a count badge and an expandable list of the failures, so you can see exactly which file broke and why without digging through logs.

A bundle with three good DAGs and one broken file registers the three and reports the one.

## Storage

Bundle DAGs — like manually-registered DAGs — are **persisted in the leader's metadata store** (KMS-encrypted RocksDB). This differs from git DAGs, which are re-derived from the repository on every sync. Because bundle DAGs are persisted state, they are included in [Backup &amp; Restore](backup.md).

## Identity and history

A bundle DAG's unique id is derived from `<bundle name>/<source path>`, so re-uploading the same bundle keeps each DAG's id stable and **continues its run history**. Deleting a bundle removes its DAGs but leaves their run history intact.
