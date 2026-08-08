# Authentication &amp; Authorization

## Authentication

Every request to the admin API is authenticated with a **JWT**.

- **Signing in** — `POST /api/v1/auth/login` with a `user` / `password` body returns a short-lived access token, a longer-lived refresh token, and flags (`isAdmin`, `requirePasswordChange`). `POST /api/v1/auth/refresh` exchanges a valid refresh token for a fresh access token. Access and refresh expiry are configurable (`kiok.admin.token.access.expiry.ms` / `kiok.admin.token.refresh.expiry.ms`).
- **Presenting the token** — the access token is sent in the `Authorization` header, accepted under **either** scheme: `Authorization: Bearer <jwt>` or `Authorization: Token <jwt>`. The Java and Python SDKs (`KiokClient`) and `submit.sh --token` sign in this way and send `Token <jwt>`; the web admin UI sends `Bearer <jwt>`.

!!! note "Access keys & STS"
    Long-lived access-key pairs (an access-key id, a secret, and a user token) and STS-style temporary credentials can be **issued, downloaded, and revoked per user** through the IAM page — see [Identity &amp; Access Management](iam.md). The admin API request path itself validates the JWT above; obtain one with a user/password login.

## Authorization

Every authenticated request is mapped to an **action** and a **resource**, then evaluated against all policies that apply to the caller — those attached to the user plus those inherited from its groups.

- Evaluation is **default-allow** with **most-specific-match wins**: the policy statement whose resource pattern most closely matches the request decides the outcome.
- An explicit **`Deny`** always beats an `Allow` at the same specificity.
- The **admin user bypasses** policy checks entirely.

DAG operations carry resource names like `dag:<origin>:<ref>/<path>`, so policies can scope access down to an individual DAG, a git repository, or a bundle. See [Identity &amp; Access Management](iam.md) for managing users, groups, and policies.

## Leader routing

Requests that mutate cluster state are always executed on the leader. If such a request lands on a follower Master it is automatically routed to the leader, so authentication and authorization decisions are made against one consistent source of truth.
