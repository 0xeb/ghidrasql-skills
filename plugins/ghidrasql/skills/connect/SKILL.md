---
name: connect
description: "Connect to ghidrasql sources, verify live access, and route to the right analysis skill."
allowed-tools:
  - Bash
  - Read
  - Glob
  - Grep
---

# Connect

Entry point for every ghidrasql session. Bootstrap the source, verify the program is bound, then hand off to the right domain skill.

## Trigger Intents

Use this skill when the user wants to:
- start a `ghidrasql` session
- connect to a live `LibGhidraHost`
- run REPL, one-shot, or background HTTP server mode
- check whether a source is reachable
- pick the right follow-on skill

## Routing Matrix

Once `db_info` and `funcs > 0` come back, route by user intent:

| Intent | Skill | First query |
|---|---|---|
| triage / "what does this binary do?" | `analysis` | `SELECT * FROM db_info; SELECT name, size FROM funcs ORDER BY size DESC LIMIT 20;` |
| disassembly / blocks / CFG | `disassembly` | `SELECT start_ea, end_ea FROM blocks WHERE func_addr = 0xX;` |
| callers / callees / call graph / xrefs | `xrefs` | `SELECT * FROM callgraph_edges WHERE src_func_addr = 0xX;` |
| find by name pattern | `grep` | `SELECT name, address FROM funcs WHERE name LIKE '%X%';` |
| strings / bytes / data items / hexdump | `data` | `SELECT * FROM strings WHERE content LIKE '%X%' LIMIT 20;` |
| decompile / pseudocode / locals / params | `decompiler` | `SELECT decompile(0xX);` |
| ctree / AST patterns | `decompiler` | `SELECT * FROM ctree_v_calls WHERE func_addr = 0xX;` |
| rename / comment / signature / lvars | `annotations` | READ first, then UPDATE, then `SELECT save_database();` |
| struct / enum / typedef / parse C | `types` | `SELECT parse_decls('...'); SELECT * FROM type_members WHERE parent_type_name = 'X';` |
| breakpoints / patch bytes | `debugger` | `SELECT * FROM breakpoints;` |
| recursive bottom-up source recovery | `re-source` | starts with leaf callees and iterates |
| SQL helper lookup | `functions` | reference catalog (no warm-start) |

## Global Contracts

These five contracts apply across every ghidrasql skill. Skip them at your peril — they prevent the most common failure modes.

### 1. Read-First

Every mutation starts with a SELECT that yields the exact write coordinates. Resolve `func_addr`, `local_id`, `address` from a query, never from guesswork.

```sql
SELECT local_id, name, type FROM decomp_lvars WHERE func_addr = 0x401000;
-- now use the literal local_id from the result, not a guess
```

### 2. Anti-Guessing

Use `.tables`, `.schema <table>`, and `PRAGMA table_xinfo(<table>)` before issuing uncertain queries. ghidrasql exposes 60 virtual tables, 73 views, and 22 SQL functions — when a query fails, introspect first, then retry.

```sql
SELECT type, name, ncol FROM pragma_table_list WHERE schema='main';
PRAGMA table_xinfo('decomp_lvars');
SELECT name, narg FROM pragma_function_list WHERE name LIKE 'search_%';
```

### 3. Mandatory Mutation Loop

Read → Write → `SELECT save_database();` → re-read to verify.

```sql
SELECT name, signature FROM funcs WHERE address = 0x4011F0;     -- read
UPDATE funcs SET name = 'parseConfig' WHERE address = 0x4011F0;  -- write
SELECT save_database();                                           -- commit
SELECT name FROM funcs WHERE address = 0x4011F0;                  -- verify
```

**Caveat — verify saves explicitly:** `save_database()` returning `1` does not always mean the change persisted (in some configurations the save is silently dropped on reopen). To prove a critical write survived, run this loop:

```
SELECT save_database();
-- shut the host down (POST /shutdown for the SQL server, or exit the REPL)
-- reconnect: ghidrasql --ghidra ... --readonly --no-analyze ...
SELECT name FROM funcs WHERE address = 0xX;   -- compare against the post-write value
```

If the post-reopen value matches, the save is durable. If it reverts, re-apply the mutation and try again before continuing.

