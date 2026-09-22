# Security and authority

GalleyPack is an offline, operator-run tool. Actor and provenance fields are
assertions, not authenticated identities. Review completion is not approval;
GalleyPack has no publishing or ACT capability.

State and caller-selected root ancestors must be operator controlled. IDs reject
traversal, managed state directories and record leaves reject symlinks, and
optional tree hashing opens every relative component without following symlinks,
then hashes through the held file handle with an enforced read budget. Its byte
receipt covers the bytes hashed; concurrent content mutation is not a snapshot.
Other path checks are not a sandbox against hostile directory replacement.
Source/config paths and stored relative artifact paths resolve against invocation
CWD; use absolute artifact paths when validating from different directories.

Inputs and stored records are limited to 1 MiB **UTF-8 bytes**, core artifacts to
64 MiB, and rendered exports to 8 MiB. Listing parses at most 1,000 candidate
records and retains at most 4 MiB of source record text per page. Continue with
`next_after` while `truncated` is true. Native directory paging retains at most
1,001 candidate filenames instead of allocating/sorting the entire directory.
It still scans the directory (O(N) work per page); pages are not filesystem snapshots.
`doctor` and whole-state `validate` never certify a truncated inspection and check
creation events against exact record bytes.

A durable transaction intent precedes immutable record and history publication.
Publication receipts must confirm file identity and directory synchronization.
Interrupted transactions retain their intent, and reads refuse pending records.
`recover` replays exact bytes without replacing conflicting evidence; ordinary I/O
failure retains redo information instead of attempting destructive rollback.
Process-owned per-ID locks are released by the OS after process death. Persistent
`writer-locks` files are stable lock inodes, not evidence of a live/stale writer;
never unlink them while any process might use the state. Compatibility markers in
`locks` keep legacy cooperating writers out until the transaction finishes.
Recognized protocol markers can be cleared safely under the process lock; unknown
legacy locks require the operator to stop old writers and opt in with `recover --force`.
See [recovery](recovery.md) for migration, replay and conflict handling.

Durability depends on the OS/filesystem honoring file and directory sync and
atomic hard-link publication. Unsupported sync or publication fails explicitly.
SIGKILL recovery is tested; hardware power interruption is not emulated. This is
not protection against arbitrary disk corruption or operator state tampering.

`--force` authorizes explicit export replacement or removal of legacy record locks
during recovery; it never authorizes record/event replacement or symlink traversal.
JSON payload secret-shaped keys are
rejected; this is not a content DLP scanner. Do not put secrets in artifact text,
metadata, actor fields or optional adapter receipts. HMAC keys are caller-owned
library arguments and are not persisted by the signing helper.

Review-package manifests are fully checked for duplicates, collisions, path
safety and size before writing. A valid manifest that encounters a later I/O
failure returns `ok: false`, the output root and completed entry receipts; inspect
and remove that partial tree before retrying. Cooperating materializers reserve
`<output_root>.galleypack-lock`; a crash can leave that reservation for manual
inspection. Archive and object-store helpers validate descriptions, not executed
adapters or external services.
