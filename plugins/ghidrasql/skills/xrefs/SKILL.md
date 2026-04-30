---
name: xrefs
description: "Trace callers, callees, call graph edges, and string references with ghidrasql."
allowed-tools:
  - Bash
  - Read
  - Glob
  - Grep
---

# Xrefs

## Trigger Intents

Use this skill when the user asks about:
- who calls a function
- what a function calls
- call graph structure
- which functions reference a string or import

Route to:
- `decompiler` once a specific function becomes interesting
- `analysis` for broader graph ranking
- `re-source` for recursive bottom-up annotation that consumes call-graph edges

## Performance Contract

| Surface | Predicate | Pushdown? | Notes |
|---|---|---|---|
| `xrefs` | `WHERE from_ea = X` or `to_ea = X` | **No (post-filter scan)** | Cached flat list; filter narrows result, not work |
| `xref_index` | `WHERE src_func_addr = X` or `dst_func_addr = X` | **No (post-filter scan)** | Same, with function-attribution columns. **`xref_index` is a TABLE, not a view.** |
| `callgraph_edges` | any | View over `disasm_calls` | Pre-attributed; cheap when query already has `func_addr` |
| `callers`, `callees` | `WHERE func_addr = X` | View | Pre-attributed; preferred for "who calls X" / "what does X call" |
| `string_refs` | `WHERE func_addr = X` or `string_value LIKE ...` | View | Pre-attributed function ↔ string |

For function-scoped questions, prefer the **views** (`callgraph_edges`, `callers`, `callees`, `string_refs`) over rebuilding joins on raw `xrefs` — they materialise the function attribution.

## Do This First

Confirm the graph is populated:

```sql
SELECT COUNT(*) AS edges FROM callgraph_edges;
SELECT COUNT(*) AS xref_edges FROM xref_index;
SELECT COUNT(*) AS string_hits FROM string_refs;
```

Then narrow to the target function, import, or string.

## Common Patterns

Callers of a function. The view exposes `caller_name` (the source's
function name) and `caller_func_addr` (the source function's start
address), keyed by `func_addr` (the callee):

```sql
SELECT caller_name, printf('0x%X', caller_func_addr) AS addr
FROM callers
WHERE func_addr = 0x401000
ORDER BY caller_name;
```

Callees from a function:

```sql
SELECT callee_name, printf('0x%X', callee_addr) AS addr
FROM callees
WHERE func_addr = 0x401000
ORDER BY callee_name;
```

Functions referencing a string:

```sql
SELECT func_name, printf('0x%X', func_addr) AS addr, string_value
FROM string_refs
WHERE string_value LIKE '%password%'
ORDER BY func_addr;
```

Recursive callers of a function:

```sql
WITH RECURSIVE callers_cte(func_addr, depth, path) AS (
    SELECT caller_func_addr, 1, printf('%lld', caller_func_addr)
    FROM callers
    WHERE func_addr = 0x401060

    UNION ALL

    SELECT c.caller_func_addr,
           callers_cte.depth + 1,
           callers_cte.path || '>' || printf('%lld', c.caller_func_addr)
    FROM callers c
    JOIN callers_cte ON c.func_addr = callers_cte.func_addr
    WHERE callers_cte.depth < 5
      AND instr(callers_cte.path, printf('%lld', c.caller_func_addr)) = 0
)
SELECT printf('0x%X', func_addr) AS caller,
       MIN(depth) AS distance
FROM callers_cte
GROUP BY func_addr
ORDER BY distance, caller;
```

Path between function A and B:

```sql
WITH RECURSIVE path(root_func, current_func, depth, path) AS (
    SELECT src_func_addr,
           dst_func_addr,
           1,
           printf('%lld>%lld', src_func_addr, dst_func_addr)
    FROM callgraph_edges
    WHERE src_func_addr = 0x401030

    UNION ALL

    SELECT path.root_func,
           c.dst_func_addr,
           path.depth + 1,
           path.path || '>' || printf('%lld', c.dst_func_addr)
    FROM path
    JOIN callgraph_edges c ON c.src_func_addr = path.current_func
    WHERE path.depth < 6
      AND instr(path.path, printf('%lld', c.dst_func_addr)) = 0
)
SELECT printf('0x%X', current_func) AS func_addr,
       depth,
       path
FROM path
WHERE current_func = 0x401060
ORDER BY depth;
```

Raw xref neighborhood around a function:

```sql
WITH neighborhood AS (
    SELECT *
    FROM xref_index
    WHERE src_func_addr = 0x401030

    UNION

    SELECT *
    FROM xref_index
    WHERE dst_func_addr = 0x401030
)
SELECT printf('0x%X', from_ea) AS from_ea,
       printf('0x%X', to_ea) AS to_ea,
       kind,
       printf('0x%X', src_func_addr) AS src_func_addr,
       printf('0x%X', dst_func_addr) AS dst_func_addr
FROM neighborhood
ORDER BY from_ea, to_ea;
```

## Critical Rules

- Prefer `callgraph_edges`, `callers`, `callees`, and `string_refs` over rebuilding joins by hand.
- Prefer `xref_index` when the query needs raw edges plus `src_func_addr` or `dst_func_addr`.
- Keep recursive CTEs target-specific, bound depth, and track visited nodes in the path string.
- Avoid correlated subqueries over `xrefs`/`xref_index`; aggregate once in a CTE and join back.
- Avoid repeated range joins on `funcs`; use `callgraph_edges`, `callers`, `callees`, or `xref_index` first.
- Decompiler-backed tables such as `pseudocode`, `decomp_lvars`, and `decomp_tokens` must be filtered by `func_addr` before joining graph results into them.
- Use resolved function addresses from these views to hand off cleanly to `decompiler` or `annotations`.
