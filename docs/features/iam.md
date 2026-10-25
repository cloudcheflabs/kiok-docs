# Identity &amp; Access Management (IAM)

kiok provides an AWS IAM-style access control system for the admin API and DAG operations.

- **Users, groups, and policies** are managed from the admin UI's **Settings → IAM** page. A user inherits the policies of every group it belongs to.
- **Policies** follow the standard AWS IAM JSON format — actions, resources, `Allow` / `Deny` — with wildcard matching. Evaluation is **default-allow**: most-specific-match wins (specificity measured by the non-wildcard length of the matching resource pattern), an explicit `Deny` breaks ties at equal specificity, and the admin user bypasses policy checks. A user with no matching statement is allowed — an ungoverned user has full access until a policy restricts it.
- **DAG-level authorization** — DAG operations are authorized against resource names of the form `dag:<origin>:<ref>/<path>`, so a policy can grant or deny access to specific DAGs, repositories, or bundles.
- **Access keys** — long-lived access-key pairs (an access-key id, a secret, and a user token) can be issued, downloaded as CSV, and revoked per user. **STS-style temporary credentials** (assume-role sessions minted from a parent access key) are also supported. A key's user token (`Authorization: Token OTOK…`) and its pair (`Authorization: AccessKey <id>:<secret>`) authenticate REST calls with that user's rights while the key is active and unexpired; revoking it takes effect at once on every master (see [Authentication &amp; Authorization](auth-authz.md#long-lived-credentials-user-token-and-access-key)). An STS session's CSV download carries its expiry as an ISO-8601 UTC timestamp.
- On the first cluster bootstrap a default admin user, group, and full-access policy are created automatically — and never recreated afterward.

## The IAM page

Settings → IAM is administrator-only; the server refuses every IAM route, reads included, to anyone
else (a user may still download their own access key). On the page:

- **Policies** open in a structured editor. A policy written through the API keeps every field it has —
  `Columns`, `Condition`, a single-string `Action` or `Resource` such as the default
  `AdministratorAccess` (`"*"`) — when it is opened and saved in the editor. Besides the listed actions,
  any other action can be typed in. A save the server refuses keeps the editor open with the reason.
- **New Policy** refuses a name that already exists (`409`) instead of replacing that policy.
- **Remove member** and **Detach policy** ask for confirmation, like every other removal. A change that
  would leave no local user with `AdministratorAccess` is refused — see
  [Authentication &amp; Authorization](auth-authz.md#settings-are-for-administrators).
- An access key's **expiry date** means the end of that day in the browser's time zone (23:59:59.999),
  not UTC midnight.
- A key shows **Expiry: <date>**, **Expired <date>** once past it, or **No expiry**.
- The built-in `admin` user is marked **built-in** and cannot be deleted.
- Side panels (new user, group, policy, token) open 960 px wide (at most 92% of the window).
- STS sessions are listed on every master, not only the leader (a sync bug used to stop every IAM
  change reaching the followers while any STS session existed).
- Revoking a key, deleting a user and downloading credentials report failures on the page.

See the [Admin UI Guide](admin-ui.md#iam) for every control on the page.

For how requests are authenticated and how that flows into authorization, see [Authentication &amp; Authorization](auth-authz.md).
