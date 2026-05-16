# Authentication &amp; Authorization

## Authentication

kiok accepts two credential types on the admin API:

- **JWT** — sign in with a user/password to receive a short-lived access token and a longer-lived refresh token. The access token is presented as `Authorization: Token <jwt>`; it is refreshed transparently. Access and refresh expiry are configurable (`kiok.admin.token.*`).
- **Access keys** — a long-lived access-key pair issued to a user, presented as `Authorization: AccessKey <ak>:<sk>`. Used by the Java/Python SDKs and `submit.sh` for non-interactive access. STS-style temporary credentials are also supported.

## Authorization

Every authenticated request is mapped to an **action** and a **resource**, then evaluated against all policies that apply to the caller — those attached to the user plus those inherited from its groups.

- Evaluation is **default-allow** with **most-specific-match wins**: the policy statement whose resource pattern most closely matches the request decides the outcome.
- An explicit **`Deny`** always beats an `Allow` at the same specificity.
- The **admin user bypasses** policy checks entirely.

DAG operations carry resource names like `dag:<origin>:<ref>/<path>`, so policies can scope access down to an individual DAG, a git repository, or a bundle. See [Identity &amp; Access Management](iam.md) for managing users, groups, and policies.

## Leader routing

Requests that mutate cluster state are always executed on the leader. If such a request lands on a follower Master it is automatically routed to the leader, so authentication and authorization decisions are made against one consistent source of truth.
