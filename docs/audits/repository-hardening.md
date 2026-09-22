**Runtime update:** Kujo 1.5.0 adoption and remaining local improvements are recorded
in [the release follow-up](kujo-1.5-adoption.md). Earlier pins below are historical.

# GalleyPack repository hardening — 2026-09-22

**Follow-up:** GP-13 and GP-14 were subsequently implemented at the user's request.
See [directory paging and crash recovery](directory-and-recovery.md) for updated
runtime requirements, compatibility, proof and final state. The original baseline
and measurements below remain historical evidence.

## Repository and scope

- Repository: `kujolang/galleypack`; branch: `main`.
- Starting SHA: `5a9ec247f943460aa759f1552ae14e48c7423d3e` (clean tree).
- Ending implementation SHA: **a6e6060fd4f8b56a6c8b6a322ae3c0dd6dabef1b**. The following audit-only
  commit records this receipt; its own self-referential SHA is intentionally not embedded.
- Purpose: offline editorial artifact/package records, exact file hashes, lineage,
  evidence/review assertions, immutable versions, drift checks and optional library helpers.
- Reviewed every tracked baseline file: seven source modules, entrypoint, launcher,
  five test suites, validation scripts, schemas, fixtures, example, CI, release metadata,
  instructions and documentation. No application network, model/provider, MCP, database,
  queue, hosted service or production subprocess interface exists.
- Runtime dependency: Kujo. CI pins `5059695d14d6726bc17fef55e0b95511624967cf`
  (1.0.2). No application packages were added. Baseline local runtime: 1.4.0;
  preserved executable SHA-256 `eeea79362ea8c89cb3e8fe0b34984a8588bc57a9eea829cd902e91d16d9a4787`.
- Known consumers inspected read-only: `kujo-workflows/scripts/publishing_house_fixture.kujo`
  checks record paths/IDs; its workflow generator and `kujo-agents` production-editor
  manifest reference GalleyPack capabilities; `kujo-skills/skills/kujo-galleypack-workflows`
  documents CLI operations. No sibling files were changed or consumer workflows executed.

## Baseline

`bash scripts/validate.sh` passed all 40 assertions in five suites, entrypoint check,
fixture/schema JSON parsing, launcher smoke checks, repository policy checks and whitespace
validation. Wall time was 2.45 s for that smaller original gate; this is not comparable to
an expanded gate containing large-state and process-contention tests.

New regressions run against unchanged source initially produced **1 pass / 9 failures**:
dry-run state writes, missing argument values, ignored version-alias options, impossible
calendar dates, duplicate manifest paths, partial output from invalid later entries,
file/directory collisions, unsafe archive-plan paths and intermediate tree symlinks.
The collision initially also raised an unstructured runtime error; the baseline probe
caught it so the remaining regressions could execute. See [baseline](evidence/baseline.txt)
and [pre-fix regressions](evidence/regressions-before.txt).

Additional source-backed defects were confirmed with targeted probes: a Unicode record
could be accepted then rejected by its reader; 10,001 tree files were hashed before a
10,000-entry receipt cap rejected them; corrupt listings accumulated all warnings. The
baseline 24-writer contention sample happened to pass; the nonexclusive-lock defect is
established by the runtime's `create_dir_all` implementation, not an invented failing race.

## Findings

