# Directory paging and crash recovery follow-up — 2026-09-22

Repository: `kujolang/galleypack`, branch `main`.
Starting SHA: `480183c5f87e0d5d8c6322e76baeb807a6eb7ee6` (clean).
Ending implementation SHA: **cd096d7897034e2870d34a8c568f76b63871fed9**. A following documentation-only
commit records this receipt without embedding its own self-referential SHA.

The user requested completion of both remaining findings in the original audit.
This follow-up closes GP-13 (eager filename allocation/sorting) and GP-14
(record/event crash consistency and stale record locks). No sibling repository was
modified. Previously unavailable native primitives now exist in published Kujo
revision `cf785c0a7953717af16b657cda05b85d628144c5`, already on its origin/main.

## Baseline and findings

The expanded starting-revision validation passed on a preserved local Kujo 1.4.0
binary, including the original 24-writer contention test. See
[baseline receipt](followup-evidence/baseline.txt). The source still used
`sort(list_dir(records))`, separate record/event writes without durable intent,
and existence-based lock cleanup. The starting audit and security scan remain
historical evidence; their sealed scan documents were not rewritten.

| ID | Priority | Evidence | Implementation | Status |
|---|---|---|---|---|
| GP-13 | P2 | Entire filename array retained and sorted before candidate bounds. | Native heap-bounded directory pages; bounded dangling-link detection; filename-consistent cursors; exposed enumeration counters. | Resolved. |
| GP-14 | P2 | Crash could leave record without event or an abandoned lock. | Durable intent, per-ID OS lock, compatibility sentinel, checked synced publication, immutable replay and explicit legacy repair. | Resolved. |
| F-01 | P1 | A record awaiting its event could otherwise appear complete. | Reads reject pending intents; doctor/list expose pending work; doctor/validate check exact event correspondence. | Fixed; crash and tampering tests. |
| F-02 | P2 | Stem comparison disagreed with full filename ordering for prefix IDs. | Restore `.json` for exclusive cursor comparisons; treat continuation strings only as comparison data. | Fixed; complete prefix-ID traversal. |
| F-03 | P2 | Recovery must not serialize independent IDs or replace conflicting evidence. | Per-ID locks, bounded per-ID replay and explicit conflicts; 24 distinct-ID writers all succeed. | Verified. |

## Implementation and contracts

`src/transactions.kujo` owns the internal protocol: bounded journal decoding,
record/event derivation, synced no-replace publication and replay. `src/storage.kujo`
validates managed directories, serializes each ID, reserves the original exclusion
path, calls protocol steps, and releases native lock handles on success/failure.
Stable native lock inodes persist in `writer-locks/`; compatibility sentinels in
`locks/` are removed once publication completes. Unknown old lock files and empty
old lock directories require an explicit maintenance override; no PID or timeout
heuristic declares another writer dead.

The durable intent precedes both record and event. It records exact serialized
bytes plus SHA-256; recovery derives paths from validated IDs/timestamps and never
accepts arbitrary filesystem destinations from the journal. Replaying an existing
matching file re-syncs its directory; differing bytes, unsafe paths or corrupted
intent fail without overwriting evidence. Initialization also syncs directory names,
including a root whose first initialization was interrupted before metadata appeared.
A failed post-intent write retains redo information instead of rolling back evidence.
An identical mutation retry can replay pending intent; a different retry cannot take
over the ID. After completion, duplicate-ID behavior remains unchanged.

The new `recover` command offers bounded pages, ID selection, read-only preview,
legacy orphan-event repair and explicit conflict results. Page failures preserve
per-ID diagnostics and permit continuation. Corrupt filenames can be passed back
as opaque cursors but are never used as recovery IDs or paths. Invalid record/event
pairs can no longer make doctor or validation claim success. The old compatibility
test had changed `tool_version` without updating the event: this now correctly fails
as tampering. A separate consistent 0.1.0 record proves compatibility; no assertion
was disabled or weakened to accept inconsistent state.

See [recovery protocol and operator guide](../recovery.md) for exact commands,
state transitions, crash windows and migration. README, CLI help, contracts,
security guidance, AGENTS.md, changelog and CI were updated accordingly.

