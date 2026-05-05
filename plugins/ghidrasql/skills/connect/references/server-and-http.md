# Server and HTTP

## Endpoint Roles

Two distinct HTTP-speaking services in a normal live setup:

- **`LibGhidraHost` RPC**, default `http://127.0.0.1:18080` — speaks protobuf RPC, **not** SQL. Only `ghidrasql` should connect here.
- **`ghidrasql` SQL HTTP server**, default `http://127.0.0.1:8081` — raw SQL via `POST /query` plus eight other endpoints.

`ghidrasql --url http://127.0.0.1:18080` is the right way to issue SQL through the upstream host: the CLI speaks the RPC dialect and runs the SQL engine locally.

Raw SQL over HTTP requires proxy or managed mode:

```bash
# Proxy mode: attach to an existing host, expose a SQL HTTP server
ghidrasql --url http://127.0.0.1:18080 --http --port 8081 --bind 127.0.0.1

curl -X POST http://127.0.0.1:8081/query --data "SELECT * FROM db_info;"
```

Wrong (18080 is RPC, not SQL):

```bash
curl -X POST http://127.0.0.1:18080/query --data "..."
```

## All Endpoints (9)

| Method | Path | Purpose |
|---|---|---|
| GET | `/` | Welcome + brief help |
| GET | `/help` | Endpoint documentation |
| POST | `/query` | Execute SQL (body = raw SQL, response = JSON; multi-statement supported) |
| GET | `/status` | Server status + program metadata |
| GET | `/health` | Liveness probe — process is up. Always `{"status":"ok"}`; does **not** probe the query worker |
| GET | `/health/deep` | Readiness probe — reflects query-worker state. Returns `healthy: true|false`, `active_queries`, `oldest_query_age_ms`, `threshold_ms`. HTTP 503 when unhealthy |
| POST | `/refresh` | Drop ghidrasql caches and reload |
| POST | `/shutdown` | Stop the SQL server (semantics depend on launch mode — see Shutdown) |
| GET | `/shutdown/status` | Shutdown lifecycle observability — returns `phase` (`idle` / `http_stopping` / `java_exiting` / `complete` / `force_killed`), `listener_running`. Available until the listener fully stops |

`/health` and `/health/deep` serve different roles. `/health` is the dumb liveness probe (load-balancer friendly, always 200). `/health/deep` is the readiness probe — it observes the in-flight `/query` worker and flips `healthy: false` (HTTP 503) when the oldest active query has been running longer than `deep_health_threshold_ms` (default 5000 ms). Use `/health` for "is the process alive?" and `/health/deep` for "is it ready to take work?".

`/query` accepts one or more SQL statements in the body. The response always wraps per-statement results in a `results` array — single-statement bodies use the same shape with `statement_count: 1`.

Success:

```json
{
  "success": true,
  "statement_count": 1,
  "row_count_total": 1,
  "results": [
    {
      "success": true,
      "columns": ["name", "address"],
      "rows": [["main", "0x4012C0"]],
      "row_count": 1,
      "elapsed_ms": 12,
      "partial": false,
      "timed_out": false
    }
  ]
}
```

Multi-statement success carries one entry per statement in `results[]`. `row_count_total` sums per-statement `row_count` across the array.

Failure (one of the statements errored):

```json
{
  "success": false,
  "error": "<message>",
  "statement_count": 3,
  "row_count_total": 0,
  "results": [
    { "success": true,  ... },
    { "success": false, "error": "..." }
  ]
}
```

`results[]` carries the partial run up to and including the failing statement; statements after the failure are not executed.

## Default Port

The SQL HTTP server defaults to **`8081`**. Use `--port <N>` to override if `8081` conflicts with other tooling on the host; either way the default is the right starting point unless you have a specific reason.

## Environment Conflict

If `GHIDRA_INSTALL_DIR` is exported, the CLI auto-derives `--ghidra` and refuses `--url`:

```
error: --ghidra and --url are mutually exclusive
```

Fix per-invocation:

```bash
unset GHIDRA_INSTALL_DIR
# or
env -u GHIDRA_INSTALL_DIR ghidrasql --url http://127.0.0.1:18080 ...
```

## REPL

```bash
ghidrasql --url http://127.0.0.1:18080
```

Nine dot-commands:

- `.help` / `.h`
- `.tables`
- `.schema <table>`
- `.info`
- `.save`
- `.discard`
- `.refresh`
- `.http start` / `.http stop`
- `.quit` / `.exit` / `.q`

## HTTP Modes

### Proxy mode (attach + expose SQL HTTP)

```bash
ghidrasql --url http://127.0.0.1:18080 --http --port 8081 --bind 127.0.0.1
```

ghidrasql speaks RPC to the existing host and exposes its SQL engine on port 8081. **Stopping `/shutdown` only stops the SQL proxy** — the upstream LibGhidraHost keeps running.

### Managed mode (spawn host + expose SQL HTTP)

```bash
ghidrasql --ghidra "$GHIDRA_INSTALL_DIR" \
          --project <project_dir> --project-name <name> \
          --program <prog>.exe --no-analyze \
          --http --port 8081 --max-runtime 0 &
```

In a multi-program project, pass a Ghidra domain path to `--program` and use
`--initial-program` when several imports/programs are supplied:

```bash
ghidrasql --ghidra "$GHIDRA_INSTALL_DIR" \
          --project <project_dir> --project-name <name> \
          --binary loader.dll --binary payload.exe \
          --initial-program /payload.exe \
          --http --port 8081 --max-runtime 0 &
```

Discover project contents with `--list-project-programs` or the metadata tables
`project_files` and `project_programs`. All other SQL tables remain scoped to
the one active program.

ghidrasql owns both the SQL server *and* the upstream Java host. Send raw SQL to the proxy:

```bash
curl -X POST http://127.0.0.1:8081/query --data "SELECT * FROM db_info;"
```

For read-only experiments, add `--readonly` (which implies `--shutdown discard`).

## Shutdown

```bash
curl -X POST http://127.0.0.1:8081/shutdown
```

The HTTP listener stops within ~150 ms. **What happens to the Java host depends on which mode launched ghidrasql:**

- **Managed mode (`--ghidra ...`)** — after the SQL server stops, the upstream headless host is closed using the launch-time policy (`--shutdown save|discard|none`, default `save`). For large pending state this can take many seconds. Wait for both `java.exe` and `ghidrasql.exe` to disappear before reusing the project directory.
- **Proxy mode (`--url ...`)** — only the SQL proxy stops. The upstream LibGhidraHost is **untouched**. To stop it, hit its own RPC or kill the Java process if you started it.

There is **no SQL `shutdown(...)` function**. Set the exit policy at launch via `--shutdown` and trigger the actual stop with the HTTP endpoint above (or by exiting the REPL after `SELECT save_database()`).

If you must restart after a crash or force-kill, delete leftover `*.lock` / `*.lock~` in the project dir before re-launching.

## Notes

- `/query` expects **raw SQL** in the request body, not JSON.
- Keep the `--url` upstream as the source of truth — it is the only thing that has a real connection to Ghidra.
- For quick debugging when HTTP behaviour is unclear, run a direct CLI query (`ghidrasql --url ... -q "SELECT ..."`) to bisect whether the issue is in the proxy or the upstream host.
