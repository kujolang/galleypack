# Kujo 1.5 adoption and audit closure

Repository: `kujolang/galleypack`, branch `main`.
Starting SHA: `0f1ceb86d44dd4eb5bcd5095f7f3dfbc378b964e`.
Ending implementation SHA: `25d06e61bfb6cf99c98771fe0405629b0b09b822`.
The subsequent audit-receipt commit changes only documentation/evidence.

GalleyPack remains an offline artifact/review ledger and package-contract library.
Its only runtime dependency is Kujo. This follow-up adopts the released runtime
and reviews the remaining limits from the [original audit](repository-hardening.md)
and [directory/recovery work](directory-and-recovery.md). Historical evidence and
runtime pins in those reports remain unchanged.

## Runtime provenance and baseline

CI now pins Kujo v1.5.0's peeled commit
`cc2d7dbb59a8dc05f00d629e100932f56f4062f6`, verified against the local annotated
release tag and GitHub `refs/tags/v1.5.0^{}`. CI retains the locked, minimal-feature
source build, pinned checkout/toolchain actions and read-only repository token.

Local verification uses the official macOS x64 release archive. Its SHA-256 is
`1aebcd482125031104b2df79abae6db57973f1874ceb196f95989b14e287d820`, matching the
release tag's archive digest; the extracted runtime reports `kujo 1.5.0`.
Extracted binary SHA-256: `3e1e475ea165c8b4a714495596fe8661ad970b27119ad44f1db4d9779f7f05d4`.
The older sibling checkout/binary was inspected but not modified. No claim is made
that its locally built 1.4.0 binary verifies the 1.5.0 source pin.

Before behavior changes, the entire existing eleven-suite validation gate passed
on the released 1.5.0 binary. No preexisting failures were observed. The 10,001-file
tree benchmark passed with the previously verified digest. See
[baseline log](kujo-1.5-evidence/baseline.txt).

## Findings and implementation

| ID | Priority | Area | Finding / evidence | Action | Status |
| --- | --- | --- | --- | --- | --- |
| V15-01 | P2 | Runtime / contracts | CI and diagnostics named prerelease 1.4 revisions despite the available stable 1.5 release. | Pin release commit; align version/doctor prerequisites, capability guard, operator docs and CLI regression. | Fixed. |
| V15-02 | P1 | Tree hashing | Component checks and metadata size preceded a separate ambient `sha256_file` open, leaving a check/read gap and no during-read growth bound. | Use `sha256_file_beneath` with no-follow components, a held file handle and remaining-byte ceiling; serialize actual hashed bytes. | Fixed for declared paths below the trusted root. |
| V15-03 | P2 | Resource efficiency | `utf8_size` allocated an entire Base64 intermediate solely to derive the byte count. | Retain helper interface, use native `byte_length`; existing byte-limit regression tests remain. | Fixed. |
| GP-13/14 | P2 | Directory enumeration / recovery | Previous follow-up implemented bounded pages, durable intents and process locks. | Rerun cursor, 24+24 writer and six SIGKILL boundary gates on 1.5. | Verification below. |

Files affected: runtime pin in `.github/workflows/validate.yml`; capability/byte
helpers in `src/common.kujo`; diagnostic prerequisites in `src/core.kujo`; confined
hashing in `src/hardening.kujo`; regression gates in `tests/tree_hash_test.kujo`,
`tests/cli_test.kujo` and `scripts/validate.sh`; corresponding operator/contract docs.

The tree preflight and established static-input errors remain intact. The runtime
rechecks regular-file type and maximum size on its held handle, rejects symlink
components, and bounds observed growth. The tree's total tracks actual bytes.
When exactly 1 GiB has been consumed, a subsequent empty file is still legal;
the native API's minimum one-byte ceiling is followed by the total-byte check.
The runtime uses a fixed 64 KiB hashing buffer. No cache or dependency was added.

New tests cover stable sorted receipts for binary, empty and Unicode files;
missing/unsafe paths, directory leaves, live/dangling/intermediate symlinks, FIFO
rejection without blocking, oversized sparse files, the exact 64 MiB / 1 GiB boundaries (including a final
empty file), aggregate overflow by one byte, and reported runtime pin.
Existing source-bound, recovery, contention and backward-record tests remain gates.

## Performance and efficiency

The reproducible [byte benchmark](kujo-1.5-evidence/utf8-benchmark.kujo) compares
old and native counting on the same 1.5 runtime, five samples of 20 calls on a
1 MiB UTF-8 string. It asserts exact byte equivalence. Raw samples are in
[utf8.json](kujo-1.5-evidence/utf8.json). The old path creates 1,398,104 Base64 bytes
per call; the replacement creates no Base64 intermediate. This is an intermediate
allocation comparison, not a measured process-RSS reduction or an end-to-end claim.

The same existing 10,001-file fixture and released runtime are used before/after
for tree hashing. Receipts and measurements are preserved alongside this report.
Shared-host, single-run timings are observational; no throughput ratchet or broad
speed claim is warranted. Receipt schema and digest must remain identical.
| Measurement | Before | After |
| --- | --- | --- |
| Byte count, median of five 20-call samples | 207.933 ms | 30.081 ms |
| Base64 intermediate per 1 MiB count | 1,398,104 bytes | 0 bytes |
| 10,001-file tree, one shared-host run | 58334.874 ms | 61011.464 ms |
| Tree digest / serialized receipt | `f04bd93d1ca8ef9f98d9042fe778818aaa44a691fed92f32a65601d33219154a` | Identical |

