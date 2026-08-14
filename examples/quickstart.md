# Quickstart

`./bin/galleypack init --state /tmp/galleypack-demo --json`

`./bin/galleypack add --state /tmp/galleypack-demo --input fixtures/core.json --actor operator --timestamp 2026-08-14T00:00:00Z --json`

The fixed timestamp makes fixture IDs deterministic; repeating the command is rejected.
