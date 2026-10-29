# Git Sync

Git Sync lets kiok treat a git repository as the source of truth for DAGs. The leader periodically clones/pulls each configured repository, compiles every DAG it finds, and registers them — so a `git push` is all it takes to deploy a workflow.

## Turning it on

Git Sync is off by default. The leader pulls repositories only when `kiok.gitsync.enabled=true` is set in `kiok.properties` (or `KIOK_GITSYNC_ENABLED=true` in the environment) on the masters; restart them after changing it.

While it is off:

- the leader pulls nothing — not on the interval, and not when it becomes leader;
- **Sync now** is unavailable: the **Git Sync** page shows a banner saying git sync is disabled, and `POST /api/v1/admin/gitsync/sync` returns `409 Conflict` with the reason;
- repositories can still be added, edited and removed, so they are ready when you turn it on;
- DAGs registered by an earlier sync stay registered and keep running on their schedules;
- a leader that starts with repositories registered logs a `WARN` naming how many will not be pulled.

`GET /api/v1/admin/gitsync/status` returns the leader's setting: `{"enabled": true, "intervalMs": 60000}`.

!!! warning "Upgrading from a build that ignored the setting"
    Before this setting took effect, the leader pulled registered repositories regardless of `kiok.gitsync.enabled`, and the bundled `kiok.properties` sets it to `false`. If you use Git Sync, set `kiok.gitsync.enabled=true` before upgrading, or the leader stops pulling.

## Configuring a repository

Add a repository from the admin UI's **Settings → Git Sync** page or the API below. A first cluster can also be seeded from `kiok.properties` (see [Seeding from kiok.properties](#seeding-from-kiokproperties)). Each repository has:

| Field | Meaning |
|---|---|
| `name` | A label for the repository, unique within the cluster |
| `url` | The clone URL — `https://…`, `git@…`, or `file:///path/to/repo` for a repository on the leader's own disk (the leader clones it, so the path must exist and be readable by the kiok user on whichever master is leader — put it on every master) |
| `ref` | The branch (default `main`) or tag name to check out |
| `tag` | `true` when `ref` names a tag, `false` (default) for a branch. In the admin UI this is **Ref type** |
| `subpath` | A directory within the repo to scan; defaults to the repo root |
| `credentialConnectionId` | For private repos — the connection holding git credentials. When empty, a connection whose id equals the repository `name` is used if one exists |

Multiple repositories are supported, so different teams can each point at their own repo, branch, or tag. Saving under an existing `name` replaces that repository. Removing a repository stops syncing it; the DAGs it already registered are kept.

Repositories are stored in the replicated metadata store, so every master sees the same list and a new leader keeps syncing them.

### Seeding from kiok.properties

`kiok.gitsync.repos` seeds the repository list **once**: when a master becomes leader and the metadata store holds no repositories yet, each entry is saved as if it had been added in the UI. After that the stored list is the source of truth — editing `kiok.gitsync.repos` later changes nothing, so manage repositories in the UI or API.

The value is a comma-separated list of `<name>|<url>|<ref>|<subpath>|<credentialConnectionId>` entries; only `name` and `url` are required. `ref` is a branch name, `branch:<name>`, or `tag:<name>` for a tag.

```properties
kiok.gitsync.enabled=true
kiok.gitsync.repos=team-dags|https://git.example.com/team/dags.git|main|src/main,\
  release-dags|git@git.example.com:team/dags.git|tag:v1.4.0||team-git-ssh
```

### API

All Git Sync endpoints are administrator-only.

| Method and path | Does |
|---|---|
| `GET /api/v1/admin/gitsync/status` | `{"enabled": <kiok.gitsync.enabled>, "intervalMs": <kiok.gitsync.interval.ms>}` as the leader has them |
| `GET /api/v1/admin/gitsync/repos` | The repositories, each with `lastSyncAt` (epoch ms, `0` = never) and `errors` (files the last sync skipped, with the reason) |
| `POST /api/v1/admin/gitsync/repos` | Saves a repository: `{"name", "url", "ref", "tag", "subpath", "credentialConnectionId"}` → `{"saved": "<name>"}` |
| `DELETE /api/v1/admin/gitsync/repos/<name>` | Removes a repository → `{"deleted": "<name>"}` |
| `POST /api/v1/admin/gitsync/sync` | Starts a sync of every repository now → `{"status": "sync triggered"}`; `409` while git sync is disabled |

A follower forwards these requests to the leader, which runs the sync and holds the last sync's results.

!!! note "Git-synced DAGs are run by administrators"
    A repository registers all of its DAGs at once, and the owner recorded on them is whoever set the repository up — not who wrote a given file. So only an administrator can run, cancel or delete a git-synced DAG; anyone else gets `403 only an administrator can run DAG '<name>': it came from git '<repo>', not from a single user`. Scheduled runs are unaffected — the scheduler is not a user. See [Who may operate a DAG](auth-authz.md#dags-and-runs).

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

## DAG ids

A git DAG keeps the id its source declares — `dag.id` in YAML, the DAG id in the Python or Java SDK, the file name stem for a plain `.sh` — exactly as a [bundle](bundles.md#identity-and-history) DAG does. That is the id you trigger it by and see in the console. A source that declares no id falls back to `<repo>_<path>`.

An id must be free. A file is skipped, and the reason recorded, when its id is:

- declared by another file of the same repository (the first file keeps it: `b.yaml: DAG id 'daily_load' is also declared by a.yaml`);
- held by a bundle, another repository, or a DAG registered through the API (`... is already used by bundle 'finance-v2'`).

The other files still sync. The **Git Sync** page shows each repository's last sync time and every file the last sync skipped, with the reason (`GET /api/v1/admin/gitsync/repos` returns them as `lastSyncAt` and `errors`). Registering a DAG through the API under an id a repository owns is refused with `409 Conflict`.

!!! note "Upgrading from a release that derived git ids"
    Earlier releases named git DAGs `<repo>_<path>`. After the upgrade the next sync registers each DAG under its declared id and prunes the old one; the run history recorded under the old id stays where it was.

## Prune on delete

When a DAG file is **removed** from the repository, git-sync prunes the corresponding DAG on the next sync — the DAG disappears from the cluster, matching the repo.

**Run history survives the prune.** History is keyed by the DAG's id, not by the DAG record itself. So if the same file is later re-pushed — same name, same path — the DAG reappears and **continues its existing run history** rather than starting fresh. Deleting and re-adding a DAG through git is therefore safe and non-destructive.

## Private repositories

Git Sync configuration never holds credentials. For a private repository, store the git credentials in a [connection](connections.md) — an SSH `privateKey`, or `username` + `password`/token — and reference it by `credentialConnectionId`. The credentials live only in the KMS-encrypted connection store.

## Manual sync

The sync loop runs on the leader on its interval, but a sync can also be triggered immediately with **Sync now** in the admin UI or `POST /api/v1/admin/gitsync/sync` — useful right after a push when you don't want to wait for the next tick. The request returns as soon as the sync has started; check the repository's **Last sync** time (or `lastSyncAt`) and the skipped-file list for the outcome.

While git sync is disabled (`kiok.gitsync.enabled=false`) **Sync now** is unavailable and the API answers `409 Conflict` with the reason. While the cluster is in [maintenance mode](cluster-maintenance.md) a sync, manual or scheduled, registers nothing.
