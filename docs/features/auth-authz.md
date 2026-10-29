# Authentication &amp; Authorization

## Authentication

Every request to the admin API is authenticated with a **JWT**, or with one of the long-lived credentials the IAM page issues (below).

- **Signing in** — `POST /api/v1/auth/login` with a `user` / `password` body returns a short-lived access token (15 minutes by default), a longer-lived refresh token (24 hours), and flags (`isAdmin`, `requirePasswordChange`). `POST /api/v1/auth/refresh` exchanges a valid refresh token for a fresh pair. Expiry is configurable (`kiok.admin.token.access.expiry.ms` / `kiok.admin.token.refresh.expiry.ms`). A wrong password answers `401` with `invalid credentials`, which the login page shows.
- **Presenting the token** — the access token is sent in the `Authorization` header, accepted under **either** scheme: `Authorization: Bearer <jwt>` or `Authorization: Token <jwt>`. The Java and Python SDKs (`KiokClient`), `submit.sh --token` and the web admin UI all send `Token <jwt>`.
- **Access and refresh tokens are not interchangeable.** Each token carries its kind (`typ`: `access` or `refresh`). The API accepts only access tokens, and `/auth/refresh` accepts only refresh tokens — so a stolen access token cannot be turned into a day-long session, and a refresh token cannot call the API.
- **Deleting a user signs them out at once.** Every request checks that the token's user still exists, on every master, so a deleted user's access token is refused immediately rather than at its expiry, and their refresh token no longer renews. (A federated user has no local account; their token is bounded by its short lifetime and the provider's own controls.)
- **Signing out revokes the session on the server.** `POST /api/v1/auth/logout` revokes the access token in the `Authorization` header and the `refreshToken` in the JSON body, and answers `{"revoked": <count>}`. Every token has its own id (`jti`), so only that session ends; the user's other sessions stay signed in. The revocation reaches every master, so a copy of either token taken from the browser is refused at once rather than at its expiry. The console's **Sign out** button calls it. The route needs no valid access token, so an expired access token does not stop the refresh token from being revoked.

- **Signing in with an identity provider** — when single sign-on is configured, the console offers an OIDC or SAML button, and this same login route also accepts LDAP/Active Directory credentials. A federated caller gets the same access token every route already accepts, and is authorized by the cluster groups its provider groups map onto. See [Single Sign-On](sso.md).

**How local passwords are stored.** As PBKDF2-HMAC-SHA256, in a self-describing format that records the algorithm, iteration count and salt with the hash (`kiok.auth.password.hash.iterations`, default 600000). Digests written by earlier releases are still verified and are re-hashed in place on the next successful login, so an upgrade needs no password resets.

### Long-lived credentials: user token and access key

A key issued on the IAM page (see [Identity &amp; Access Management](iam.md)) authenticates REST calls
directly, without a login:

- `Authorization: Token OTOK…` — the key's **user token** (what `KIOK_USER_TOKEN` / `submit.sh --token` send);
- `Authorization: AccessKey <access-key-id>:<secret>` — the key pair (`submit.sh --accesskey/--secretkey`).

Only an **active, unexpired** key of an existing user is accepted; the caller gets exactly that user's
rights. Revoking the key (or deleting the user) refuses it at once on every master, and a key past its
expiry date is refused (`401`). A JWT is still what the console and the SDK logins use.

## Authorization

### Settings are for administrators

Everything under **Settings** in the console — IAM, KMS, STS sessions, connections, SSO, git sync, DAG
bundles, backup and restore, logs, nodes, metrics and maintenance mode — is administrator-only, for
reading as well as writing, and the server enforces it: a non-administrator calling any of those
routes gets `403 administrator access required`, whatever the console shows. Hiding the Settings link
is a convenience; the server check is what stops a user from, for example, adding themselves to
`admin-group` through the API. Two exceptions: anyone signed in may read the maintenance-mode flag
(`GET /api/v1/admin/maintenance`, which the console shows as a banner), and a user may download their
own access key (`GET /api/v1/iam/download-key/<their id>/<key id>`).

The SSO settings (`/api/v1/sso/*`) are administrator-only for reads too, and — like every route except
`GET /auth/session`, `GET /whoami` and `POST /auth/change-password` — are refused until the factory
`admin` password has been changed, so a default-password session cannot point the cluster at an identity
provider of its choosing.

An administrator is any user whose policies (directly or through a group) include
`AdministratorAccess`; for a federated user, the groups their provider maps them to. The cluster
refuses a change that would leave no local user with `AdministratorAccess` — detaching it from
`admin-group` while that group holds the only administrators, or removing the last administrator from
the group — with `400 ... would leave no administrator`.

### DAGs and runs

**Who may operate a DAG.** Running a DAG, cancelling or re-running its runs, deleting it, and registering
a DAG under a name that is already taken are allowed to:

| Caller | DAGs they may operate |
|---|---|
| Administrator | every DAG |
| Any other user | only DAGs **they registered themselves** — origin `manual`, i.e. posted as YAML (`POST /api/v1/dags`, the console's **Register DAG**, or the Python / Java SDK, which send the same YAML) — whose owner is them |

DAGs synced from a **git repository** or loaded from a **bundle** arrive many at once, and their owner is
whoever set up the repository or uploaded the bundle, not the author of a given DAG. They are run,
cancelled and deleted by administrators. A DAG with no recorded owner is treated the same way. Anyone else
gets `403` with the reason — who owns the DAG, or which repository or bundle it came from — and the
console does not show the **Trigger run**, **Re-run**, **Cancel run** and delete controls for those DAGs.

Reading is not affected: listing and viewing DAGs and runs stays governed by `dag:Read` below. The
policies below still apply on top of this rule — a `Deny` on `dag:Trigger` keeps even the owner from
running their DAG.

Every authenticated request is mapped to an **action** and a **resource**, then evaluated against all policies that apply to the caller — those attached to the user plus those inherited from its groups.

- Evaluation is **default-allow** with **most-specific-match wins**: the policy statement whose resource pattern most closely matches the request decides the outcome.
- An explicit **`Deny`** always beats an `Allow` at the same specificity.
- **Administrators bypass** policy checks entirely.
- A run, its arguments and its task logs are governed by the run's DAG: reading them needs `dag:Read` on that DAG (`GET /api/v1/runs/{id}` and `.../tasks/{task}/log` answer `403` otherwise). `GET /api/v1/runs?limit=N` (default 120, at most 1000) returns the newest runs across every DAG the caller may read, newest first; the console's Executions page loads with that one request. `GET /api/v1/runs?dag=<id>&limit=N` returns one DAG's newest runs the same way (the DAG page's Runs tab). The run of a DAG that has since been deleted is visible to administrators only: a non-administrator can neither list (`GET /dags/{id}/runs`), read, nor cancel it (`403`), and a task log of an unknown run answers `404`.
- A **federated identity has no local user account**, so only its mapped groups' policies apply — there is nothing user-specific to attach a policy to. See [Single Sign-On](sso.md).

DAG operations carry resource names like `dag:<origin>:<ref>/<path>`, so policies can scope access down to an individual DAG, a git repository, or a bundle. See [Identity &amp; Access Management](iam.md) for managing users, groups, and policies.

## Leader routing

Requests that mutate cluster state are always executed on the leader. If such a request lands on a follower Master it is automatically routed to the leader, so authentication and authorization decisions are made against one consistent source of truth. This includes SSO settings: a change saved through a follower is applied on the leader and replicated back, instead of being overwritten by the leader's next snapshot.

## Errors

A request the server refuses because of its input — an unknown user, a duplicate name, an invalid
value, a DAG definition that does not parse — answers `400` (or `404` for something that does not
exist) with an `error` message, which the console shows. `500` is reserved for failures of the server
itself.
