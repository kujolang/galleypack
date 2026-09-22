# Agent instructions

Keep CLI, domain behavior, validation, storage, fixtures, release checks, and tests in Kujo. Preserve immutable records, append-only history, atomic writes, bounded I/O, path/symlink protection, offline behavior, and authority boundaries. Run `/Users/robertdevore/2026/Kujolang/kujo-repos/kujo/target/release/kujo run tests/test.kujo` and `git diff --check`. Never force-push or use live credentials in tests.

Use `bash scripts/validate.sh` for the full gate (including CLI, byte bounds and
24-process contention). `KUJO_BIN` selects the runtime. Select individual
`tests/*_test.kujo` files for focused checks; CLI/concurrency tests take the runtime
path after `--`. Run `kujo run scripts/benchmark.kujo` or append `-- tree` for
reproducible corruption-page and 10,001-file measurements. Preserve all page
cursors/warnings. Trust and crash-recovery limits are in `docs/security.md`.
Use the runtime revision in CI; version 1.4.0 alone is not a capability check.
`tests/recovery_test.kujo -- <runtime>` kills isolated workers at transaction
boundaries. Keep journal publication before records/events; preserve per-ID
process locking, stable lock inodes and no-replace replay. See `docs/recovery.md`.