### 4. Performance

| Surface | Mandatory predicate | If you skip it |
|---|---|---|
| `pseudocode`, `decomp_lvars`, `decomp_tokens`, `decomp_comments` | `WHERE func_addr = X` | Decompiles every function. At ~500 ms per function, a binary with thousands of functions takes many minutes per query |
| `blocks`, `cfg_edges` | `WHERE func_addr = X` | Per-function CFG built for every function |
| `instructions` | `WHERE address BETWEEN A AND B` (post-filter scan) | Cache build over every instruction in the program |
| `xrefs`, `xref_index` | `WHERE from_ea = X` or `to_ea = X` (post-filter scan) | Cache build over every xref edge |
| `memory_bytes` | range (post-filter scan) | Full byte cache build |
| `comments` | `address = X` or range (post-filter scan) | Whole-comment-table scan |

`pseudocode`/`decomp_lvars`/`blocks`/`cfg_edges` *push* the filter into the source — without it, you trigger work proportional to the whole program. `instructions`/`xrefs`/`memory_bytes`/`comments` are post-filter scans — the filter narrows the **result set** but not the **work**. When the question is function-scoped, prefer the narrow entry-point views (`disasm_calls`, `disasm_blocks`, `function_calls`, `string_refs`, `callgraph_edges`) over raw tables.

See [references/cost-model.md](references/cost-model.md) for the full table with file:line citations.

### 5. Failure Recovery

