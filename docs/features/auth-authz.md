# Authentication &amp; Authorization

## Authentication

Every request to the admin API is authenticated with a **JWT**.

- **Signing in** — `POST /api/v1/auth/login` with a `user` / `password` body returns a short-lived access token, a longer-lived refresh token, and flags (`isAdmin`, `requirePasswordChange`). `POST /api/v1/auth/refresh` exchanges a valid refresh token for a fresh access token. Access and refresh expiry are configurable (`kiok.admin.token.access.expiry.ms` / `kiok.admin.token.refresh.expiry.ms`).
- **Presenting the token** — the access token is sent in the `Authorization` header, accepted under **either** scheme: `Authorization: Bearer <jwt>` or `Authorization: Token <jwt>`. The Java and Python SDKs (`KiokClient`) and `submit.sh --token` sign in this way and send `Token <jwt>`; the web admin UI sends `Bearer <jwt>`.

- **Signing in with an identity provider** — when single sign-on is configured, the console offers an OIDC or SAML button, and this same login route also accepts LDAP/Active Directory credentials. A federated caller gets the same access token every route already accepts, and is authorized by the cluster groups its provider groups map onto. See [Single Sign-On](sso.md).

**How local passwords are stored.** As PBKDF2-HMAC-SHA256, in a self-describing format that records the algorithm, iteration count and salt with the hash (`kiok.auth.password.hash.iterations`, default 600000). Digests written by earlier releases are still verified and are re-hashed in place on the next successful login, so an upgrade needs no password resets.

!!! note "Access keys & STS"
    Long-lived access-key pairs (an access-key id, a secret, and a user token) and STS-style temporary credentials can be **issued, downloaded, and revoked per user** through the IAM page — see [Identity &amp; Access Management](iam.md). The admin API request path itself validates the JWT above; obtain one with a user/password login.

## Authorization

Every authenticated request is mapped to an **action** and a **resource**, then evaluated against all policies that apply to the caller — those attached to the user plus those inherited from its groups.

- Evaluation is **default-allow** with **most-specific-match wins**: the policy statement whose resource pattern most closely matches the request decides the outcome.
- An explicit **`Deny`** always beats an `Allow` at the same specificity.
- The **admin user bypasses** policy checks entirely.
- A **federated identity has no local user account**, so only its mapped groups' policies apply — there is nothing user-specific to attach a policy to. See [Single Sign-On](sso.md).

DAG operations carry resource names like `dag:<origin>:<ref>/<path>`, so policies can scope access down to an individual DAG, a git repository, or a bundle. See [Identity &amp; Access Management](iam.md) for managing users, groups, and policies.

## Leader routing

Requests that mutate cluster state are always executed on the leader. If such a request lands on a follower Master it is automatically routed to the leader, so authentication and authorization decisions are made against one consistent source of truth.