| ID | Priority | Area | Finding / evidence | Action | Status |
|---|---|---|---|---|---|
| GP-01 | P0 | Data integrity | `create_dir` is recursive/idempotent, not exclusive; old check-then-create locks and overwrite-enabled record/event writes could race. | Atomic no-replace lock files, metadata, records and events; centralized cleanup with explicit cleanup failure. | Fixed; 24-process regression. |
| GP-02 | P1 | Failure semantics | Dry runs initialized state; invalid manifests wrote earlier entries before rejecting later ones. | Read-only state inspection; complete manifest preflight; export/init previews; explicit partial I/O receipts. | Fixed; invalid input has no output writes. |
| GP-03 | P1 | Security | Tree hasher checked root/leaf only; real intermediate symlink hashes an outside file. | Check every relative component; retain trusted-root/concurrent-mutation assumptions. | Fixed; [sealed security report](security/report.md), severity low in this deployment. |
| GP-04 | P1 | Correctness | Duplicate package paths overwrite prior bytes yet return both hashes; directory collisions throw. | Shared path-set validation and exclusive package reservation/publication. | Fixed. |
| GP-05 | P1 | Resource/output | Result count did not bound corrupt warnings or filtered scans; whole-state validation could succeed when truncated. | 1,000-candidate and 4 MiB retained-source budgets, cursor, explicit incomplete inspection. | Fixed; directory-name allocation remains GP-13. |
| GP-06 | P1 | Byte limits | `len(string)` counts characters; 1,048,704-byte Unicode record saved successfully but failed read. | Exact UTF-8 counts compatible with 1.0.1; reject oversized writes, correct receipt byte counts. | Fixed. |
| GP-07 | P1 | Public library | Claimed 100,000-file tree API delegated to 10,000-entry checksum limit. | Separate receipt construction from public checksum input cap; explicit file/aggregate byte bounds. | Fixed; 10,001-file proof. |
| GP-08 | P2 | CPU / complexity | Recursive JSON rebuilding repeated work for internally constructed scalar tree rows. | Native sorted JSON only for those rows; preserve generic public canonical behavior. | Fixed; measured byte-equivalent encoding. |
| GP-09 | P2 | CLI / errors | Missing values silently defaulted, version alias dropped options, malformed config types reached runtime errors, calendar dates were regex-only. | Strict parser/config checks, calendar validation, operational exception envelope retaining diagnostics. | Fixed; exit-code/JSON tests. |
| GP-10 | P2 | Adapters | Archive paths unvalidated; invalid signing-key types silently produced unsigned receipts. | Reject ambiguous/unsafe paths and nonstring signing keys; preserve explicit empty-string unsigned mode. | Fixed. |
| GP-11 | P2 | Portability / docs | Launcher used a developer-specific path; example lacked required `--path`; schema excluded supported 0.1.0 records; docs overstated adapter execution. | Portable runtime discovery, schema/example corrections, precise library and recovery contracts. | Fixed. |
| GP-12 | P2 | Supply chain / CI | CI compiled unused runtime default features; no contention, bounds or >10,000-file ratchets. | Locked minimal-runtime build and expanded deterministic correctness gates. | Verification recorded below. |
| GP-13 | P2 | Large ledgers | `sort(list_dir(...))` still materializes every filename before bounded record parsing. | Document and retain source evidence; needs additive runtime directory paging or a recoverable index. | Resolved in [follow-up](directory-and-recovery.md); bounded filename heap, O(N) traversal. |
| GP-14 | P2 | Crash recovery | Record/event commits remain two separate filesystem publications; process/power failure can leave a lock or orphan record. | Document manual recovery; ordinary error rollback tested. Automatic journaling/reconciliation requires a design and compatibility decision. | Resolved in [follow-up](directory-and-recovery.md) with durable intents and replay. |

## Changes and compatibility proof

- **Storage and cleanup** (`src/storage.kujo`, storage/concurrency tests): one lock owner,
  no replacement of immutable evidence, bounded metadata inspection, explicit rollback and
  cleanup outcomes. Existing legacy lock directories still block writes at the same paths.
  A real dangling history entry exercises a late write failure, rollback and lock release.
  Record/event formats and IDs are unchanged. The test verifies the sole event checksum
  against the winning record under 24 independent processes.
- **CLI/state contracts** (`src/args.kujo`, `src/core.kujo`, `src/common.kujo`, CLI/audit/bounds
  tests): validated config types, real calendar dates, usage errors, pure previews, no
  unknown-command records, explicit operational JSON failures. Invalid input is rejected,
  never silently interpreted as a default. `--version --json` now honors the flag.
- **Bounded reads** (`src/storage.kujo`, `src/core.kujo`, bounds tests): parse/hash/retained
  record work is capped independently of matches. Listing no longer recomputes unused
  checksums or revalidates the same state directory for every record; direct `show` and
  comparison still return exact checksums. `next_after` allows pages containing only
  corrupt or nonmatching candidates to make progress without hiding warnings.
