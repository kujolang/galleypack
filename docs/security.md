# Security and authority

GalleyPack is an offline, operator-run tool. Actor and provenance fields are
assertions, not authenticated identities. Review completion is not approval;
GalleyPack has no publishing or ACT capability.

State and caller-selected root ancestors must be operator controlled. IDs reject
traversal, managed state directories and record leaves reject symlinks, and
optional tree hashing rejects symlinks in every relative path component. These
checks are not a sandbox against an adversary replacing directories concurrently.
Source/config paths and stored relative artifact paths resolve against invocation
CWD; use absolute artifact paths when validating from different directories.

Inputs and stored records are limited to 1 MiB **UTF-8 bytes**, core artifacts to
64 MiB, and rendered exports to 8 MiB. Listing parses at most 1,000 candidate
records and retains at most 4 MiB of source record text per page. Continue with
`next_after` while `truncated` is true. Directory-name enumeration is still eager
in the minimum supported runtime. `doctor` and whole-state `validate` fail with
`inspection_incomplete` when they cannot inspect the entire state in one page.

New records, history events, metadata, and lock files use atomic no-replace
publication. Cooperating duplicate writers cannot overwrite evidence. Existing
legacy lock directories still block writes. A crash can leave a lock or a record
without its history event because the two files are separate commits. Stop all
writers, inspect the record/event checksums, and reconcile manually before
removing a stale lock; the tool never guesses that a lock is stale. Ordinary
history-write failure attempts record rollback and reports incomplete rollback.

`--force` authorizes replacement of an explicitly named export only; it does not
bypass symlink checks or record immutability. JSON payload secret-shaped keys are
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
