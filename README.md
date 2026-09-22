# GalleyPack

[![Version](https://img.shields.io/badge/version-0.2.0-black)](VERSION)
[![License](https://img.shields.io/badge/license-MIT-lightgrey)](LICENSE)
[![built with Kujo](https://img.shields.io/badge/built%20with-Kujo-white.svg)](https://github.com/kujolang/kujo)
[![CI](https://github.com/kujolang/galleypack/actions/workflows/validate.yml/badge.svg)](https://github.com/kujolang/galleypack/actions/workflows/validate.yml)

GalleyPack is a Kujo-native production packager that binds editorial artifacts,
lineage, evidence, reviews, and package versions to exact SHA-256 checksums. It
replaces mutable “latest” folders with deterministic records that independent
operators can compare and verify offline.

## Production capabilities

GalleyPack is ready for serious local package-control workflows: immutable
records, atomic writes, per-record locks, bounded files and queries, exact
artifact hashes, declarative review-tree materialization, signed tree manifests,
bounded streaming hashes, reproducible archive adapter plans,
credential-free object-store conformance, deterministic and semantic claim
deltas, and fail-closed drift validation. It does not modify source artifacts,
average away missing reviews, interpret review as approval, or publish.

See the [production review](docs/PRODUCTION_READINESS_REVIEW.md) and completed
[hardening worklist](docs/NEXT_SESSION.md).

## Quick install

```bash
git clone https://github.com/kujolang/galleypack.git
cd galleypack
export KUJO_BIN=/absolute/path/to/kujo
export PATH="$PWD/bin:$PATH"
galleypack --version --json
galleypack doctor --json
```

Kujo 1.5.0 or a compatible later runtime is required.
Use the tested runtime revision `cc2d7dbb59a8dc05f00d629e100932f56f4062f6`
(or a compatible later build); the version string alone does not identify these APIs.
Older runtimes must be upgraded before writing. Existing 0.1.0/0.2.0 records do not
need conversion. No hosted service or sibling tool is required.

## Quick start

```bash
galleypack init --json
galleypack add --input fixtures/core.json --path article.md \
  --actor production-editor --timestamp 2026-08-14T12:00:00Z --json
galleypack freeze --input package.json --path manifest.json \
  --actor production-editor --json
galleypack validate --id package-example-v1 --json
galleypack diff --id package-example-v1 --other-id package-example-v2 --json
```

## Commands

| Command | Purpose |
| --- | --- |
| `add` | Bind an artifact record to an exact regular file. |
| `relate` | Record source, derivative, adaptation, variant, or replacement lineage. |
| `evidence attach`, `review attach` | Bind upstream evidence or completed review references. |
| `build`, `freeze` | Record an exact package version and manifest/package checksum. |
| `claims compare`, `diff` | Compare two immutable records and artifact hashes. |
| `validate` | Re-hash bound files and fail on missing files or byte drift. |
| `show`, `report`, `history`, `export` | Inspect and emit bounded package evidence. |
| `recover` | Preview or replay interrupted transactions and repair missing legacy creation events. |
| `doctor`, `version` | Report health and runtime compatibility. |

Common flags include `--state`, `--config`, `--input`, `--actor`, `--timestamp`,
`--id`, `--other-id`, `--path`, `--type`, `--after`, `--limit`, `--output`,
`--force`, `--dry-run`, and `--json`. Files are capped at 64 MiB in the core;
records and inputs are capped at 1 MiB of UTF-8 bytes. Query pages inspect at most
1,000 candidate records and retain at most 4 MiB of record text. Continue with
`--after <next_after>` while `truncated` is true. Whole-state inspection fails
explicitly when its bounds prevent a complete result.

State defaults to `.galleypack/`. Traversal, symlinks, secret-shaped fields,
malformed JSON, schema-major mismatch, duplicate IDs, concurrent duplicate
writes, unsafe overwrite, and artifact drift fail closed. See
[contracts](docs/contracts.md) and [security](docs/security.md).

## Project structure and verification

`galleypack.kujo` is the canonical entrypoint. Runtime logic lives under
`src/`; tests, fixtures, schemas, scripts, and docs are isolated by purpose.

```bash
bash scripts/validate.sh
```

CI builds a pinned Kujo runtime and runs the identical gate. The launcher uses
`KUJO_BIN`, then PATH, then a sibling Kujo release build. Optional review-tree,
checksum, archive-plan, object-store and semantic helpers are imported from
`src.hardening`; see their [contracts](docs/contracts.md).

The [repository hardening audit](docs/audits/repository-hardening.md) records
baselines, fixes, compatibility notes, measurements and remaining limitations.

For interrupted writes, run `./bin/galleypack recover --state PATH --dry-run --json`,
then repeat without `--dry-run`. Follow `next_after` while `truncated` is true.
Legacy locks require stopping old writers and explicitly adding `--force`.
See [recovery and migration](docs/recovery.md).
