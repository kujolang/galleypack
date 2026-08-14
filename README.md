# GalleyPack

[![Version](https://img.shields.io/badge/version-0.2.0-black)](VERSION)
[![CI](https://github.com/kujolang/galleypack/actions/workflows/validate.yml/badge.svg)](https://github.com/kujolang/galleypack/actions/workflows/validate.yml)
[![License](https://img.shields.io/badge/license-MIT-lightgrey)](LICENSE)
[![built with Kujo](https://img.shields.io/badge/built%20with-Kujo-white.svg)](https://github.com/kujolang/kujo)

GalleyPack is a Kujo-native production packager that binds editorial artifacts,
lineage, evidence, reviews, and package versions to exact SHA-256 checksums. It
replaces mutable “latest” folders with deterministic records that independent
operators can compare and verify offline.

## Readiness posture

GalleyPack is ready for serious local package-control workflows: immutable
records, atomic writes, per-record locks, bounded files and queries, exact
artifact hashes, deterministic version comparison, and fail-closed drift
validation. It does not modify source artifacts, average away missing reviews,
interpret review as approval, or publish.

See the [production review](docs/PRODUCTION_READINESS_REVIEW.md) and
[next-session worklist](docs/NEXT_SESSION.md).

## Quick install

```bash
git clone https://github.com/kujolang/galleypack.git
cd galleypack
export KUJO_BIN=/absolute/path/to/kujo
export PATH="$PWD/bin:$PATH"
galleypack --version --json
galleypack doctor --json
```

Kujo 1.0.1 or newer is required. No hosted service or sibling tool is required.

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
| `doctor`, `version` | Report health and runtime compatibility. |

Common flags include `--state`, `--config`, `--input`, `--actor`, `--timestamp`,
`--id`, `--other-id`, `--path`, `--type`, `--after`, `--limit`, `--output`,
`--force`, `--dry-run`, and `--json`. Files are capped at 64 MiB in the core;
records and inputs are capped at 1 MiB; queries are capped at 1,000 records.

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

CI builds a pinned Kujo runtime and runs the identical gate.