- **Library filesystem boundary** (`src/hardening.kujo`, hardening/audit/bounds tests):
  validate every manifest before effects, reject duplicates and ancestor collisions,
  serialize cooperating materializers with a no-replace reservation, reject intermediate
  tree symlinks, and report operational I/O failures. Existing input order and hash encoding
  are preserved. Trailing output-root separators remain supported. A mid-write I/O failure
  retains a clearly identified partial tree for inspection rather than claiming success.
- **Encoding and signing** (`src/common.kujo`, `src/hardening.kujo`): an isolated UTF-8 length
  helper uses standard Base64 length/padding because 1.0.1 lacks `byte_length`. It incurs a
  bounded temporary encoding allocation; no cache or newer runtime requirement was added.
  Native JSON replaces recursive encoding only for generated tree rows with string/scalar
  fields; mixed Unicode/control-string fixtures prove byte equivalence. Public checksum
  manifests preserve their existing canonical serialization and entry ordering.
- **Developer operation** (`bin/galleypack`, `scripts/validate.sh`, tests/support): portable
  runtime discovery, explicit invalid runtime errors, test fixture cleanup, selective test
  commands in AGENTS.md and small benchmark receipts. Detailed evidence is linked, not
  eagerly embedded in agent instructions. No prompt/model/schema token count is applicable.
- **CI and documentation** (workflow, schema, README, contracts/security/quickstart,
  changelog): runtime lockfile is enforced; feature reduction is validated against the
  pinned source; tests remain Kujo-native, with fixed POSIX subprocess orchestration only.
  SHA-pinned actions and read-only GitHub token permissions remain intact.

### Contract changes

| Surface | Outcome |
|---|---|
| Public functions | Signatures preserved; invalid/ambiguous paths and invalid signing types now fail; tree byte budgets made explicit. |
| CLI | Commands preserved; missing values/invalid version options exit 2; malformed config and incomplete inspections exit 1; dry runs do not write. |
| JSON envelope | Same required fields and meanings. Operational exceptions use `operation_failed` with diagnostics. |
| Pages | Additive `next_after` and `scanned`; byte/work budgets can yield fewer records, requiring pagination. Doctor/validation disclose truncation. |
| Record/event files | Same paths, IDs and content schemas; readable 0.1.0/0.2.0 records preserved. Lock implementation becomes a regular file at the same path. |
| JSON schemas | Record schema now allows both supported tool versions and requires the already-emitted contract version. |
| Config / environment | Same keys and `KUJO_BIN`; stricter invalid-type rejection. Launcher defaults become portable. |
| External consumers | Small fixture workflows keep their commands, IDs and paths. Large-state consumers must follow cursors and handle incomplete inspections. No sibling change is required for shipped fixes. |

## Performance and efficiency

Measurements use the preserved 1.4.0 binary and deterministic fixtures. This machine had
other active builds/audits, so timings are observations, not throughput promises or CI
latency budgets. Semantic assertions, counts and bounds are the regression gates.

| Dimension / fixture | Before | After | Interpretation |
|---|---:|---:|---|
| 1,500 corrupt records, returned warnings | 1,500 | 1,000 + cursor | Processing and diagnostics bounded; remainder remains retrievable. |
| Same compact result bytes | 141,045 | 94,087 | Measured output reduction, not a token estimate. |
| Same early post-bound timing sample | 30,862 ms | 32,550 ms | No speedup claim; loaded host and validation work changed. |
| Internally generated 1,000-row serialization | 971.152 ms recursive | 8.875 ms native | Same 105,891 bytes in one process; narrow encoding optimization. |
| Unicode record write/read round trip | saved 1,048,704 bytes, unreadable | rejected; 0 stored bytes | Correctness/resource bound restored. |
| 10,001-file tree | failed after hashing | successful exact digest | Public file-count contract repaired; see receipts. |
| Retained record source text per page | up to 1,000 × 1 MiB | at most 4 MiB | Source-enforced bound, **not measured process RSS**. |
| CI runtime dependency package versions on this host | 391 | 255 | `cargo tree -e normal,build`; excludes root and dev-only nodes, deduplicates repeated markers. Target-specific, not a universal crate count. |

