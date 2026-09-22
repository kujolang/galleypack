# Changelog

## Unreleased

- Added bounded native directory pages and corrected continuation for prefix-related IDs.
- Added durable write intents, per-ID OS locks, idempotent `recover`/dry-run, legacy
  orphan repair and crash-injection gates; conflicting evidence is never overwritten.
- Doctor and validation now detect creation-event inconsistency. Runtime prerequisite
  advances to the CI-pinned Kujo 1.4.0 build with required filesystem primitives.

- Made record locks and immutable record/event publication exclusive under contention.
- Made dry runs side-effect free; validate CLI/config shapes and calendar dates.
- Bound filtered/corrupt record scans and retained page bytes with continuation cursors;
  fail incomplete whole-state validation instead of certifying partial coverage.
- Enforced UTF-8 byte limits, validated complete review manifests before writes,
  rejected intermediate tree symlinks and unsafe adapter paths, and fixed >10,000-file hashing.
- Corrected schema compatibility, portable launcher resolution and library capability docs;
  added concurrency, bounds, CLI and security regression gates.

- Standardized README badge ordering and repository-local artifact ignores.
- Kept Loop Engineering evidence available locally while removing it from published source.

## 0.2.0 - 2026-08-14

- Preserved validation compatibility with immutable 0.1.0 records while emitting 0.2.0 records.
- Prevented audit-history conflicts from leaving partial records and added clean-retry regression coverage.
- Enforced safe relationship/package references, non-self relationships, bounded artifact sets, state compatibility, managed-directory safety, and stricter immutable-record validation.
- Added modular runtime architecture, strict package contracts, checksum drift validation, and deterministic version comparison.
- Added atomic storage/export, per-record locks, bounded pagination, configuration, structured errors, and adversarial security tests.
- Added pinned-runtime CI, production-readiness documentation, and a future enhancement worklist.

## 0.1.0 - 2026-08-14

- Initial Kujo-native release with working local records, validation, contracts, fixtures, and safety boundaries.
