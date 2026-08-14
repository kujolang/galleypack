# Changelog

## 0.2.0 - 2026-08-14

- Preserved validation compatibility with immutable 0.1.0 records while emitting 0.2.0 records.
- Prevented audit-history conflicts from leaving partial records and added clean-retry regression coverage.
- Enforced safe relationship/package references, non-self relationships, bounded artifact sets, state compatibility, managed-directory safety, and stricter immutable-record validation.
- Added modular runtime architecture, strict package contracts, checksum drift validation, and deterministic version comparison.
- Added atomic storage/export, per-record locks, bounded pagination, configuration, structured errors, and adversarial security tests.
- Added pinned-runtime CI, production-readiness documentation, and a future enhancement worklist.

## 0.1.0 - 2026-08-14

- Initial Kujo-native release with working local records, validation, contracts, fixtures, and safety boundaries.