No binary-size, total-memory, cold-build-time or model-token improvement is claimed.
The original 2.45-second gate and expanded gate exercise different workloads and must not
be presented as a performance regression comparison. JSON remains pretty-printed for
compatibility; benchmark tools emit concise machine receipts. No runtime caches, external
packages, network calls or opaque heuristics were introduced.

## Security review and limits

An independent static review and architecture review complemented parent source review and
local reproduction. Standard capability preflight was ready; Daybreak access was granted
(Daybreak Blue). The workbench launcher failed on its Python union-type annotation; local
Python 3.11 finalized the canonical scan documents and generated the sealed report.
Baseline source coordinates are preserved in that report and must not be confused with
post-fix line numbers. Finding identity and artifact hashes are in `security/`.

Reviewed CLI/config/JSON boundaries, record IDs, file/symlink access, locks and atomic writes,
rollback, hash/signing inputs, stdout/export limits, adapter declarations, runtime selection,
CI permissions and dependencies. Only the intermediate-tree-symlink issue crosses an
identified lower-trust supplied-tree boundary; its impact is a hash/size leak and false
confinement claim, not arbitrary file contents or remote code execution. It is fixed.

The operator controls state and root ancestors. Concurrent adversarial directory swaps,
external adapter enforcement, owner-authorized state tampering and power-loss transactions
are not magically solved by lexical checks or per-file atomic writes. There are no
application credentials or services to audit. Runtime transitive packages were inspected
as pinned build inputs; this is not a fresh advisory scan or full audit of Kujo itself.

## Cross-repository follow-ups

- **Kujo directory API (optional):** GalleyPack still enumerates/sorts all filenames. An
  additive deterministic bounded directory reader would remove GP-13 without changing
  existing `list_dir`; an index alternative needs recovery semantics. Not required now.
  Related existing SignalBox item: `sig_806c90dc-02bb-4569-8014-d08e1b7d7525` (Dossier).
- **Kujo nested indexed assignment (already tracked):** the inspected 1.4.0 binary can
  silently omit a nested map update in one context and raise stack underflow in another.
  A fixture probe showed a 524,288-character value while serialized `record` stayed 14
  characters after `record["payload"]["note"] = text`. GalleyPack tests construct and
  replace the payload explicitly; production code does not depend on nested assignment.
  Existing SignalBox `sig_ff7d3cd8-d044-42cf-b9d8-fbfd8f103f7d`; no duplicate capture.
  Review runtime mutation semantics and parity; no runtime change is required for this pass.
- The SDK/workbench Python launcher incompatibility affects its tooling, not GalleyPack.
  Local canonical report generation succeeded without editing plugin/system configuration.

No external notification, remote issue or sibling modification was performed. Repository commits are pushed as requested; local SignalBox follow-up is recorded below.

## Remaining work

- **P0/P1:** no unresolved introduced regressions or validated high-severity vulnerabilities.
- **P2:** the original GP-13/GP-14 items are now resolved in the [follow-up](directory-and-recovery.md).
- **P3:** no cosmetic refactor proposed.
- **Needs more evidence:** hostile concurrent filesystem mutation would require a different
  threat model and descriptor-relative runtime APIs; do not promise this protection today.
  Absolute artifact-path migration and automatic stale-lock recovery require compatibility
  decisions; current behavior is documented.
- **Not worth changing:** generic canonical JSON, historical `history` record-list semantics,
  optional adapter plans, unused-but-public profile metadata, pretty JSON and established IDs
  remain intact. No dead public feature was removed on the assumption of no consumers.

## Verification receipt

Commands below ran from the repository root unless a different directory is shown.
`AUDIT=/tmp/galleypack-audit-20260922` is a local evidence directory; preserved summaries
are in [evidence](evidence/). Runtime build/source artifacts are temporary, not dependencies.

