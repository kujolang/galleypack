# Contracts

Contract 1.0.0. GalleyPack owns Artifact, Artifact Manifest, Artifact Relationship,
Claim Delta, Review Requirement, Review Index, Package Version and Drift Report.
Records carry schema/tool/contract versions, stable IDs, actor, timestamp,
provenance, command and payload. Record schema 1.0.0 and tool versions 0.1.0 and
0.2.0 remain readable. Input payloads accept the existing 1.x schema-major
contract; safe unknown payload metadata is preserved. The JSON envelope is
`ok/data/error/error_code/tool_version/contract_version`.

## CLI and storage

Supported commands and flags are listed by `./bin/galleypack --help`. Successful
operations exit 0, operational failures exit 1, and usage errors exit 2. Missing
flag values and trailing invalid `--version` options are usage errors.
`--version --json` honors the JSON envelope. `--dry-run` validates mutations and
returns previews without initializing state or writing exports. Config accepts
only `state` (nonempty string), `actor` (bounded nonempty string), and `limit`
(integer 1..1000); CLI settings override config.

`report`, `history` and `export` return record pages, preserving the historical
meaning of `history`; raw creation events live in `<state>/history/`. Pages add
`next_after`, and record listings add `scanned`, `directory_entries_buffered` and
`directory_entries_examined`. Pass `--after <next_after>` until
`truncated` is false, including pages containing only warnings or nonmatching
records. A cursor is a filename stem; the `.json` suffix is restored for exclusive filename
comparison. This preserves filename ordering for IDs that prefix other IDs. It is
never treated as a path.
The native directory heap retains at most 1,001 names; enumeration remains O(N).
Directory entries must have UTF-8 names; native enumeration errors fail explicitly.
Each page parses up to 1,000 candidates, returns up to the requested record limit,
and retains at most 4 MiB of source record text. A full page may conservatively
report truncation if more candidates remain, even when they do not match a filter.
Invalid filenames are warnings and can be passed back as continuation cursors.
Whole-state `validate` and `doctor` never certify a truncated inspection: they exit
1 with `inspection_incomplete`. For larger states, paginate `report` and validate
each returned ID; preserve and review all warnings.

UTF-8 byte limits: config/metadata 64 KiB, input/record 1 MiB, core artifact 64 MiB,
rendered export 8 MiB. A page is not silently discarded to fit a rendered export:
an oversized rendering fails with `output_too_large`. Record/event filenames and
JSON layouts remain unchanged. Additive `transactions/` intent files and persistent
`writer-locks/` lock inodes implement replay; temporary compatibility markers remain
at the original `locks/<id>.lock` paths. Interrupted publication returns
`recovery_required`; reads never expose a pending transaction as a complete record.
Doctor/validation diagnose missing or conflicting creation events. `recover` repairs
missing events and replays intents, but never overwrites conflicting evidence.
`--force` can remove legacy locks during recovery only after old writers stop.
See [recovery and migration](recovery.md).

## Optional library APIs

Import `src.hardening` directly; these functions are not CLI subcommands:

- `materialize_review_package(manifest, output_root)`: 1..1,000 entries, 1 MiB per
  UTF-8 content string, 64 MiB total. Paths must be distinct safe relative paths
  without file/directory collisions. Invalid input creates no output tree.
  Input order remains significant to the returned tree digest.
- `checksum_manifest(entries, signing_key)`: 1..10,000 unique checksum entries;
  preserves entry order in canonical JSON. Empty string requests unsigned SHA-256;
  a nonempty key must be a string containing at least 16 UTF-8 bytes. Other types
  fail instead of silently downgrading to unsigned output. HMAC is a symmetric
  integrity receipt, not a public-key signature or a verification API.
- `streaming_tree_hash(root, relative_paths, maximum_files)`: 1..100,000 distinct
  declared files; lexicographic path ordering; 64 MiB per file and 1 GiB total.
  File contents use the runtime's streaming hash; entry metadata is retained in
  memory. The public checksum-manifest limit does not constrain this tree path.
- `archive_adapter_plan(format, entries, adapter)`: deterministic TAR/ZIP **plans**
  for 1..10,000 safe distinct relative file paths, normalized timestamps/owners and
  declared adapter capabilities. Does not create an archive.
- `object_store_conformance(adapter)`: checks offline fixture declarations,
  explicit credential policy and put/head/get operation names; makes no requests.
- `local_claim_delta` and `semantic_delta`: deterministic local comparison, with
  optional caller-provided redacted offline semantic receipts. No model calls.

No runtime package dependencies beyond Kujo are introduced. Supported execution
is POSIX; CI builds pinned Kujo source. Runtime primitives remain part of the
trusted computing base.


The tested runtime pin is `cf785c0a7953717af16b657cda05b85d628144c5` (Kujo 1.4.0).
Runtime builds predating the required native APIs are no longer supported; storage
mutators resolve these capabilities before filesystem changes. This is an explicit
runtime prerequisite change, not a record-format migration.
