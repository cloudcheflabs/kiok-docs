# Getting Started

This walks through your first end-to-end interaction with a freshly-installed kiok cluster: signing in to the admin UI, registering a DAG, triggering a run, and watching it execute. If the cluster isn't running yet, start with [Download &amp; Install](installation.md).

## Open the Admin UI

Once the Master reports the cluster is ready, point a browser at:

```
http://localhost:8080
```

The first sign-in uses the bootstrap credentials `admin` / `admin`. The UI immediately requires a password change before any other action is permitted.

The workflow view lists every DAG, its origin (manual, git, or bundle), its schedule, and recent run history. The settings area — under **Settings** — covers the cluster: topology, KMS, IAM, connections, git-sync, bundles, and backup.

## Write a DAG

A DAG is a set of tasks plus the dependencies between them. The simplest way to author one is YAML:

```yaml
dag:
  id: hello_kiok
  schedule: "0 * * * *"   # hourly; omit for manual-only
tasks:
  - id: extract
    type: shell
    script: |
      #!/bin/bash
      echo "extracting data"
  - id: transform
    type: python
    requires: [extract]
    script: |
      print("transforming", sum(range(10)))
  - id: load
    type: shell
    requires: [transform]
    script: |
      #!/bin/bash
      echo "loading results"
```

`requires` declares upstream dependencies — `transform` runs only after `extract` succeeds, and `load` only after `transform`. kiok validates the graph (no cycles, no dangling references) before accepting it.

A DAG can equally be authored in **Python or Java code** instead of YAML — see [Authoring DAGs](../features/dag-authoring.md).

## Register the DAG

Register the YAML from the admin UI (**DAGs → Register**), or from the bundled CLI:

```bash
bin/submit.sh register hello_kiok.yaml --token <jwt>
```

`submit.sh` authenticates with either a JWT (`--token`, or `KIOK_USER_TOKEN`) or an access-key pair (`--accesskey` / `--secretkey`). It targets `localhost:8080` by default; override with `--master host:port`.

## Trigger a Run

The DAG above has an hourly schedule, but you can also trigger it on demand — from the UI's **Run** button, or the CLI:

```bash
bin/submit.sh trigger hello_kiok --token <jwt>
```

This returns a run id. The leader assigns the whole run to one worker — its *driver* — which walks the graph, dispatches each task as its dependencies are satisfied, and streams status and logs back.

## Watch the Run

Open the run in the admin UI to see:

- the **graph** view — each task node coloured by state (pending, running, success, failed);
- the **job list** — per-task worker, timing, exit code;
- the **run log** — the driver's orchestration log, streamed live;
- per-task logs — click any task to tail its stdout/stderr.

Or poll from the CLI:

```bash
bin/submit.sh status <run-id> --token <jwt>
bin/submit.sh log <run-id> extract --token <jwt>
```

## What's Next

- Author DAGs in YAML, Python, or Java — [Authoring DAGs](../features/dag-authoring.md).
- Let kiok pull DAGs straight from a git repository — [Git Sync](../features/git-sync.md).
- Understand how runs are placed and recovered — [Worker-Driver Execution](../features/execution.md).
- Configure cron schedules and catchup — [Scheduler &amp; Triggers](../features/scheduler.md).
- Run a real deployment — [Cluster Operations](../operations/operations.md).
