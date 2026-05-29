# Date Placeholders

kiok evaluates `#{ ... }` tokens inside every task's `script` and every value in its `config` map immediately before the task runs. The six functions match the [chango query-exec date functions](https://cloudcheflabs.github.io/chango-docs/latest/user-guide/query-exec/#date-functions) verbatim, so a SQL snippet that worked in chango copies over unchanged.

## Why

YAML and Python/Java SDK DAGs all compile to the same `DagSpec`, but YAML has no place to compute "yesterday" the way code can. For daily-scheduled jobs whose source table partitions slide forward each day — typically `MERGE INTO` queries against an Iceberg landing table — the SQL needs to refer to the current run date *at run time*, not at registration time. `#{ nowMinusFormatted(...) }` is the substitution kiok performs in the worker right before submit.

Java/Python SDK DAGs typically compute dates in code; their tasks contain no `#{ ... }` tokens and the substitution pass is a no-op for them. There's no penalty for mixing — you can have one DAG with placeholders and another without.

## Functions

| Function | Returns | Example |
|---|---|---|
| `nowInMillis()` | epoch milliseconds | `1748520896000` |
| `nowFormatted("yyyy-MM-dd")` | current time, formatted | `2026-05-29` |
| `nowPlusInMillis(y, mo, d, h, mi, w)` | epoch ms after adding the six units | `1748607296000` |
| `nowPlusFormatted(y, mo, d, h, mi, w, "fmt")` | formatted, after adding | `2026-05-30` |
| `nowMinusInMillis(y, mo, d, h, mi, w)` | epoch ms after subtracting | `1748434496000` |
| `nowMinusFormatted(y, mo, d, h, mi, w, "fmt")` | formatted, after subtracting | `2026-05-28` |

Argument order is fixed: **years, months, days, hours, minutes, weeks**. No seconds slot — match the chango grammar exactly.

The format string accepts everything `java.time.format.DateTimeFormatter` does. kiok rewrites any run of `Y` to `y` silently because `YYYY-MM-dd` is what every chango example uses and almost nobody wants ISO week-based year — if you do, write `uuuu`.

## Examples

### Daily MERGE INTO against an Iceberg landing table

```yaml
- id: merge_orders
  type: trino
  config:
    trino.url:      "https://trino-gw.example/"
    trino.user:     "${conn.trinoGw.username}"
    trino.password: "${conn.trinoGw.password}"
    trino.catalog:  "iceberg"
    trino.schema:   "warehouse"
  script: |
    MERGE INTO warehouse.orders t
    USING (
      SELECT id, customer_id, total, updated_at
      FROM staging.orders
      WHERE date(updated_at) = DATE '#{ nowMinusFormatted(0, 0, 1, 0, 0, 0, "yyyy-MM-dd") }'
    ) s
    ON t.id = s.id
    WHEN MATCHED THEN UPDATE SET total = s.total, updated_at = s.updated_at
    WHEN NOT MATCHED THEN INSERT (id, customer_id, total, updated_at)
                     VALUES (s.id, s.customer_id, s.total, s.updated_at)
```

Schedule this DAG at `0 3 * * *` and each daily run targets the previous day's partition automatically — no DAG re-registration, no template engine wrapper.

### Naming a Spark batch with the current date

```yaml
- id: spark_etl
  type: livy
  config:
    livy.url:  "http://livy:8998"
    livy.file: "s3a://my-jobs/etl-1.0.jar"
    livy.name: "etl_#{ nowFormatted(\"yyyyMMdd\") }"
    livy.args: ["--date", "#{ nowFormatted(\"yyyy-MM-dd\") }"]
```

The substitution runs on every value in the config map (not just on `script`), so `livy.name` and the entries in `livy.args` both get their `#{ ... }` resolved before kiok issues the Livy POST.

### Window relative to the current minute

```yaml
- id: late_arrivals
  type: trino
  config:
    trino.url:  "..."
    trino.user: "${conn.trino.username}"
  script: |
    SELECT *
    FROM events.late_inbox
    WHERE arrived_at < TIMESTAMP '#{ nowMinusFormatted(0, 0, 0, 0, 50, 0, "yyyy-MM-dd HH:mm:ss.SSS") }'
```

50 minutes ago, formatted to millisecond precision. Useful for scheduled cleanup that wants a "settled" cutoff and not the wall clock.

## What gets substituted

The DAG-run driver runs the substitution pass on:

- **Task `script`** — the inline body for `shell` / `python` / `trino` / `ontul` jobs.
- **Every value in `config`** — including `livy.name`, `livy.args` (as a JSON-encoded string in the underlying spec), `ontul.deps`, etc.

In that order: `${conn.*}` / `${secret.*}` are resolved first (so a credential whose stored value happens to contain `#{ ... }` is *not* re-substituted), then date placeholders.

Unknown function names or arity mismatches throw — a typo in a placeholder lands as an obvious task-failure rather than running with a garbled SQL filter.

## Resolving in code instead

If you author the DAG in Python or Java and prefer to compute dates locally, just don't use `#{ ... }`. The Java SDK example:

```java
String day = LocalDate.now().minusDays(1).toString();
dag.task("merge_orders")
   .trino(GATEWAY_URL, USER, """
       MERGE INTO ... USING (SELECT ... WHERE day = DATE '%s') ...
       """.formatted(day))
   .trinoPassword("${conn.trinoGw.password}");
```

Both styles produce the same `DagSpec`. The substitution is purely a YAML-ergonomics feature.
