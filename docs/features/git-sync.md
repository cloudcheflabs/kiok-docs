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
- `*.jar` — Java DAG jars (every `KiokDag` class is compiled).

Each compiled DAG is upserted into the metadata store with `origin: git`, tagged with the repository and source path. A DAG that already exists is updated in place; its run history is preserved.

## Prune on delete

When a DAG file is **removed** from the repository, git-sync prunes the corresponding DAG on the next sync — the DAG disappears from the cluster, matching the repo.

**Run history survives the prune.** History is keyed by the DAG's unique id (derived from repo + source path), not by the DAG record itself. So if the same file is later re-pushed — same name, same path — the DAG reappears and **continues its existing run history** rather than starting fresh. Deleting and re-adding a DAG through git is therefore safe and non-destructive.

## Private repositories

Git Sync configuration never holds credentials. For a private repository, store the git credentials in a [connection](connections.md) — an SSH `privateKey`, or `username` + `password`/token — and reference it by `credentialConnectionId`. The credentials live only in the KMS-encrypted connection store.

## Manual sync

The sync loop runs on the leader on its interval, but a sync can also be triggered immediately from the admin UI — useful right after a push when you don't want to wait for the next tick.
