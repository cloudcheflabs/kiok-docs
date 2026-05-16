# Encryption &amp; Key Management (KMS)

kiok encrypts cluster state at rest with a built-in distributed key management system — no external KMS service required.

- Uses **envelope encryption**: data is encrypted with a per-purpose data key, which is itself wrapped by a master-derived key (AES-256-GCM).
- The **master key** is supplied as the environment variable `KIOK_MASTER_KEY` (32+ characters) and never written to a config file. Losing it means losing access to all encrypted state.
- Everything sensitive is encrypted with a named KMS key: the metadata store (DAGs, runs), the IAM store, the connection store, job logs, and — optionally — the internal protocol traffic between nodes.
- **Key rotation** is supported — a new key version encrypts future writes while older versions stay available to decrypt existing data. Rotate from the admin UI's **Settings → KMS** page.
- Key state lives on the leader and is synchronised to every Master and Worker, so any node can decrypt what it needs.
