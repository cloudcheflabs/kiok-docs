# Identity &amp; Access Management (IAM)

kiok provides an AWS IAM-style access control system for the admin API and DAG operations.

- **Users, groups, and policies** are managed from the admin UI's **Settings → IAM** page. A user inherits the policies of every group it belongs to.
- **Policies** follow the standard AWS IAM JSON format — actions, resources, `Allow` / `Deny` — with wildcard matching. Evaluation is most-specific-match wins, an explicit `Deny` breaks ties, and the admin user bypasses policy checks.
- **DAG-level authorization** — DAG operations are authorized against resource names of the form `dag:<origin>:<ref>/<path>`, so a policy can grant or deny access to specific DAGs, repositories, or bundles.
- **Access keys** — long-lived access-key pairs can be issued per user for programmatic access (the SDKs and `submit.sh`). **STS-style temporary credentials** are also supported.
- On the first cluster bootstrap a default admin user, group, and full-access policy are created automatically — and never recreated afterward.

For how requests are authenticated and how that flows into authorization, see [Authentication &amp; Authorization](auth-authz.md).
