# Query Patterns

## Orphan Functions

```sql
SELECT f.name, printf('0x%X', f.address) AS addr
FROM funcs f
LEFT JOIN callers c ON c.func_addr = f.address
WHERE c.func_addr IS NULL
ORDER BY f.size DESC
LIMIT 20;
```

## Bridge Functions

```sql
WITH caller_counts AS (
  SELECT func_addr, COUNT(*) AS caller_cnt
  FROM callers
  GROUP BY func_addr
),
callee_counts AS (
  SELECT func_addr, COUNT(DISTINCT callee_addr) AS callee_cnt
  FROM callees
  GROUP BY func_addr
)
SELECT f.name,
       printf('0x%X', f.address) AS addr,
       COALESCE(cr.caller_cnt, 0) AS caller_cnt,
       COALESCE(ce.callee_cnt, 0) AS callee_cnt
FROM funcs f
LEFT JOIN caller_counts cr ON cr.func_addr = f.address
LEFT JOIN callee_counts ce ON ce.func_addr = f.address
ORDER BY caller_cnt + callee_cnt DESC
LIMIT 20;
```

## Imports by Fan-In

```sql
SELECT dst_func_name AS import_name, COUNT(DISTINCT src_func_addr) AS callers
FROM callgraph_edges
WHERE dst_func_addr IN (SELECT address FROM imports)
GROUP BY dst_func_name
ORDER BY callers DESC
LIMIT 20;
```

If `imports` is empty (some PE binaries — packed, stripped, or unusually
crafted — Ghidra exposes IAT slots as `data_items` named `PTR_<API>_<addr>`),
fan-in over those instead:

```sql
SELECT d.name AS import_name, COUNT(DISTINCT ce.src_func_addr) AS callers
FROM data_items d
JOIN callgraph_edges ce ON ce.dst_func_addr = d.address
WHERE d.name LIKE 'PTR_%'
GROUP BY d.name
ORDER BY callers DESC
LIMIT 20;
```

## Recursive Callers

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
SELECT printf('0x%X', func_addr) AS func_addr,
       MIN(depth) AS distance
FROM callers_cte
GROUP BY func_addr
ORDER BY distance, func_addr;
```

## Call Hierarchy

```sql
WITH RECURSIVE call_hierarchy(func_addr, depth, path) AS (
  SELECT 0x401030,
         0,
         printf('%lld', 0x401030)

  UNION ALL

  SELECT c.dst_func_addr,
         ch.depth + 1,
         ch.path || '>' || printf('%lld', c.dst_func_addr)
  FROM call_hierarchy ch
  JOIN callgraph_edges c ON c.src_func_addr = ch.func_addr
  WHERE ch.depth < 5
    AND instr(ch.path, printf('%lld', c.dst_func_addr)) = 0
)
SELECT printf('0x%X', h.func_addr) AS func_addr,
       COALESCE(f.name, printf('sub_%X', h.func_addr)) AS func_name,
       h.depth
FROM call_hierarchy h
LEFT JOIN funcs f ON f.address = h.func_addr
ORDER BY h.depth, h.func_addr;
```

## Path Between Functions

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

## Xref Neighborhood

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

## String and Import Traversal

```sql
SELECT sr.func_name,
       printf('0x%X', sr.func_addr) AS func_addr,
       sr.string_value,
       ce.dst_func_name AS downstream_callee
FROM string_refs sr
LEFT JOIN callgraph_edges ce ON ce.src_func_addr = sr.func_addr
WHERE sr.string_value LIKE '%error%'
ORDER BY sr.func_addr, downstream_callee
LIMIT 30;
```

## Performance Notes

- Prefer `callgraph_edges`, `callers`, `callees`, and `xref_index` over manual range joins on `funcs`.
- Keep recursive CTEs bounded and carry a visited-path guard to avoid graph explosions.
- Use `xref_index` when the query needs `src_func_addr` or `dst_func_addr`; it is the pre-attributed xref surface.
- Aggregate in a CTE and join once instead of using correlated `COUNT(*)` subqueries per function.
- Filter `pseudocode`, `decomp_lvars`, and `decomp_tokens` by `func_addr` before joining graph results into decompiler-heavy queries.
