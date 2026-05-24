# Git Sync

Git Sync lets kiok treat a git repository as the source of truth for DAGs. The leader periodically clones/pulls each configured repository, compiles every DAG it finds, and registers them — so a `git push` is all it takes to deploy a workflow.

## Configuring a repository

Add a repository from the admin UI's **Settings → Git Sync** page, or via configuration. Each repository has:

| Field | Meaning |
|---|---|
| `name` | A label for the repository, unique within the cluster |
| `url` | The clone URL (`https://…` or `git@…`) |
| `ref` | A branch (default `main`) or a tag (`tag:<name>`) to check out |
| `subpath` | A directory within the repo to scan; defaults to the repo root |
| `credentialConnectionId` | For private repos — the connection holding git credentials |

Multiple repositories are supported, so different teams can each point at their own repo, branch, or tag.

## What gets synced

On each interval (`kiok.gitsync.interval.ms`, default 60s) the leader checks out the configured ref and scans the subpath for:

- `*.yaml` / `*.yml` — YAML DAG definitions;
- `*.py` — Python-authored DAGs (compiled with `python3 -m kiok.compile`);
- `*.java` — Java-authored DAGs, batch-compiled in-process against kiok-sdk on the master's classpath;
- `*.sh` — shell-script DAGs: filename stem becomes the DAG id, file body becomes a single task named `run`.

Each compiled DAG is upserted into the metadata store with `origin: git`, tagged with the repository and source path. A DAG that already exists is updated in place; its run history is preserved.

### Recommended layout

The scanner is location-agnostic — it picks up files by extension wherever they sit under the configured subpath — but the recommended convention mirrors the standard Maven/Gradle source tree:

```
<repo>/
├── src/main/yaml/     # *.yaml DAGs
├── src/main/script/   # also a fine place for *.yaml; *.sh shell DAGs live here
├── src/main/python/   # *.py DAGs (kiok Python SDK)
└── src/main/java/     # *.java DAGs (kiok Java SDK)
```

The repo holds **source only** — there is no build step on the operator side. kiok's leader handles every compile in-process when it pulls the repo.

### Java source compilation

For `.java` sources, the leader uses the JDK's in-process `javax.tools.JavaCompiler` (kiok master must run on a JDK, not a JRE — kiok's packaged distribution already ships with one). The compiler's classpath is auto-derived from the master's own `lib/` directory, so a user-authored DAG can reference the kiok SDK (`com.cloudcheflabs.kiok.sdk.{Dag, Task, KiokDag, Conn}`) without any build configuration on the repo side.

Multiple `.java` files are compiled in a single javac invocation so cross-file references resolve naturally — a DAG class can import a helper from a sibling source file. Each compiled `KiokDag` subclass is then loaded with an isolated `URLClassLoader` and its `define()` method invoked once.

The original `.java` source text — not a "compiled from N files" marker — is stored as the DAG's `source` field, so the admin UI's **Source** tab shows operators the actual Java code they authored, with the resulting `DagSpec` (the in-memory structure the scheduler executes) rendered just below it.

## Prune on delete

When a DAG file is **removed** from the repository, git-sync prunes the corresponding DAG on the next sync — the DAG disappears from the cluster, matching the repo.

**Run history survives the prune.** History is keyed by the DAG's unique id (derived from repo + source path), not by the DAG record itself. So if the same file is later re-pushed — same name, same path — the DAG reappears and **continues its existing run history** rather than starting fresh. Deleting and re-adding a DAG through git is therefore safe and non-destructive.

## Private repositories

Git Sync configuration never holds credentials. For a private repository, store the git credentials in a [connection](connections.md) — an SSH `privateKey`, or `username` + `password`/token — and reference it by `credentialConnectionId`. The credentials live only in the KMS-encrypted connection store.

## Manual sync

The sync loop runs on the leader on its interval, but a sync can also be triggered immediately from the admin UI — useful right after a push when you don't want to wait for the next tick.
