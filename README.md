# elk_2_vlogs

A parallel, resumable migration script for moving data from **Elasticsearch** to **VictoriaLogs**.

## What it does

`elk_2_vlogs.sh` splits a user-defined time range into equal chunks and launches one worker process per chunk. Each worker independently paginates through its slice of the Elasticsearch index using `search_after` and ships batches to VictoriaLogs via the ES Bulk API compatibility endpoint (`/insert/elasticsearch/_bulk`).

Key properties:

- **Parallel** — workers run concurrently, one per CPU core by default
- **Resumable** — atomic state files in `./migration_state/` record each worker's position; re-running the script skips already-completed work

## Where to run the script

Run the script **on or close to the VictoriaLogs host**. Parallelism is applied to the Elasticsearch side — each worker opens its own connection to ES and reads independently. The write path to VictoriaLogs is a single sequential POST per batch per worker, which is fast and rarely the bottleneck.

```
  [ES cluster]  ←── many parallel reads ───  [this script]  ──── bulk POST ────→  [VictoriaLogs]
   (slow)                                     (run here)                            (fast)
```

> **Field observation:** Retrieval from ES can be significantly slower than ingestion into VictoriaLogs, regardless of the number of workers. The script works correctly, but overall migration speed is limited by how fast ES can serve `search_after` pages.

## Dependencies

| Tool | Purpose |
|------|---------|
| `curl` | HTTP requests to Elasticsearch and VictoriaLogs |
| `jq` | JSON parsing and bulk payload construction |
| `date` (GNU coreutils) | Date arithmetic and ISO 8601 formatting |

**Debian / Ubuntu:**

```bash
sudo apt-get update && sudo apt-get install -y curl jq coreutils
```

**CentOS / RHEL:**

```bash
sudo yum install -y curl jq coreutils
```

> **macOS note:** The system `date` on macOS is BSD-based and does not support the `-d` flag. Install GNU coreutils via Homebrew (`brew install coreutils`) and replace `date` calls with `gdate`, or run the script in a Linux container.

## Quick start

```bash
# 1. Clone
git clone https://github.com/YOUR_USERNAME/elk_2_vlogs.git
cd elk_2_vlogs

# 2. Configure
cp .env.example .env
$EDITOR .env

# 3. Make executable and run
chmod +x elk_2_vlogs.sh
./elk_2_vlogs.sh
```

## Configuration

Settings are read from a `.env` file in the same directory as the script. A `.env.example` template with all available variables is included — copy it to `.env` and edit before running.

The script loads `.env` automatically on startup. You can also override any variable inline:

```bash
ES_INDEX=logs-2025 MAX_WORKERS=16 ./elk_2_vlogs.sh
```

Priority order (highest to lowest):

1. Variables set in the shell environment before running the script
2. Variables set in `.env`
3. Defaults baked into the script

> **Security:** `.env` may contain credentials — add it to `.gitignore` and do not commit it.

### Configuration reference

| Variable | Default | Description |
|----------|---------|-------------|
| `ES_HOST` | `http://localhost:9200` | Elasticsearch base URL |
| `ES_INDEX` | `your-index-name` | Index or data stream to read from |
| `ES_USER` | `elastic` | Elasticsearch username |
| `ES_PASS` | `password` | Elasticsearch password |
| `VL_HOST` | `http://localhost:9428` | VictoriaLogs base URL |
| `START_DATE` | `2024-01-01T00:00:00.000Z` | Start of the migration window (ISO 8601) |
| `END_DATE` | `2024-01-31T23:59:59.999Z` | End of the migration window (ISO 8601) |
| `TIMESTAMP_FIELD` | `@timestamp` | Document field used for time-range queries |
| `SORT_ORDER` | `asc` | Sort direction within each worker (`asc` or `desc`) |
| `PAGE_SIZE` | `1000` | Documents per Elasticsearch query |
| `MAX_WORKERS` | `$(nproc)` | Number of parallel workers (defaults to CPU count) |
| `STATE_DIR` | `./migration_state` | Directory for per-worker state files |
| `VL_STREAM_FIELDS` | *(unset)* | Comma-separated fields that form the VictoriaLogs log stream identity. If unset, the `VL-Stream-Fields` header is not sent and all logs go into a single default stream. |
| `VL_MSG_FIELD` | *(unset)* | Field VictoriaLogs should use as the human-readable log message. If unset, VictoriaLogs falls back to its default (`_msg`). |
| `VL_EXTRA_FIELDS` | *(unset)* | Additional `key=value` pairs to inject into every log entry |
| `VL_ACCOUNT_ID` | *(unset)* | VictoriaLogs `AccountID` header for multi-tenancy |
| `VL_PROJECT_ID` | *(unset)* | VictoriaLogs `ProjectID` header for multi-tenancy |
| `DEBUG_MODE` | `false` | Set to `true` to enable `set -x` verbose output per worker |

