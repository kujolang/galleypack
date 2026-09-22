# Quickstart

Run from the repository root with a fresh operator-selected state directory:

```sh
./bin/galleypack init --state /tmp/galleypack-demo --json
./bin/galleypack add --state /tmp/galleypack-demo --input fixtures/core.json \
  --path fixtures/core.json --actor operator \
  --timestamp 2026-08-14T00:00:00Z --json
./bin/galleypack validate --state /tmp/galleypack-demo --json
```

The fixture is also the bound artifact in this example. The fixed timestamp makes
its ID deterministic; repeating the add command is rejected. To use another
runtime, set `KUJO_BIN=/absolute/path/to/kujo`. The launcher otherwise checks PATH,
then the sibling Kujo release build.