| Exact command | Result |
|---|---|
| `/usr/bin/time -p bash scripts/validate.sh` at starting SHA | PASS: 40 baseline assertions and existing checks. |
| `../kujo/target/release/kujo run tests/audit_test.kujo` before fixes | Expected FAIL: 1 pass / 9 failures, evidence retained. |
| `KUJO_BIN=$AUDIT/kujo-current /usr/bin/time -p bash scripts/validate.sh` | PASS: final source, expanded suites, contention and all checks. |
| `KUJO_BIN=$AUDIT/kujo-minimum bash scripts/validate.sh` | PASS on Kujo 1.0.1; later storage/audit/hardening assertions rerun selectively. |
| `$AUDIT/kujo-minimum run tests/storage_test.kujo` | PASS: 8 assertions. |
| `$AUDIT/kujo-minimum run tests/audit_test.kujo` | PASS: 11 assertions. |
| `$AUDIT/kujo-minimum run tests/hardening_test.kujo` | PASS: 10 assertions. |
| `cargo build --release --locked --manifest-path $AUDIT/kujo-pinned/Cargo.toml` | Stopped during unused optional-feature compilation; not a passed build. |
| `cargo build --release --locked --no-default-features --manifest-path $AUDIT/kujo-pinned/Cargo.toml` | PASS: release build, lockfile enforced, no optional default features. |
| `KUJO_BIN=$AUDIT/kujo-pinned/target/release/kujo bash scripts/validate.sh` | PASS: final source, all nine suites and repository checks on minimal-feature Kujo 1.0.2. |
| `$AUDIT/kujo-current run scripts/benchmark.kujo` in baseline source and changed source | PASS: corruption-page counts/bytes measured. |
| `$AUDIT/kujo-current run scripts/benchmark.kujo -- bytes` in both source trees | PASS: unreadable Unicode record reproduced, then prevented. |
| `$AUDIT/kujo-current run scripts/benchmark.kujo -- tree` in both source trees | Baseline receipt FAIL / changed receipt PASS for 10,001 files. |
| `$AUDIT/kujo-current run scripts/benchmark.kujo -- serialization` | PASS: byte equality and timings. |
| `$AUDIT/kujo-current run scripts/benchmark.kujo -- tree $AUDIT/tree-fixture` | PASS: 10,001 files; independently recomputed digest matches. |
| `$AUDIT/kujo-pinned/target/release/kujo run scripts/benchmark.kujo -- tree $AUDIT/tree-fixture` | PASS: 10,001 files; exact same independently checked digest. |
| `cargo tree --locked --edges normal,build --prefix none --format '{p}' --manifest-path $AUDIT/kujo-pinned/Cargo.toml` (and with `--no-default-features`) | PASS: package-version counts 391 / 255 after normalization. |
| `python3.11 <security-plugin>/scripts/finalize_scan_contract.py --scan-dir docs/audits/security --source-root $AUDIT/baseline-src` | PASS: canonical documents sealed and report generated. |
| Audit evidence JSON parsing and sealed security artifact SHA-256 verification | PASS. |
| `git diff --check` | PASS. |

The full validation script runs entrypoint checking, all nine Kujo suites, JSON parsing for
fixtures/schemas, help/version/doctor launcher smoke tests, foreign-runtime dependency
checks, badge/ignore policy checks and whitespace checking. No formatter, separate linter,
web E2E service or native GalleyPack binary build exists. CI additionally checks the
10,001-file benchmark's semantic assertion; no timing threshold can make the gate flaky.

## Durable record

SignalBox admitted one unresolved crash-recovery design finding:
- Capture: `cap_d9804fed-21c0-4f11-9943-bc15b8de47ee`.
- Signal: `sig_040ce4b1-ac74-4ba7-9935-4a0fc124ed8e`.
- Exact-ID retrieval passed for both; concept searches “GalleyPack crash” and
  “record and creation-event” retrieved the signal and capture respectively.
- Existing directory-enumeration and nested-assignment findings listed above were
  deduplicated; resolved fixes, routine verification and implementation recaps were rejected.

Strata receives the final commit, project-state milestone and continuation pointers
as one scoped session handoff after publication. The final response records its ID.
