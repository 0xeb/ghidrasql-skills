---
name: functions
description: "Reference ghidrasql SQL functions and choose the right helper for a task."
allowed-tools:
  - Bash
  - Read
  - Glob
  - Grep
---

# Functions

## Trigger Intents

Use this skill when the user asks:
- what SQL function exists for a task
- whether a helper function is better than a table write
- how to save, parse declarations, decompile, or manage source lifecycle

Route to:
- `decompiler` for decompile-centric workflows
- `types` for `parse_decls()` and signature-related work
- `annotations` for full mutation loops
- `grep` for full-text search functions (`search_*`)

## Catalog (22 functions)

ghidrasql registers **22 distinct names, 23 entries** (`search_snippet` registers at arity 2 *and* 3). 19 are general-purpose helpers; 3 are cache-control helpers.

There is **no** `shutdown(...)` function — set the exit policy at launch via `--shutdown` and trigger the stop with `POST /shutdown`. There is **no** force-flag on `decompile()` — it takes a single argument; use `cache_invalidate('pseudocode')` or `refresh_database()` to drop stale state.

### Decompiler

```sql
SELECT decompile(0x401000);                                 -- full pseudocode text
SELECT rename_local(0x401000, '<local_id>', 'buffer');       -- == UPDATE decomp_lvars SET name
SELECT set_local_type(0x401000, '<local_id>', 'char *');     -- == UPDATE decomp_lvars SET type
```

### Type import

```sql
SELECT parse_decls('typedef struct { int x; int y; } Point;');
```

### Search (full-text on tokens / strings / metadata)

```sql
SELECT normalize_text('SomeMixed_Case');                     -- canonical form
SELECT search_match(haystack, query);                        -- 0/1 — every term matches
SELECT search_score(haystack, query);                        -- relevance score
SELECT search_snippet(haystack, query);                      -- snippet, default radius
SELECT search_snippet(haystack, query, radius);              -- snippet, custom radius
SELECT search_rank(domain, haystack, query);                 -- domain-weighted rank
```

### Type analysis (declaration introspection)

```sql
SELECT type_family('struct foo { int x; }');                 -- aggregate|enum|alias|...
SELECT type_is_pointer('char *');                            -- 0/1
SELECT type_strip_cv('const volatile int *');                -- strips const/volatile
```

### Formatting / housekeeping

```sql
SELECT hex(0x401000);                                        -- '0x401000'
SELECT program_revision();                                   -- engine revision counter
SELECT string_count();                                       -- live string-table size
SELECT rebuild_strings();                                    -- refresh string table
```

### Persistence and lifecycle

```sql
SELECT save_database();                                      -- commit pending mutations
SELECT discard_changes();                                    -- roll back pending mutations
SELECT refresh_database();                                   -- invalidate caches, reload
```

### Cache helpers

Materialisation is query-scoped in normal one-shot use — each `/query` (or `-q`) rebuilds the tables it touches. The cache helpers matter mainly inside a batched script (`-f`, multi-statement REPL input) where the cache persists across statements:

```sql
SELECT cache_stats();                                        -- JSON: invalidations_total, revision, cacheable tables
SELECT cache_invalidate('pseudocode');                       -- drop one table's cache; useful inside a batch after a write
SELECT cache_invalidate_all();                               -- drop every cached table
```

## When to Prefer Functions Over Direct Table Writes

- Use `decompile(addr)` for a quick full-function read instead of
  `SELECT text FROM pseudocode WHERE func_addr = addr`.
- Use `rename_local()` / `set_local_type()` when a function-shaped
  call is clearer than `UPDATE decomp_lvars`.
- Use `parse_decls()` for any C declaration import — it goes through
  Ghidra's CParser and avoids per-row INSERTs.
- Use `save_database()` explicitly after a mutation batch.

## Critical Rules

- Function helpers do not replace the need to verify affected rows or
  pseudocode after mutation.
- Saving is explicit. Nothing auto-commits.
- `func_addr` discipline still applies to decompiler-backed tables
  (`pseudocode`, `decomp_lvars`, `decomp_tokens`) even when you went
  through a helper function.

## Additional Resources

- Function families and examples: [references/sql-functions.md](references/sql-functions.md)
