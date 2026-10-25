# Admin UI Guide

The admin console is served by every master on its admin port (`kiok.master.admin.port`, default 8080),
under the configured context path. It talks only to the REST API, so everything below can also be done
with the API; this page walks through each page and control.

Dates and times are always shown in **the browser's locale and time zone**, from the epoch values the
server returns. A missing value shows as `—`, never as an empty cell, `Invalid Date` or a 1970 date.

## Login

- **User** / **Password** — a local account, or an LDAP / Active Directory account when LDAP is enabled
  (a note under the form says so). A wrong password shows `invalid credentials` and stays on the page.
- **Sign in with OpenID Connect** / **Sign in with SAML** — appear only for the providers enabled under
  Settings → SSO (read from `GET /api/v1/auth/sso/providers`, no rebuild needed). See
  [Single Sign-On](sso.md).
- The first sign-in with the factory `admin` / `admin` password opens a **Change your password** dialog
  that cannot be dismissed: nothing else works until the password is changed (at least 6 characters,
  typed twice).
- A link opened while signed out (a run page, say) is restored after signing in.

The header has **DAGs**, **Executions**, **Schedules**, **Settings** (administrators only — a settings
URL sends anyone else back to DAGs), the user name, and **Sign out**, which revokes the session on the
server (see [Authentication &amp; Authorization](auth-authz.md)).

## DAGs

- **Search** — server-side, case-insensitive, over the DAG name, git repository and source path.
- **Table** — DAG (link to its page), **Origin** (`git` / `bundle` / `manual`, with the repository or
  bundle and the source path), **Schedule** (a plain-English description such as *Every Monday through
  Friday at 09:00* with the cron in parentheses, or the cron alone when it cannot be described exactly;
  *manual trigger* when there is none; and **next run** — the next fire time), **Tasks**, **Owner**.
- **Pagination** — 15 DAGs per page. Deleting the last DAG of a page moves back a page.
- **Register DAG** — toggles a form for a YAML definition. An invalid definition is refused with the
  parser's or validator's reason and the form stays open. An id that belongs to a bundle or a git
  repository is refused (`409`).
- **Delete** (bin icon) — asks first (**Cancel** / **Delete**); the definition is removed, run history is kept.

## DAG detail

The subtitle shows origin, schedule, next run and owner.

- **Graph** — the task dependency graph (zoom in / out and fit-view controls), and a table of tasks with
  type and dependencies.
- **Source** — the original source (read-only), the compiled DAG spec, and each task's script or config.
- **Runs** — the newest 50 runs of this DAG, loaded in one request (`GET /api/v1/runs?dag=<id>`):
  a **Run history** chart (one point per run at its start time, coloured by outcome, the line showing the
  duration trend — click a point to open that run) and a table (Run, Trigger, State, Started, Duration;
  10 per page). While a run of this DAG is still going the tab refreshes itself every 3 seconds.
