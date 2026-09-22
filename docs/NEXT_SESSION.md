# GalleyPack next-session worklist

- [x] Materialize the full review-directory package shape from a declarative manifest.
- [x] Add checksum manifests with optional detached HMAC integrity receipts.
- [x] Add streaming directory-tree hashing with benchmarked file-count limits.
- [x] Add adapter plans for reproducible TAR and ZIP output.
- [x] Add object-store adapters with offline contract fixtures and no implicit credentials.
- [x] Add claim-delta semantic adapters while preserving a complete deterministic local path.

Completed 2026-08-14. Hardening contracts materialize safe review trees, hash up to 100,000 declared files, produce detached HMAC receipts, require reproducible TAR/ZIP adapter evidence, validate credential-free object-store adapters, and always retain the deterministic local claim delta.

These are library contracts and adapter plans, not executed archive/object-store
implementations. See [contracts](contracts.md) and the
[2026-09-22 audit](audits/repository-hardening.md) for corrected bounds and gaps.

The [directory/recovery follow-up](audits/directory-and-recovery.md) closes the two
original open storage findings. Use the updated runtime pin and `recover --dry-run`
before repairing interrupted state; see [operator guide](recovery.md).

Kujo 1.5.0 adoption, confined tree hashing and native byte counting are recorded in
[audit update](audits/kujo-1.5-adoption.md). No repository-local audit item remains open.