| Symptom | Fix |
|---|---|
| `funcs` returns 0 rows | Program isn't bound — pass `--program <name>` (or `--binary` to import) and restart the host |
| Need to see every program in a project | Run `ghidrasql ... --list-project-programs`, or query `SELECT path,name FROM project_programs ORDER BY path;` |
| `--url` rejected with "mutually exclusive" | `unset GHIDRA_INSTALL_DIR` (or `env -u GHIDRA_INSTALL_DIR ghidrasql --url ...`) and retry |
| `analyzeHeadless` never prints `LIBGHIDRA_HEADLESS_READY` | Check the log for an exception, port conflict (18080 RPC, 8081 SQL), or unsupported binary format |
| Stale `*.lock` / `*.lock~` after force-kill | Confirm the owning `java.exe` PID is gone, then delete both lock files. **Never `taskkill /IM java.exe`** — that kills all JVMs on the machine, including other concurrent ghidrasql sessions. Target the specific PID instead |
| GUI host stalls under concurrent decompiler reads | Overlapping requests against `pseudocode`/`decomp_lvars`/`decomp_comments` can deadlock the host on a fair `ReentrantReadWriteLock`. Workaround: serialise decompiler-backed queries against the same host; do not run two ghidrasql clients hitting decompiler tables in parallel |
| Query returns stale rows after a write or external edit | Check `SELECT program_revision()` and `SELECT cache_stats();`. Libghidra live sources refresh automatically when the native freshness token changes; use `SELECT cache_invalidate('<table>');` or `refresh_database()` to force a rebuild, especially inside batched scripts |
| Host hangs after a wide `comments` range query | `getComments()` over wide ranges can hang the host. Use exact-address `WHERE address = X` instead of range scans |
| `/query` HTTP layer wedges while `/health` still answers | Recovery path: spawn a fresh `ghidrasql --url http://127.0.0.1:<rpc-port>` directly against the LibGhidraHost RPC port (`unset GHIDRA_INSTALL_DIR` first if set). The new process bypasses the wedged HTTP layer and can run `SELECT save_database()` to commit pending state before the orphaned tree is killed |
| Individual RPC wedges (e.g. a pathological signature parse) tie up the worker | Lower the per-RPC timeout via `--rpc-timeout-ms <N>` (default 0 → libghidra's 120 s). Wedged calls fail with a timeout error and the worker frees up; observable via `/health/deep` |
| Local rename / type apply over-propagates through reused decompiler temporaries | The decompiler shares storage between source-level variables that happen to use the same register/stack slot, so `apply_type_local` (or a direct `decomp_lvars` UPDATE) on one site can change unrelated locals. Mitigation: back the local out to a neutral type (e.g. `char *`) and keep the recovered struct on prototypes; verify in `pseudocode` after each edit |
| Pointer-return signature update fails with `Can't parse name: *fn` | Ghidra's CParser tokenises `<type> *fn(...)` differently from `<type>* fn(...)`. Workaround: use `char* fn` form (asterisk on the type, not the name). Same applies to `void **`, `int *`, etc. |

## Pre-flight

If `GHIDRA_INSTALL_DIR` is set in the environment, the CLI auto-fills `--ghidra` from it and rejects `--url` as conflicting. For any `--url`-mode invocation either prefix with `unset GHIDRA_INSTALL_DIR;` or use `env -u GHIDRA_INSTALL_DIR ghidrasql --url ...`.

## Do This First

Prove the source is alive and learn the current program:

```bash
ghidrasql --url http://127.0.0.1:18080 -q "SELECT * FROM db_info;"
ghidrasql --url http://127.0.0.1:18080 -q "SELECT COUNT(*) AS n FROM funcs;"
```

Treat `funcs > 0` as the real "program is bound" signal. `db_info.program_name` may legitimately read `"active-program"` (host placeholder) — that alone is not a failure.

## Connection Modes

Three mutually exclusive modes. Pick one.

### Attach to a running host (`--url`)

```bash
ghidrasql --url http://127.0.0.1:18080                 # REPL
ghidrasql --url http://127.0.0.1:18080 -q "<sql>"      # one-shot
ghidrasql --url http://127.0.0.1:18080 -f script.sql   # script
```

### Spawn a headless host against a project (`--ghidra`)

```bash
ghidrasql --ghidra <ghidra_dist> \
          --project <project_dir> --project-name <name> \
          --program <prog>.exe --no-analyze \
          -q "SELECT name FROM funcs LIMIT 5;"
```

Use Ghidra domain paths for multi-program projects:

```bash
ghidrasql --ghidra <ghidra_dist> \
          --project <project_dir> --project-name <name> \
          --program /samples/payload.exe --no-analyze \
          -q "SELECT * FROM db_info;"
```

`--binary` and `--program` are repeatable. A host still has one active program; choose it with `--initial-program` when importing or naming more than one target:

```bash
ghidrasql --ghidra <ghidra_dist> \
          --project <project_dir> --project-name <name> \
          --binary loader.dll --binary payload.exe \
          --initial-program /payload.exe --http
```

List project programs without changing the program-scoped SQL model:

```bash
ghidrasql --url http://127.0.0.1:18080 --list-project-programs
ghidrasql --url http://127.0.0.1:18080 \
  -q "SELECT path,name,folder_path FROM project_programs ORDER BY path;"
```

All analysis tables (`funcs`, `pseudocode`, `xrefs`, etc.) describe the active program only. To switch, save/close and reopen a different domain path, or re-invoke `ghidrasql` with another `--program`.

### Background HTTP SQL server (the "keep it running" recipe)

```bash
ghidrasql --ghidra <ghidra_dist> \
          --project <project_dir> --project-name <name> \
          --program <prog>.exe --no-analyze \
          --http --port 8081 --max-runtime 0 &
```

`--max-runtime 0` disables the auto-exit timer. The SQL endpoint becomes `POST http://127.0.0.1:8081/query` (default port). For read-only experiments add `--readonly`, which implies `--shutdown discard` and prevents accidental mutations being saved.

## Endpoint Model

| URL | What it speaks | Who talks to it |
|---|---|---|
| `http://127.0.0.1:18080` | LibGhidraHost protobuf RPC | `ghidrasql` only |
| `http://127.0.0.1:8081/query` | raw SQL (POST body) | any HTTP client |

Wrong: `curl -X POST http://127.0.0.1:18080/query ...` — the upstream port speaks RPC, not SQL.

The SQL HTTP server exposes nine endpoints:

| Method | Path | Purpose |
|---|---|---|
| GET | `/` | Welcome + brief help |
| GET | `/help` | Endpoint documentation |
| POST | `/query` | Execute SQL (body = raw SQL, response = JSON) |
| GET | `/status` | Server status + program info |
| GET | `/health` | Liveness probe (`{"status":"ok"}`) — does not probe the query worker |
| GET | `/health/deep` | Readiness probe — reflects query-worker state, returns 503 when the oldest in-flight query exceeds the configured threshold |
| GET | `/shutdown/status` | Lifecycle observability — `phase` (idle / http_stopping / java_exiting / complete / force_killed), `listener_running`. Useful for polling during managed-mode shutdowns |
| POST | `/refresh` | Drop ghidrasql caches and reload |
| POST | `/shutdown` | Stop the SQL server (see Session End for managed-vs-proxy semantics) |

## Session End

Save before stopping (skip this if you started with `--readonly`):

```sql
SELECT save_database();
```

Stop the HTTP server:

```bash
curl -X POST http://127.0.0.1:8081/shutdown
```

`/shutdown` returns `{"success":true}` once the listener is stopping. **What happens next depends on launch mode:**

- **Managed mode (`--ghidra ... --http`):** ghidrasql owns both the SQL server and the upstream headless Java host. After the SQL server stops, the headless host is closed using the launch-time `--shutdown save|discard|none` policy (default `save`). For large pending state this can take many seconds. Wait for both `java.exe` and `ghidrasql.exe` to actually exit before reusing the project directory.
- **Proxy mode (`--url ... --http`):** ghidrasql owns only the SQL proxy. `/shutdown` stops the proxy. The upstream LibGhidraHost is **untouched** — it keeps running. Stop it via its own RPC or kill the Java process if you own it.

**Never `taskkill /F` Java without saving first** — that discards changes since the last `save_database()` and leaves orphaned `*.lock` / `*.lock~` files in the project dir.

**Never kill by image name in parallel workflows.** Commands like
`taskkill /F /IM java.exe` or `taskkill /F /IM ghidrasql.exe` kill
**every** instance on the machine, destroying other concurrent sessions.
If you must kill a stuck process, target the specific PID only.  If a
port is already occupied by another session, choose a different port —
do not kill the occupant.

## What ghidrasql does NOT have

These exist in adjacent tooling (idasql) but are explicit non-features here:

- **No LocalClient transport.** ghidrasql talks only to LibGhidraHost over HTTP+protobuf. There is no offline-decompiler SQL path.
- **No `idapython()` analog.** Java-side scripting is not exposed through ghidrasql.
- **No `ui-context()` function.** The GUI plugin does not export selection / widget / cursor state.
- **No `shutdown()` SQL function.** Exit policy is fixed at launch (`--shutdown save|discard|none`); trigger the actual stop with `POST /shutdown`.
- **No force-flag on `decompile()`.** It is a 1-arg function. Each `/query` invocation rebuilds the relevant tables, so a fresh `decompile()` after a write will reflect the write in normal one-shot use. Inside a batched script, call `SELECT cache_invalidate('pseudocode');` before the second `decompile()` if the first call materialised the cache.

## REPL

`ghidrasql --url http://127.0.0.1:18080` (or `--ghidra ...`) drops you into an interactive REPL. Nine dot-commands:

| Command | What |
|---|---|
| `.help` / `.h` | Help text |
| `.tables` | List virtual tables |
| `.schema <table>` | Show columns for one table |
| `.info` | Server + program metadata (same as `/status`) |
| `.save` | `save_database()` shortcut |
| `.discard` | `discard_changes()` shortcut |
| `.refresh` | `refresh_database()` shortcut |
| `.http start` / `.http stop` | Start / stop the embedded SQL HTTP server |
| `.quit` / `.exit` / `.q` | Leave the REPL |

## Common Patterns

```bash
# CLI one-shot
ghidrasql --url http://127.0.0.1:18080 \
  -q "SELECT name, printf('0x%X', address) AS addr FROM funcs LIMIT 10;"

# HTTP one-shot (requires --http server already running)
curl -X POST http://127.0.0.1:8081/query \
  --data "SELECT name, signature FROM funcs LIMIT 5;"

# Read-only experiment (no save, host discards on exit)
ghidrasql --ghidra "$GHIDRA_INSTALL_DIR" \
          --project /path/to/proj --project-name myproj --program sample.exe \
          --no-analyze --readonly --http --port 8081 --max-runtime 0 &
```

## Additional Resources

- HTTP and REPL details: [references/server-and-http.md](references/server-and-http.md)
- Per-table cost model: [references/cost-model.md](references/cost-model.md)
