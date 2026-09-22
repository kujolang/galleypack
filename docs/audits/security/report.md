# Security Review: kujolang/galleypack

## Scope

Whole tracked repository baseline; dynamic tests isolated in temporary directories; independent static review and architecture review.

- Scan mode: repository
- Target kind: git_revision
- Target ID: galleypack-3befaac6903e3ebd04add1f3509f787e9aced89e472f4e0385a9a4ccb704e8db
- Revision: 5a9ec247f943460aa759f1552ae14e48c7423d3e
- Inventory strategy: repository
- Included paths: .
- Excluded paths: none
- Runtime or test status: Kujo 1.4.0 local baseline; CI-pinned runtime verification separately recorded.

### Scan Summary

| Field | Value |
| --- | --- |
| Scan outcome | completed |
| Reportable findings | 1 |
| Severity mix | low: 1 |
| Confidence mix | high: 1 |
| Coverage | complete |
| Validation mode | Source review and local regression reproduction |

Canonical artifacts: `scan-manifest.json`, `findings.json`, and `coverage.json`. This report is a deterministic projection of those files.

## Threat Model

Operator-run offline Kujo CLI binds editorial artifact bytes and assertions to immutable records and SHA-256 hashes. bin/galleypack:3 selects the runtime; galleypack.kujo calls src/core.kujo:176 main. Optional src/hardening.kujo exports are not CLI commands. No network, publishing, archive execution or credential retrieval is implemented.

### Assets

- Record and audit integrity: \<state\>/records/\<id\>.json; \<state\>/history/\<sha256(timestamp+newline+id)\[0:32\]\>.json; \<state\>/locks/\<id\>.lock; \<state\>/metadata.json (src/storage.kujo:35-76).
- Operator-selected source files and output filesystem; sources must remain unchanged (src/core.kujo:55,107).
- Caller-owned HMAC signing key is consumed but not persisted by checksum_manifest (src/hardening.kujo:8).
- Bounded processing of config, payloads, records, artifacts and optional manifest/tree library inputs (src/core.kujo:26-60; src/hardening.kujo:6-10).

### Trust Boundaries

- CLI/config/input to core: allowed flags, object parsing, schema-major and recursive sensitive-key checks, actor/timestamp and domain validation (src/core.kujo:26-81). Actor is an assertion, not authenticated identity.
- Explicit --path to regular-file and size checks to SHA-256. Stored relative paths resolve against invocation CWD (src/core.kujo:55,95).
- State precedence: nonempty --state, config.state, .galleypack. Safe IDs and managed-directory checks protect ordinary cooperating operations; local state is operator controlled (src/core.kujo:35; src/storage.kujo:27-89; SECURITY.md:3).
- Export uses literal --output and --force to authorize replacement, with leaf-symlink and size checks (src/core.kujo:107).
- Library caller chooses output_root/root and supplies manifest or relative paths; entries cross into filesystem operations under that chosen root (src/hardening.kujo:3-10).
- Adapter descriptions only assert offline behavior, normalized timestamps and ordering; no external execution is enforced or performed (src/hardening.kujo:12-17).
- Operator chooses KUJO_BIN; CI builds commit 5059695d14d6726bc17fef55e0b95511624967cf with pinned actions and contents:read permissions (.github/workflows/validate.yml:5-25).

### Attacker Capabilities

- A contributor may supply a tree, manifest, JSON input or artifact that an operator chooses to process; they do not thereby control CLI configuration, signing keys or the operator account.
- A tree supplier may include symlink directories pointing outside the declared root.
- Remote/multitenant exposure is not established. An actor who already controls operator state has no new authorization boundary to cross (SECURITY.md:3).

### Security Objectives

- Preserve immutable IDs, exact artifact hashes, append-only audit behavior and safe explicit output writes.
- Keep processing bounded and offline; reject malformed input and secret-shaped record fields.
- Separate review assertions from approval or publishing authority.
- Reject tree paths that redirect declared-root hashing outside that root.

### Assumptions

