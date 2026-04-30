# SQL Functions

Complete catalog. **22 distinct names, 23 registrations** (`search_snippet` registers at arity 2 *and* 3 sharing one name).

Verify live with:

```sql
SELECT name, narg FROM pragma_function_list
WHERE name IN ('decompile','rename_local','set_local_type','parse_decls',
               'cache_stats','cache_invalidate','cache_invalidate_all',
               'save_database','discard_changes','refresh_database',
               'program_revision','rebuild_strings','string_count','hex',
               'normalize_text','search_match','search_score','search_snippet',
               'search_rank','type_family','type_is_pointer','type_strip_cv')
ORDER BY name, narg;
```

## Decompiler Variable Mutation

- `rename_local(func_addr, local_id, new_name)` — rename via direct RPC. Equivalent to `UPDATE decomp_lvars SET name = ... WHERE func_addr = ... AND local_id = ...`.
- `set_local_type(func_addr, local_id, new_type)` — same shape, type edit.

## Decompile

- `decompile(func_addr)` — returns the full pseudocode text. **Single argument only — no force-flag.** Faster than `SELECT text FROM pseudocode WHERE func_addr = ...` when you only need the text.

In one-shot use each `/query` rebuilds its tables, so a `decompile()` call after a write already reflects the write. Inside a batched script, drop the cached surface first:

```sql
SELECT cache_invalidate('pseudocode');
SELECT decompile(0x401000);
```

## Type Import

- `parse_decls(source_text)` — runs C declarations through Ghidra's CParser. Standard C only; no preprocessor, no `#include`. Returns `count_of_newly_created_types` (0 if everything already existed) or `-1` on parse error. Verify by re-reading `types`.

## Search / Full-text

- `normalize_text(text)` — squashes whitespace, lowercases, strips non-alphanumeric. Same routine the search index uses.
- `search_match(haystack, query)` — `1` if every term in `query` appears in `haystack` after normalization, else `0`.
- `search_score(haystack, query)` — term-frequency relevance score.
- `search_snippet(text, query)` — context window around the first match. Default radius 48.
- `search_snippet(text, query, radius)` — same with explicit radius.
- `search_rank(domain, text, query)` — domain-weighted rank.

## Type Analysis

- `type_family(decl)` — classifier returning exactly one of these 10 strings:
  ```
  aggregate | enum | alias | function | pointer | boolean
  floating  | integral | void | unknown
  ```
- `type_is_pointer(decl)` — `0`/`1`.
- `type_strip_cv(decl)` — removes `const` / `volatile` / `restrict` / `mutable`, preserves `*` and `&`.

## Formatting / Housekeeping

- `hex(value)` — `"0x"`-prefixed uppercase hex string for an int64. Note: SQLite ships its own `hex()` (returns lowercase BLOB hex). ghidrasql overrides for int64 inputs; verify with `SELECT hex(0x401000)` returning `"0x401000"`.
- `program_revision()` — engine session revision counter. Matches `db_info.revision`.
- `string_count()` — live count of detected strings.
- `rebuild_strings()` — refresh string-table cache and return new count.

## Source and Lifecycle

- `save_database()` — commit pending mutations to the Ghidra project. Returns `1` on success. **Caveat — verify before trusting:** in some configurations `save_database()` can return `1` without the change persisting after a project reopen. To prove a write survived, save → shut down the host → reconnect with `--readonly --no-analyze` → re-query the row and compare.
- `discard_changes()` — roll back pending mutations. Returns `1`.
- `refresh_database()` — invalidate caches and reload from source. Returns `1`.

## Cache Helpers

These three are real SQL functions — `pragma_function_list` shows them.

- `cache_stats()` — returns a JSON string with `cache_invalidations_total` (cumulative counter), `last_seen_revision`, `source_revision`, `schema_tables` (list of cacheable tables).
- `cache_invalidate('<table>')` — drop one table's cache so the next read inside the same batch pulls fresh data. Returns `0`/`1`.
- `cache_invalidate_all()` — drop every cached table. Returns `0`/`1`.

**When this matters:** materialisation is **query-scoped** in normal one-shot use — every `/query` call (or `-q "..."`) rebuilds whatever tables it touches and discards them when the statement returns. The cache helpers only have an effect **inside a batched script** (`-f`, multi-statement REPL input) where the same materialisation persists across statements.

Worked example inside a batch (`-f`):

```sql
SELECT set_local_type(0x401000, 'arg0', 'char *');   -- write
SELECT cache_invalidate('decomp_lvars');              -- drop the cache built earlier in the same batch
SELECT local_id, name, type FROM decomp_lvars         -- fresh read
WHERE func_addr = 0x401000;
```

## Notes

- Saving is explicit. **Nothing auto-commits** — even after `parse_decls()`, `rename_local()`, `set_local_type()` you still need `save_database()` to persist.
- `parse_decls()`, `rename_local()`, `set_local_type()` already go through the Ghidra type system / decompiler — in one-shot use, the next query already sees the new state. Inside a batch script, `cache_invalidate('<table>')` between the write and a re-read is the right tool.
- There is **no `shutdown(...)` SQL function**. Stop the server with `POST /shutdown`; the launch-time `--shutdown save|discard|none` policy decides what happens to the project on exit in managed mode (in proxy mode the upstream host is untouched).
