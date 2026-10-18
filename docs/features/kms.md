# Encryption &amp; Key Management (KMS)

kiok encrypts cluster state at rest with a built-in distributed key management system — no external KMS service required.

- Uses **envelope encryption**: data is encrypted with a per-purpose data key, which is itself wrapped by a master-derived key (AES-256-GCM).
- The **master key** is supplied as the environment variable `KIOK_MASTER_KEY` (32+ characters) and never written to a config file. Losing it means losing access to all encrypted state.
- Everything sensitive is encrypted with a named KMS key: the metadata store (DAGs, runs), the IAM store, the connection store, job logs, and — optionally — the internal protocol traffic between nodes.
- **Key rotation** is supported — a new key version encrypts future writes while older versions stay available to decrypt existing data. Rotate from the admin UI's **Settings → KMS** page.
- **Creating and rotating keys report real outcomes.** A key id must be 1–128 letters, digits, `_`, `.`, `:` or `-`. Creating an id that exists answers `400`; rotating an id that does not exist answers `404` (it used to create a new key silently); on a cluster whose KMS provider is not the built-in cluster KMS both answer `409` rather than claiming success. Key status lists each version's number, state and creation time — never key material. The KMS page is administrator-only.
- Key state lives on the leader and is synchronised to every Master and Worker, so any node can decrypt what it needs.
