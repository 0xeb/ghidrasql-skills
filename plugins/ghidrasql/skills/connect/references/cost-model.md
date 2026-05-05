# Cost Model

How ghidrasql's hot tables actually behave under filters. The headline distinction:

- **Pushdown** — the filter rewrites into the source RPC; only the matching rows are materialised. Cheap.
- **Post-filter scan** — the cache builder fetches every row, then SQLite filters the result. The filter narrows what you *see*, not what ghidrasql *does*. Expensive on large surfaces.

The numbers below are order-of-magnitude expectations, not benchmarks. Re-measure on the binary in front of you with a quick `SELECT COUNT(*)` per table.

## Per-table behaviour

| Surface | Predicate | Pushdown? | Reality |
|---|---|---|---|
| `pseudocode`, `decomp_lvars`, `decomp_tokens`, `decomp_comments` | `WHERE func_addr = X` | **Yes** | Per-function lazy decompile RPC. Sub-second per function is typical. Without `func_addr`, decompiles every function — minutes for a non-trivial binary. |
| `blocks`, `cfg_edges` | `WHERE func_addr = X` | **Yes** | Per-function CFG build. Cheap. |
| `instructions` | `WHERE address BETWEEN A AND B` | **No (post-filter scan)** | Reads the full instruction set, then SQLite filters. There is no `func_addr` column — join `funcs.address`/`funcs.end_ea` if you need a function slice, but the cache build still happens. Expect seconds for whole-program scans. |
| `xrefs` | `WHERE from_ea = X` or `to_ea = X` | **No (post-filter scan)** | Cached flat list, post-filter. Sub-second to build the cache for typical binaries. |
| `xref_index` | `WHERE src_func_addr = X` or `dst_func_addr = X` | **No (post-filter scan)** | Same shape as `xrefs`, with function-attribution columns. |
| `memory_bytes` | range | **No (post-filter scan)** | Full byte cache build. Always specify a tight range. |
| `comments` | `address = X` or range | **No (post-filter scan)** | Has an address index declared but the filter handler still does a full cache scan. **Wide-range `getComments()` can hang the host** — prefer exact-address lookups. |

## Recommendation: prefer narrow entry-point views

For function-scoped questions, the post-filter scan tables have view shortcuts that materialise the join you would otherwise pay for in-memory:

| Instead of | Use | Why |
|---|---|---|
| `instructions` joined to `funcs` | `disasm_blocks` (per-function blocks) or `disasm_calls` (per-function call sites) | Both push `func_addr` |
| `xrefs` joined to `strings` | `string_refs` (`func_addr`, `func_name`, `string_value`, `string_addr`) | Pre-attributed |
| `xrefs` joined to `funcs` | `callgraph_edges`, `callers`, `callees` | Pre-attributed |
| `xrefs` for raw edges plus function context | `xref_index` (still scan, but already carries `src_func_addr` / `dst_func_addr`) | Saves the join |

## Cache scope

Materialisation is **freshness-token scoped** for libghidra live sources. Two `/query` calls against the same active program can reuse cached table rows while `program_id`, Ghidra's native modification number, program path, and available file metadata are unchanged. If the Ghidra UI or another libghidra client edits the active program, or the active program switches, the next query invalidates before reading. Custom sources without an explicit freshness token keep the old conservative behaviour: every one-shot query invalidates first.

Inside a **batched script** (`-f script.sql`, multi-statement REPL input), writes and revision changes force invalidation before later reads. Explicit cache helpers are still useful when you want to drop a specific table before a later statement in the same batch.

| Function | Effect |
|---|---|
| `cache_stats()` | Returns JSON: cumulative `cache_invalidations_total`, `last_seen_revision`, `source_revision`, `revision_tracked`, list of cacheable tables |
| `cache_invalidate('<table>')` | Drop one table's cache so the next read pulls fresh data. Most useful in batched scripts or after source-specific uncertainty. |
| `cache_invalidate_all()` | Same as above for every table |
| `refresh_database()` | Forces source refresh and invalidates everything. Use when you want to bypass revision-gated reuse. |

Practical implication: with a normal libghidra host, one-shot `/query` calls stay fresh across external GUI/API edits because revision changes invalidate automatically. Use `refresh_database()` only when you suspect the source did not advertise a revision change.

## Worked timing example

```sql
-- Single function (cheap, pushed):
SELECT length(text) FROM pseudocode WHERE func_addr = 0x401000;
-- typical: well under a second

-- Five functions (still pushed, linear in count):
SELECT COUNT(*) FROM pseudocode
WHERE func_addr IN (SELECT address FROM funcs ORDER BY size DESC LIMIT 5);
-- typical: a few seconds

-- Whole program (anti-pattern, do not run):
-- SELECT COUNT(*) FROM pseudocode;
-- cost: per-function decompile time × number_of_funcs (minutes for thousands of functions)
```
