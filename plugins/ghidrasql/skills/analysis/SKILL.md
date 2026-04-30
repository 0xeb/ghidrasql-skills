---
name: analysis
description: "Analyze binaries with ghidrasql using safe, high-signal query patterns."
allowed-tools:
  - Bash
  - Read
  - Glob
  - Grep
---

# Analysis

## Trigger Intents

Use this skill when the user asks:
- what a binary or component appears to do
- which functions are largest, hottest, or most connected
- which imports, strings, or tags stand out
- for a quick triage or suspiciousness pass

Route to:
- `xrefs` for deeper caller/callee tracing
- `decompiler` for concrete function interpretation
- `annotations` once a function is understood well enough to clean up
- `debugger` when triage uncovers patches or breakpoints worth inspecting
- `re-source` for a structured bottom-up campaign over the binary

## Do This First

Get a quick structural snapshot:

```sql
SELECT * FROM db_info;

SELECT COUNT(*) AS funcs, SUM(size) AS total_bytes
FROM funcs;
```

Then use bounded, high-signal summaries:

```sql
SELECT name, printf('0x%X', address) AS addr, size
FROM funcs
ORDER BY size DESC
LIMIT 20;
```

```sql
SELECT func_name, printf('0x%X', func_addr) AS addr, hotness_score
FROM function_metrics_scored
ORDER BY hotness_score DESC
LIMIT 20;

-- Or the rank-only projection:
SELECT func_name, printf('0x%X', func_addr) AS addr, rank, score
FROM function_metrics_ranked
ORDER BY rank
LIMIT 20;
```

## Practical Query Patterns

Most called functions:

```sql
SELECT dst_func_name, printf('0x%X', dst_func_addr) AS addr, COUNT(*) AS caller_count
FROM callgraph_edges
GROUP BY dst_func_addr, dst_func_name
ORDER BY caller_count DESC
LIMIT 20;
```

String-heavy functions:

```sql
SELECT func_name, printf('0x%X', func_addr) AS addr, COUNT(*) AS string_count
FROM string_refs
GROUP BY func_addr, func_name
ORDER BY string_count DESC
LIMIT 20;
```

Functions calling a suspicious import:

```sql
SELECT DISTINCT src_func_name, printf('0x%X', src_func_addr) AS addr
FROM callgraph_edges
WHERE dst_func_name LIKE '%Crypt%'
ORDER BY src_func_name;
```

## Critical Rules

- Prefer `callgraph_edges`, `callers`, `callees`, and `string_refs` over rebuilding those joins yourself.
- Start broad with summary views, then narrow down to specific functions.
- Once a function looks important, hand off to `decompiler` or `annotations`.

## Failure and Recovery

- If a query fans out too widely:
  - add `LIMIT`
  - switch to a summary view instead of a raw table
- If the next step is "what does this function really do?":
  - stop broad triage and move to `decompiler`

## Additional Resources

- Query recipes: [references/query-patterns.md](references/query-patterns.md)