### Compatibility

- Record/event JSON, paths, IDs, schema and tool-version compatibility remain intact.
- A transaction-intent JSON Schema is added; existing record/event schemas are unchanged.
- Storage gains additive `transactions/` and `writer-locks/` directories. Old states
  are upgraded during initialization; no bulk record conversion or index migration.
- `recover` is additive. Ordinary reads and dry runs do not automatically mutate state.
- `--force` additionally permits legacy-lock removal during recovery after old writers
  stop; it never authorizes record/event replacement or following symlinks.
- Operational publication failure now reports `recovery_required` and retains intent.
  Unknown legacy locks report an explicit recovery conflict rather than being guessed stale.
- Record pages add measured `directory_entries_buffered`/`directory_entries_examined`.
  Doctor/validation deliberately reject inconsistent creation history.
- **Runtime prerequisite changes:** minimum is the tested Kujo 1.4.0 revision above
  or a compatible later build. The earlier 1.0.x floor cannot supply the required APIs.
  There is no silent eager-enumeration or unsafe-lock fallback. An old-runtime probe
  verifies mutation fails before creating state. Upgrade runtime before deployment;
  the `1.4.0` version label alone is insufficient because API additions share that label.
- Existing configurations and environment variables are unchanged. Existing public
  library signatures remain intact; new transaction exports are documented internal
  steps, not supported standalone mutation entrypoints.

## Bounds and measured impact

| Dimension | Before | After / proof |
|---|---|---|
| Filename retention, 5,000-entry fixture | 5,000 names allocated/sorted (source-derived from complete enumeration) | At most 1,001 names, reported by native heap instrumentation; all 2,500 JSON records traversed across three pages. |
| Isolated 10,001-name directory phase | 10,001 names retained; 217.356 ms | 1,001 peak retained names; 62.297 ms; exact first-page equality in the same process (single warm-order sample). |
| Directory traversal CPU | O(N) enumeration plus full sort | Still O(N) enumeration per page, bounded heap and sort; not a constant-time lookup or filesystem snapshot. |
| Record parsing / retained record source | 1,000 candidates / 4 MiB | Preserved. Pending-work warnings are bounded separately to two summary entries. |
| Recovery filename buffers | No recovery | At most `4 × (limit + 2)` retained native names, plus O(limit) merged candidates; limit 1..1,000. |
| Journal / record input | No intent | Journal <=2,162,688 bytes, decoded record <=1 MiB; one ID replayed at a time. |
| Same-ID contention | One winner in original 24-worker sample | One winner, exact matching event, no pending intent/sentinel; OS ownership also survives SIGKILL correctly. |
| Different-ID contention | Per-ID locks | All 24 concurrent writers complete; no new global writer bottleneck. |
| Crash boundaries | No durable replay protocol | Six actual SIGKILL boundaries (exit 137, no timeout/cancellation) recover deterministically, including staged-but-uncommitted and fully published cases. |
| Compact 1,500-corrupt-record receipt | 94,087 bytes | 94,155 bytes: +68 bytes for bounded-enumeration diagnostics. |

The preserved listing timing samples are 9,353.777 ms before and 20,534.613 ms after
on a host under substantial concurrent compilation and memory pressure. A subsequent
sequential repeat measured 5,961.408 ms before and 5,490.486 ms after, reversing the
early difference. Both receipts are preserved; these observations do **not** establish
a stable throughput change. Added recovery inspection also has work costs. No
latency improvement or timing gate is claimed. The regression gate is filename
retention plus complete behavior, not a throughput promise. Durable publication
adds journal/file/directory sync operations by design. Persistent native lock files
cost one small inode per attempted ID; unlinking them during operation would
reintroduce lock-identity races. No process-RSS or model-token claim is made.

## Security and failure review

Reviewed trusted root/ancestor assumptions, metadata/state validation, native lock
ownership and no-follow behavior, publication receipts, byte caps, intent decoding,
legacy override semantics, immutable conflicts, error propagation and cleanup.
Native capability names are resolved before a storage mutator makes changes.
Same/different-ID contention, active lock exclusion, corrupted journal checksums,
malformed JSON, conflicting records, event symlinks and I/O failure are exercised.
No shell command consumes a production user path; fixed POSIX shell orchestration
is limited to tests, including a child shell killing its own parent worker.

