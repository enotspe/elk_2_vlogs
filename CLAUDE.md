# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

A single Bash script (`elk_2_vlogs.sh`) that migrates data from Elasticsearch to VictoriaLogs. It splits a time range into chunks and runs one worker process per chunk in parallel, using `search_after` pagination and atomic state files for resumability.

## Running the script

```bash
chmod +x elk_2_vlogs.sh
./elk_2_vlogs.sh
```

To start fresh (discard all resume state):
```bash
rm -rf ./migration_state
```

## Dependencies

`curl`, `jq`, and GNU `date` (not BSD date — macOS requires `brew install coreutils`).

## Configuration

All user-tunable variables are in the `# --- CONFIGURATION ---` block at the top of `elk_2_vlogs.sh`. Key ones:

- `ES_HOST`, `ES_INDEX`, `ES_USER`/`ES_PASS` — Elasticsearch source
- `VL_HOST` — VictoriaLogs destination
- `START_DATE`, `END_DATE` — ISO 8601 migration window
- `MAX_WORKERS` — parallel worker count (defaults to `nproc`)
- `STATE_DIR` — where per-worker `.state` files are written (`./migration_state/`)
- `VL_STREAM_FIELDS`, `VL_TIME_FIELD`, `VL_MSG_FIELD` — VictoriaLogs header semantics
- `DEBUG_MODE` — set `true` to enable `set -x` per worker

## Architecture

**Main process** divides `[START_DATE, END_DATE]` into `MAX_WORKERS` equal millisecond chunks, then forks one `run_worker` subshell per chunk in the background and calls `wait`.

**Each worker** loops:
1. Reads its state file (if present) to get the last `search_after` cursor
2. Queries ES with a range filter + `search_after` + sort on `[<timestamp>, _shard_doc]`
3. Writes the new cursor atomically (`tmp` → `mv`) to the state file
4. Builds NDJSON bulk payload via `jq` and POSTs to VictoriaLogs `/insert/elasticsearch/_bulk`
5. Repeats until `hits_count == 0`, then deletes its state file

**Resumability**: state files survive crashes. Re-running the script replays only incomplete workers from their last cursor. Completed workers have their state file deleted on finish; re-running them will query ES once, get 0 hits, and exit.

**Variables are exported** with `export -f run_worker ...` so the subshell workers inherit all configuration.
