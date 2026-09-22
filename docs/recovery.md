# Transaction recovery

Use the Kujo revision pinned in CI (`cf785c0a7953717af16b657cda05b85d628144c5`)
or a compatible later runtime. Older 1.0.x runtimes and early 1.4.0 builds lack
required APIs. Install/upgrade the runtime before changing state; record and
history JSON formats, IDs, checksums and artifact paths do not need migration.
An undefined `list_dir_page`, `file_lock` or `publish_file_noreplace` error means
the runtime needs this upgrade; it is not a reason to alter stored records.
Do not run old binaries as recovery tools. The additive managed names
`transactions` and `writer-locks` must be regular directories; a preexisting
conflicting file or symlink is rejected and must be inspected before migration.

## Operator workflow

```sh
./bin/galleypack recover --state /path/to/state --dry-run --json
./bin/galleypack recover --state /path/to/state --json
# Target one ID or continue a bounded page:
./bin/galleypack recover --state /path/to/state --id artifact-example --json
./bin/galleypack recover --state /path/to/state --after LAST_ID --limit 100 --json
```

A preview creates no directories, lock files, records or events. Recovery processes
at most `--limit` IDs (default 100, maximum 1,000), with a bounded union of record,
journal, journal-stage and legacy-lock filenames. Retained directory heaps total
at most `4 × (limit + 2)` names; the merged candidates also remain O(limit).
Pages are not snapshots. Follow `next_after` whenever `truncated` is true, including
failed pages. Review each result: failures preserve evidence and do not prevent
independent IDs on that page from recovering.

`replay` reconstructs absent record/event files from a checked durable intent;
`repair_event` reconstructs a missing creation event from an existing legacy record;
`discard_uncommitted_stage` removes a stage that never became a published intent;
`remove_legacy_lock` removes an eligible abandoned marker; `already_consistent`
requires no repair. `not_found` means no record, intent, stage or lock exists for
the requested ID. Recovery never fetches or rewrites source artifacts.

Unrecognized legacy `locks/<id>.lock` files/directories cannot prove that their
writer died. Stop **all old writers**, inspect the preview, then explicitly use:

```sh
./bin/galleypack recover --state /path/to/state --id artifact-example --force --json
```

`--force` only permits removal of a regular legacy lock or empty legacy lock
directory. Symlinks, nonempty directories, malformed journals, oversized inputs,
changed record bytes and conflicting creation events still fail closed. Fix the
underlying permission/disk problem before retrying an I/O failure. Preserve a copy
of conflicting evidence for investigation; never delete records to make recovery pass.
A matching retry of the original mutation automatically replays its pending intent.
Once completed, another mutation with the same ID remains `duplicate_id`.

## Protocol and failure boundaries

1. Validate the managed state directories and obtain a nonblocking per-ID OS lock
   at `writer-locks/<id>.lock`. Different IDs can progress concurrently.
2. Reserve `locks/<id>.lock` using no-replace publication. New markers identify
   the journal protocol; old cooperating writers see the original exclusion path.
3. Publish `transactions/<id>.json` durably. It contains schema `1.0.0`, the exact
   serialized record text and its SHA-256 (see `schemas/transaction.schema.json`). The journal is capped at 2,162,688 bytes;
   the embedded record remains capped at 1 MiB. Paths are derived from validated
   IDs/timestamps, never arbitrary paths stored in the journal.
4. Publish the record and deterministic creation event separately, using synced
   staged bytes and checked no-replace publication receipts. Existing exact bytes
   are accepted on replay and their directory is synced again. Conflicts are errors.
5. Remove matching staging residues and the intent only after both publications
   are durable; clear the compatibility marker and release the OS lock.

`writer-locks` files intentionally persist. Their existence does not mean the
lock is held; the OS owns that state and releases it even after SIGKILL. Never
unlink these stable inodes while writers/recovery might be active. There is one
small file per ID that has attempted a write or recovery; this avoids one global
writer bottleneck and does not require PID guessing, expiry timers or sleeps.

Crashing before intent publication does not commit a record; recovery discards
unpublished journal staging. Crashing after intent publication permits exact
roll-forward. A crash after intent deletion may resurrect that directory entry
on an unsynced filesystem: replay is still safe because both data files were
already synced. Reads reject pending intents. Doctor and validation separately
check creation events, including old orphan records that have no journal.

The runtime syncs staged file contents and publication directories. Directory
creation is followed by an explicit checked publication barrier in its parent.
Initialization also syncs the managed-directory names and, until metadata exists,
the state root's parent (including when resuming interrupted initialization).
This barrier requires write access to that parent when first initializing a state.
Power-loss guarantees depend on the filesystem honoring these operations; tests
exercise SIGKILL at six protocol boundaries, not physical power removal. A crash
inside a runtime atomic-write/barrier primitive may leave uniquely named temporary
files; these are not committed intents or records and are not treated as evidence.
State roots and their ancestors remain operator controlled; this is not a sandbox
against hostile simultaneous directory replacement.

`src/transactions.kujo` exports internal protocol steps to share the production
implementation with crash workers. These are not independent public mutation APIs:
callers must validate managed paths and hold the corresponding per-ID process lock.
Use `save_new` or `recover` for supported storage operations.