The tree run is slightly slower in this sample; the change strengthens confinement
and byte accounting, not a claimed speed optimization. No meaningful token/context,
binary-size or dependency-surface change is claimed.

## Security, compatibility and remaining work

Public function signatures, record/history/journal formats, JSON schemas, config,
environment variables, CLI commands and exit codes are unchanged. Runtime minimum
is intentionally 1.5.0; diagnostic values now report the exact release source pin.
External callers must upgrade their runtime; stored records need no migration.
Tree hashes on stable supported inputs remain identical. Concurrent file growth
or replacement may now fail safely rather than producing an unbounded or mismatched
receipt. A digest is not a snapshot guarantee against in-place concurrent writes.

No P0/P1/P2 repository-local audit item remains open. GP-13/GP-14 remain closed.
No sibling code changes or required cross-repository follow-ups were identified.

Retained, explicit contracts (not unresolved implementation promises):

- Directory paging scans O(N) entries; pages are not snapshots. The new alternative
  `list_dir_beneath` has a different cursor model and hard scan ceiling; switching
  would reject previously supported large directories, so the bounded-heap API stays.
- 1.5 exposes no direct directory-sync API to replace the tested publication barrier.
  Parent write access at first initialization and filesystem sync/hard-link support
  remain required. SIGKILL tests do not emulate physical power loss.
- Operator-selected roots/ancestors remain trusted. Whole-repository sandboxing or
  snapshots would be a new contract; not justified by evidence in this task.
- Review-package materialization retains its documented partial-tree/manual-lock
  inspection contract. Executed archive/object-store adapters remain optional plans,
  not unfinished features promised by the CLI.

## Verification receipt

Evidence directory: `docs/audits/kujo-1.5-evidence/`.
Local runtime: `/tmp/galleypack-kujo15/kujo`.
Release download: `gh release download v1.5.0 --repo kujolang/kujo --pattern kujo-v1.5.0-macos-x64.tar.gz --dir /tmp/galleypack-kujo15`.
Archive verification: `shasum -a 256 /tmp/galleypack-kujo15/kujo-v1.5.0-macos-x64.tar.gz`.

| Exact command / check | Result |
| --- | --- |
| `git -C ../kujo rev-parse 'v1.5.0^{commit}'` and `git -C ../kujo ls-remote origin refs/tags/v1.5.0 'refs/tags/v1.5.0^{}'` | PASS: same peeled release SHA. |
| `/tmp/galleypack-kujo15/kujo --version` | PASS: 1.5.0. |
| `KUJO_BIN=/tmp/galleypack-kujo15/kujo bash scripts/validate.sh` before changes | PASS: all eleven original suites and repository gates. |
| `/Users/robertdevore/2026/Kujolang/kujo-repos/kujo/target/release/kujo run tests/test.kujo` before changes | PASS: prior AGENTS command, 14 assertions on old local runtime. |
| `/tmp/galleypack-kujo15/kujo run tests/tree_hash_test.kujo` | PASS, including exact aggregate boundaries. |
| `/tmp/galleypack-kujo15/kujo run tests/cli_test.kujo -- /tmp/galleypack-kujo15/kujo` | PASS: 13 contract assertions. |
| `/tmp/galleypack-kujo15/kujo run /tmp/galleypack-kujo15/utf8-benchmark.kujo` | PASS: five before/after samples, identical sizes. Script copied into evidence. |
| `/tmp/galleypack-kujo15/kujo run scripts/benchmark.kujo -- tree /tmp/galleypack-audit-20260922/tree-fixture` before and after | PASS: 10,001 files, identical entire receipt except separately reported elapsed time. |
| `../kujo/target/release/kujo run galleypack.kujo -- init --state /tmp/galleypack-kujo15/unsupported-state --json` then `test ! -e /tmp/galleypack-kujo15/unsupported-state` | PASS: unsupported old binary exits nonzero and creates no state. Failure envelope retained. |
| `git diff --check` | PASS. |

Final local gate: **PASS**, `KUJO_BIN=/tmp/galleypack-kujo15/kujo bash scripts/validate.sh`
after the implementation commit, including all twelve suites, extended tree-byte
boundaries, 5,000-entry directory traversal, 24 same-ID plus 24 independent writers,
six SIGKILL recovery boundaries, fixture/schema JSON, entrypoint checking, launcher
smoke and repository gates. [Full concise log](kujo-1.5-evidence/validation.txt).
All benchmark and CI JSON evidence files also pass `scripts/validate_json.kujo`.

CI: **PASS** on implementation SHA `25d06e61bfb6cf99c98771fe0405629b0b09b822`:
[GitHub Actions run 35796406088](https://github.com/kujolang/galleypack/actions/runs/35796406088).
The Linux job built the exact pinned runtime using
`cargo build --release --locked --no-default-features --manifest-path .runtime/kujo/Cargo.toml`,
then passed `bash scripts/validate.sh` and `"$KUJO_BIN" run scripts/benchmark.kujo -- tree`.
[Step receipt](kujo-1.5-evidence/ci.json) preserves completion status and timing.
The following documentation commit preserves these results without changing executable code.

Strata consolidation records this release milestone and supersedes the prior runtime
prerequisite while preserving historical audit evidence. SignalBox: no captures warranted.
