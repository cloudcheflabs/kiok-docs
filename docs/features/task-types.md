# Task Types

A task is one node in a DAG. kiok runs three task types; all three are authored the same way in YAML, Python, or Java.

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

## `python`

Runs an inline Python script with the Worker's `python3` (`kiok.python.path`). Useful for data manipulation without a shell wrapper.

```yaml
- id: transform
  type: python
  requires: [extract]
  script: |
    print("rows:", sum(range(100)))
```

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

## Common task fields

| Field | Meaning |
|---|---|
| `id` | Task identifier, unique within the DAG |
| `type` | `shell`, `python`, or `http` |
| `script` | Inline body for `shell` / `python` tasks |
| `requires` | List of upstream task ids that must succeed first |
| `timeout` | Per-task execution timeout; overrides the DAG / cluster default |
| `config` | Type-specific settings (e.g. `url`, `method`, `body` for `http`) |

## Referencing credentials

Task bodies must never embed secrets. Reference a stored credential indirectly — `${conn.<id>.<key>}` for a connection field, `${secret.<name>}` for a named secret — and kiok resolves it at execution time from the encrypted [connection store](connections.md). The DAG definition stays safe to commit to a public git repository.
