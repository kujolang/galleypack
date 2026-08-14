# GalleyPack

Versioned editorial production packages with artifact checksums, lineage, reviews, freeze, and drift verification.

GalleyPack 0.1.0 is an independently installable, local-first Kujo tool. It requires no hosted service, Chain of Command, WebOps, or sibling Publishing House tool. The canonical entrypoint is `galleypack.kujo`; `bin/galleypack` contains no product logic.

## CLI

Commands: add; relate; evidence attach; review attach; claims compare; build; freeze; validate; diff; show; export; doctor; version; init; report; history. Run `./bin/galleypack help` for flags. Mutations require `--actor`; JSON input uses `--input`. Common flags include `--json`, `--dry-run`, `--state`, `--output`, `--config`, and `--force`. Exit codes: 0 success, 1 validation/operation failure, 2 usage error.

State defaults to `.galleypack/`. Immutable JSON records and append-only history use atomic writes. IDs reject traversal; symlinks and oversized inputs are rejected. See [contracts](docs/contracts.md), [security](docs/security.md), and [quickstart](examples/quickstart.md).

Test with `/Users/robertdevore/2026/Kujolang/kujo-repos/kujo/target/release/kujo run tests/test.kujo`, then run `./bin/galleypack doctor --json`.

0.1.0 covers the documented local records, fixtures, validation, checksums, deterministic fixed-time IDs, and structured export. It does not manufacture human judgment, consent, rights, approval, or causation. GalleyPack validates and freezes exact files but does not modify sources, approve, or publish.