Durability relies on filesystem file/directory sync and atomic hard links. Tests
kill processes, not physical hardware. Operator-controlled state is not an
adversarial multi-tenant sandbox. A crash inside a native atomic-write/barrier
primitive may leave uncommitted temporary files; they are not treated as records
or intent. Optional review-package materialization retains its separately
documented partial-tree/manual-inspection contract; this finding concerned the
record/history store. Conflicting externally modified evidence requires operator
inspection, not a destructive automatic repair.

## Verification receipt

`FOLLOWUP=/tmp/galleypack-followup-20260922`; detailed local build logs stay there.
All repository tests remain Kujo-native. No unrelated lint or test was disabled.

| Command | Result |
|---|---|
| `KUJO_BIN=$FOLLOWUP/kujo bash scripts/validate.sh` at starting SHA | PASS, unchanged baseline. |
| `$FOLLOWUP/kujo run $FOLLOWUP/probe.kujo` | PASS, bounded page and verified synced publication receipt. |
| `$FOLLOWUP/kujo run tests/recovery_test.kujo -- $FOLLOWUP/kujo` | PASS, actual SIGKILL boundaries, legacy repair, conflicts, symlink/checksum rejection and replay. |
| `$FOLLOWUP/kujo run tests/concurrency_test.kujo -- $FOLLOWUP/kujo` | PASS, 24 same-ID and 24 distinct-ID processes. |
| `$FOLLOWUP/kujo run tests/directory_page_test.kujo` | PASS, 5,000 filenames and bounded complete traversal. |
| Old 1.0.1 runtime: `run galleypack.kujo -- init --state $FOLLOWUP/unsupported-runtime-state --json` | Expected failure before any state creation; receipt retained. |
| `$FOLLOWUP/kujo run scripts/benchmark_directory.kujo -- /tmp/galleypack-audit-20260922/tree-fixture` | PASS, exact page equality and native filename counts. |
| `$FOLLOWUP/kujo run scripts/benchmark.kujo` in starting and changed source | PASS, compact receipts retained; no speed claim. |
| `cargo build --release --locked --no-default-features --manifest-path $FOLLOWUP/runtime/Cargo.toml --target-dir $FOLLOWUP/target` | PASS: locked release build, default features disabled. |
| `KUJO_BIN=$FOLLOWUP/kujo bash scripts/validate.sh` on changed source | PASS: all eleven suites; later initialization/option checks also rechecked in focused tests. |
| `KUJO_BIN=$FOLLOWUP/target/release/kujo bash scripts/validate.sh` | PASS: committed source, all eleven suites, JSON schema checks, CLI smoke and repository gates. |
| `$FOLLOWUP/target/release/kujo run scripts/benchmark.kujo -- tree /tmp/galleypack-audit-20260922/tree-fixture` | PASS: 10,001 files; digest matches original independently checked receipt. |
| `/Users/robertdevore/2026/Kujolang/kujo-repos/kujo/target/release/kujo run tests/test.kujo` | PASS: AGENTS.md required check, 14 assertions. |
| Audit receipt JSON parsing / tree digest comparison | PASS. |
| `git diff --check` | PASS. |

## Remaining work and durable tracking

GP-13 and GP-14 are closed in GalleyPack. No new cross-repository implementation is
required; deployment must use the updated runtime prerequisite. Filesystem sync
semantics and live-directory non-snapshot ordering are explicit platform contracts,
not promised protections that the implementation omits.

The previous SignalBox crash-recovery capture `cap_d9804fed-21c0-4f11-9943-bc15b8de47ee`
and signal `sig_040ce4b1-ac74-4ba7-9935-4a0fc124ed8e` are historical provenance for
the now-implemented work. No completed-work Capture or duplicate Signal is created.
Strata receives one deduplicated follow-up handoff/state milestone after push, linked
to the earlier audit note `238de5be-cae9-4625-bdde-a54053a98530` and final commits.