## How it works

```
┌─────────────────────────────────────────┐
│  Main process                           │
│  1. Parse START_DATE / END_DATE         │
│  2. Divide total range into N chunks    │
│  3. Spawn one worker per chunk (bg)     │
│  4. wait for all workers to finish      │
└────────────┬────────────────────────────┘
             │  (one per chunk)
     ┌───────▼────────┐
     │  Worker N      │
     │                │
     │  ┌──────────┐  │
     │  │  Check   │  │  ← reads state file if present (resume)
     │  │  state   │  │
     │  └────┬─────┘  │
     │       │        │
     │  ┌────▼─────┐  │
     │  │  ES      │  │  ← GET /{index}/_search with range + search_after
     │  │  search  │  │
     │  └────┬─────┘  │
     │       │        │
     │  ┌────▼─────┐  │
     │  │  Write   │  │  ← atomic mv to state file
     │  │  state   │  │
     │  └────┬─────┘  │
     │       │        │
     │  ┌────▼─────┐  │
     │  │  Build   │  │  ← jq: {"create":{}}\n<source>\n ...
     │  │  bulk    │  │
     │  └────┬─────┘  │
     │       │        │
     │  ┌────▼─────┐  │
     │  │  POST to │  │  ← POST /insert/elasticsearch/_bulk
     │  │  VL bulk │  │
     │  └────┬─────┘  │
     │       │        │
     │  repeat until  │
     │  hits == 0     │
     └───────┬────────┘
             │
     delete state file (worker done)
```

**search_after pagination** — the script avoids deep pagination performance issues by sorting on `[<timestamp_field>, _shard_doc]` and passing the last document's sort values into every subsequent query.

**Atomic state writes** — state is written to `<state_file>.tmp` then renamed with `mv`, so a crash mid-write never leaves a corrupt state file.

## Resuming an interrupted run

Simply re-run the script:

```bash
./elk_2_vlogs.sh
```

Workers that already completed have no state file and are skipped (they finish immediately with 0 documents). Workers that were interrupted read their state file and resume from the last successfully sent batch.

To start completely fresh, delete the state directory:

```bash
rm -rf ./migration_state
```

## VictoriaLogs header semantics

VictoriaLogs uses HTTP request headers to understand the structure of incoming bulk data:

| Header | Variable | Purpose |
|--------|----------|---------|
| `VL-Stream-Fields` | `VL_STREAM_FIELDS` | Comma-separated field names whose values are combined to form the **log stream** label set. Analogous to Loki label selectors. Choose low-cardinality fields (e.g. log type, source host). |
| `VL-Time-Field` | `VL_TIME_FIELD` | Field containing the log timestamp. VictoriaLogs parses this and uses it as the native log time. |
| `VL-Msg-Field` | `VL_MSG_FIELD` | Field whose value becomes the primary human-readable log message shown in queries and the UI. |
| `VL-Extra-Fields` | `VL_EXTRA_FIELDS` | Static `key=value` pairs appended to every log entry — useful for tagging migrated data (e.g. `source=migration`). |
| `AccountID` | `VL_ACCOUNT_ID` | Multi-tenant account identifier (leave blank for single-tenant deployments). |
| `ProjectID` | `VL_PROJECT_ID` | Multi-tenant project identifier (leave blank for single-tenant deployments). |