- All cited source coordinates refer to starting revision 5a9ec247f943460aa759f1552ae14e48c7423d3e.
- Local state and caller-selected root ancestors are trusted; simultaneous adversarial filesystem replacement is not a supported sandbox boundary. POSIX launchers; Windows support not established.
- README minimum Kujo 1.0.1 is descriptive; version/doctor do not interrogate the runtime. CI actually pins 1.0.2 source.
- Baseline docs overstated archive execution and a 100,000-file hash bound; helpers return plans and the tree receipt delegated to a 10,000-entry limit. Fixed/documented in repository-hardening.md.
- Baseline history command lists records rather than audit events; retain that CLI behavior.
- Standard capability preflight ready; native V2 cap 4 permits 3 workers. Daybreak granted/Daybreak Blue. Workbench launcher failed with Python TypeError; prompt-only local contract uses Python 3.11.

## Findings

| Finding | Severity | Confidence | Detailed write-up |
| --- | --- | --- | --- |
| [Intermediate directory symlinks escape the declared hash root](#finding-1) | low | high | inline below |

### Confidence Scale

| Label | Meaning |
| --- | --- |
| high | Direct evidence supports the finding with no material unresolved blocker. |
| medium | Evidence supports a plausible issue, but material runtime or reachability proof remains. |
| low | Evidence is incomplete and the item is retained only for explicit follow-up. |

<a id="finding-1"></a>

### [1] Intermediate directory symlinks escape the declared hash root

| Field | Value |
| --- | --- |
| Severity | low |
| Confidence | high |
| Confidence rationale | Reproduced by tests/audit_test.kujo against baseline, then verified rejected after component validation. |
| Category | symlink-following |
| CWE | CWE-61 |
| Affected lines | src/hardening.kujo:10 |

#### Summary

The exported tree hasher checks only the root and final file for symlinks. A supplied root/link/file can hash an outside file through a directory symlink.

#### Root Cause

Lexical safe_relative rejects traversal components, but leaf-only symlink checks do not confine filesystem resolution.

#### Validation

A real root/link -\> outside fixture makes baseline streaming_tree_hash(root,\[link/file\],10) succeed; the same regression rejects it after the fix.

#### Dataflow

Tree supplier places link directory under root; caller supplies link/file; joined path resolves to outside/file; file_size and sha256_file consume it.

#### Reachability

Directly reachable through exported library API; absent from CLI. Attacker must supply the tree or alter its entries.

#### Severity

**Low** — Requires a lower-trust supplied tree; exposes only an outside file hash/size, not contents. No network sink or remote service. Local state is not multitenant.

Additional runtime or deployment evidence could raise or lower this severity.

#### Remediation

Reject every relative path component that is a symlink before hashing. Require operator-owned stable roots; do not promise protection from hostile concurrent replacement.

Tests:
- tests/audit_test.kujo: real intermediate symlink is rejected

## Reviewed Surfaces

| Surface | Risk Area | Outcome | Notes |
| --- | --- | --- | --- |
| Optional library manifest/tree paths | not recorded | Reported | All src/hardening.kujo reviewed. Intermediate symlink reproduced; duplicates, collisions, false signing and size limits fixed as documented correctness/defense improvements. |
| Local state, concurrency, immutability and export | not recorded | No issue found | src/storage.kujo and src/core.kujo fully reviewed. Operator-controlled state per SECURITY.md. Nonexclusive create_dir locks and overwrite=true fixed as data-integrity defects. No separate multitenant vulnerability claimed. |
| CLI, configuration, domain validation and schemas | not recorded | No issue found | All src/args.kujo, src/common.kujo, src/domain.kujo, src/profile.kujo, schemas and fixtures reviewed. JSON parsing and safe record IDs prevent traversal. Dry-run, missing flag values, UTF-8 bounds, calendar dates and truncated validation fixed. |
| Launchers, CI, tests, docs and dependency contracts | not recorded | No issue found | All tracked baseline files inspected. Only Kujo runtime dependency, pinned CI actions/source; no application network/subprocess or hosted service. Test subprocesses fixed argv and bounded. Docs corrected to describe library plans and local authority. |

## Open Questions And Follow Up

- Hostile concurrent root replacement is outside operator-controlled filesystem model; descriptor-relative APIs are required if a future integration needs that boundary.