- **Trigger run** — opens a dialog for optional **arguments**, one `name=value` per line. A line without
  `=` is refused in the dialog; a name that is not `[A-Za-z_][A-Za-z0-9_]*` is refused by the server.
  A shell or python task reads each argument as `KIOK_ARG_<NAME>` (see
  [Scheduler &amp; Triggers](scheduler.md#manual-triggers)). After triggering, a confirmation links to
  the new run. During a [maintenance window](cluster-maintenance.md) the trigger is refused with `503`
  in the dialog.

## Executions

The newest 120 runs across every DAG you may read, newest first, in one request.

- **Search** — over run id, DAG, state and trigger (e.g. `FAILED`).
- **Table** — Run, DAG, Trigger (`manual`, `scheduled`, `catchup`), State, Started, Finished, Elapsed;
  20 per page.
- **Refresh** — reloads the list. While any listed run is still going the page refreshes itself every
  4 seconds, so a run turns `SUCCESS` / `FAILED` without a reload.

## Execution detail

- **Header** — the run id, a link to its DAG, the state badge, and **Cancel run** while it runs (it asks
  first: **Keep running** / **Cancel run**; running tasks are killed, an external job — ontul, Livy,
  Trino — is killed at the engine) or **Re-run** once it has finished (triggers the same DAG with the same
  arguments and opens the new run).
- **Meta** — Trigger, Triggered, Started, Finished (full date and time), Elapsed.
- **Arguments** — the run's trigger arguments, when it has any.
- **Graph** — each node's border is its task's state colour (SUCCESS green, FAILED red, RUNNING blue,
  PENDING grey, SKIPPED amber, CANCELLED orange); **click a node** to open its log.
- **Job List** — per task: State, Worker, Started, Finished, Elapsed, Exit code, and **Log**.
- **Task log** — a 960 px panel (at most 92% of the window) with the task's output, **Copy log**,
  **Download log** (`<run>-<task>.log`), and close by the X, a click outside, or **Esc**.
- **Timeline** — one bar per task across the run's time window, coloured by state, with its duration.
- **Run Log** — the driver's orchestration log followed by every task's output in one scrolling view.
  While the run goes it streams every 2 seconds (**LIVE**, click to **PAUSE**); a finished run's log reads
  **COMPLETE**. **Download** saves every section in one file.

The page follows a running run every 3 seconds.

## Schedules

DAGs with a cron schedule: DAG, **Cron**, **Runs** (the plain-English description, `—` when the cron uses
a form that cannot be described exactly), **Next run**, **Catchup**, Tasks. The cron is evaluated in the
masters' time zone (shown when you hover the next run) and displayed in yours. Search matches the DAG,
the cron and the description.

## Settings

Settings are administrator-only, and the server enforces it. The left bar lists the pages; the bottom
shows the version of the build. **Back to workflows** returns to DAGs; **Sign out** is in the header.

### Dashboard

**Masters** / **Workers** counts, and the last hour of **CPU Usage (%)** and **JVM Heap Used (MB)** per
node, sampled in 10-second buckets and refreshed every 15 seconds. **all / masters / workers** selects
which nodes are plotted. Until two samples exist the page shows *Collecting metrics…*.

### Topology

- **ZooKeeper** — each server of the connect string with role, status (and the error when it is down),
  version, connections, latency, znodes and zxid, plus this master's session state.
- **Masters** — node, host, admin and internal ports, Leader / Follower, ready; **Workers** — node, host,
  internal port, ready. Refreshed every 10 seconds.
- **Log** — a live viewer of that node's log, fetched through whichever master you are on: **LIVE** /
  **PAUSED**, closed by the X or **Esc**.
- **Enter Maintenance** / **Exit Maintenance** — asks first, then opens or closes the
  [maintenance window](cluster-maintenance.md); a banner shows while it is open.

### KMS

- **Generate Key** — asks for confirmation (**Cancel** / **Create Key**). A duplicate or invalid id
  (allowed: 1-128 letters, digits, `_ . : -`) opens an **Operation Failed** dialog with the reason.
- The list shows each key's version and *Last Rotated: <date>*, or *Never rotated · created <date>* for a
  key still at version 1. The refresh button reloads it.
- **Manage Key** — the version lifecycle (each version's state and creation time, newest first), when it
  was last rotated, and **Rotate Key Now** (asks first). **Back to List** returns.
- **Change Admin Password** — current password, new password (at least 6 characters) twice.

See [Encryption &amp; Key Management](kms.md).

### IAM

- **Users** — groups (**Join** adds the user to a group), keys and actions. The built-in `admin` is marked
  *built-in* and cannot be deleted. **New User** opens a side panel (960 px).
- **New Token** — optional expiry date (the end of that day, your time). The panel shows the **Access Key
  ID**, **Secret Access Key** and the **Token** (`OTOK…`) once, with **Download Credentials (CSV)**. The
  token and the key pair authenticate REST calls with that user's rights (see
  [Authentication &amp; Authorization](auth-authz.md#long-lived-credentials-user-token-and-access-key)).
  Each key shows *Expiry: <date>*, *Expired <date>* or *No expiry*, a download button and **Revoke**
  (asks first; refused at once on every master).
- **Groups** — **New Group**; click a row to expand its members and policies: **Attach Custom** attaches a
  policy, the X on a member removes it, the X on a policy detaches it (both ask first); a group with no
  policy reads *no policies*. Removing the last administrator path is refused with the reason.
- **Policies** — **New Policy** (side panel: a name, a **Governance Template** that fills the JSON, the
  JSON itself; a taken name or malformed JSON is refused with the reason). **Edit** opens the structured
  editor: statements (**Add Statement**, remove), Effect, Sid, actions (toggle the listed ones or type any
  other and press Enter), resources (comma-separated). **Delete** asks first; `AdministratorAccess`
  cannot be deleted.
- **STS Sessions** — generate temporary credentials from a parent access key with a lifetime (hours /
  minutes) and session name; the banner shows them once (show / hide, copy, **Download CSV**) with the
  expiry and time left. The table lists each session's parent key, name, creation time, time left and
  Active / Expired, with download and revoke; **Clear Expired** removes the expired ones.

See [Identity &amp; Access Management](iam.md).

### SSO

Administrators only (reads included). Tabs **OpenID Connect**, **SAML 2.0**, **LDAP / AD** (a dot marks a
provider that is active) and **Identity mapping**. Switches turn a provider or option on and off; a secret
field shows *set* / *not set*, is never read back, and keeps the stored value unless you type a new one;
an emptied field is removed (the properties file applies again). **Save (n)** counts the changed fields
and applies them on every master without a restart. **Test the connection** / **Check the configuration**
/ **Test the bind** report what the provider answered. On the SAML tab: download **This cluster's SP
metadata**, **Generate an SP keypair**, and paste the provider's metadata to **Read the provider's
metadata** (entity ID, sign-on URL and certificate are filled from it). See [Single Sign-On](sso.md).

### Connections

The list (secrets as `***`), **Edit** (fills the form; `key=***` keeps a secret, `key=` removes a key) and
**Delete** (browser confirmation). The form: id (no `.`, `}`, `/`, `\` or whitespace), type (`s3`, `git`,
`secret`, `jdbc`, `kafka`, `generic`), description, `key=value` properties. See [Connections](connections.md).

### Git Sync

**Sync now**, **Add repository** (name, subpath, URL, **Ref type** branch / tag, branch or tag name, and a
credential connection for a private repository; **Cancel** closes the form). Each repository shows its
ref, subpath and last sync time (*not yet* before the first), **Edit** (prefills the form) and **Remove**
(asks first; synced DAGs are kept). Files that were not registered in the last sync are listed under the
repository with the reason. See [Git Sync](git-sync.md).

### DAG Bundles

Upload a `.zip` of YAML / Python DAGs and/or a Java DAG jar under a bundle name (both are required).
Each bundle shows its DAG count, jar, upload time and owner; entries that failed to compile or whose id is
taken are listed under it. **Delete** asks first and removes the bundle's DAGs. See [DAG Bundles](bundles.md).

### Backup & Restore

- **Enable** / **Disable**, and **Back up now** (disabled while off) — the stored backup id is shown.
- **Automatic backup schedule** — a cron (a malformed one is refused with the reason), **Save**, **Clear**,
  and the **Next fire** time.
- **Backup S3 target** — the job-log S3 default or a stored S3 connection.
- **Backups in S3** — each backup with the time it was taken and **Restore** (browser confirmation; the
  restored state reaches every master and worker).

See [Backup &amp; Restore](backup.md).
