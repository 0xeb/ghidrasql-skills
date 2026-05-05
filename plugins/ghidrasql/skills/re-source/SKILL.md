---
name: re-source
description: "Recursive bottom-up source recovery methodology — annotate leaf callees first, work up through the call graph, refining types iteratively."
allowed-tools:
  - Bash
  - Read
  - Glob
  - Grep
---

# Re-Source

Methodology skill, not new SQL. The pieces are already exposed by `decompiler`, `annotations`, `types`, `xrefs`, and the `function_tags` progress catalog — this skill orchestrates them into a campaign.

## Trigger Intents

Use this skill when the user asks to:
- "recover structures" / "recursively annotate" / "rebuild the source"
- understand the binary bottom-up rather than top-down
- track per-function progress across multiple sessions
- correlate struct-field accesses across many callers/callees to discover layout

Route to:
- `decompiler` for inspection (`pseudocode`, `decomp_lvars`, `ctree*`)
- `annotations` for the per-function read → write → save loop
- `types` for `parse_decls()` and `apply_type_*` views
- `xrefs` to walk the call graph

## Mental Model

A binary becomes readable when its callees are readable. Pick leaf functions first — they have no internal calls, so their behaviour is constrained by their inputs and outputs. Annotate one leaf, then move up: every annotated callee makes its callers easier to read.

Cross-function struct correlation: a single function rarely reveals a full struct layout. Function `A(s)` might only read `s->flags`; `B(s)` reads `s->name`; `C(s)` writes `s->size`. The struct's full layout is the union of every accessor — which means you have to **look at multiple callers and callees of the same function** to discover all fields. The `ctree*` views and `decomp_tokens` are how you find those access patterns.

Project scope: ghidrasql can list many programs in one Ghidra project via
`project_programs`, but every analysis table in this workflow is scoped to the
one active program. For a multi-program campaign, list targets first, then work
one domain path at a time:

```sql
SELECT path, name FROM project_programs ORDER BY path;
```

Switch programs by saving/closing and reopening a different path, or by
re-invoking `ghidrasql --program /path/in/project`. Do not mix progress tags
from different active programs unless your tag names include the program path.

## Phase 1 — Pick the Seed Set

The cheapest seed is leaf functions. ghidrasql exposes two leaf-function views:

```sql
-- Leaf via decompiler call graph (preferred — uses ctree_v_calls)
SELECT printf('0x%X', address) AS addr, name
FROM ctree_v_leaf_funcs
ORDER BY name
LIMIT 50;

-- Leaf via disassembly call graph (fallback when decompiler is slow)
SELECT printf('0x%X', address) AS addr, name
FROM disasm_v_leaf_funcs
ORDER BY name
LIMIT 50;
```

Filter to non-trivial leaves (some are short stubs):

```sql
SELECT printf('0x%X', l.address) AS addr, l.name, f.size
FROM ctree_v_leaf_funcs l
JOIN funcs f ON f.address = l.address
WHERE f.size BETWEEN 64 AND 2048
ORDER BY f.size DESC
LIMIT 30;
```

Tag the seed set so progress is visible across sessions:

```sql
INSERT OR IGNORE INTO function_tags (name, comment)
VALUES ('progress:seed', 're-source seed set, picked from ctree_v_leaf_funcs');

INSERT INTO function_tag_mappings (func_addr, tag_name)
SELECT address, 'progress:seed'
FROM ctree_v_leaf_funcs
WHERE address IN (SELECT address FROM funcs WHERE size BETWEEN 64 AND 2048)
LIMIT 30;
```

## Phase 2 — Annotate One Function

For each seed, hand off to `annotations` with the standard mutation loop:

1. `SELECT decompile(addr);` — read.
2. `SELECT local_id, name, type FROM decomp_lvars WHERE func_addr = addr;` — discover variables.
3. Apply types via `apply_type_param` / `apply_type_local` / `set_local_type()`.
4. Rename meaningful locals via `rename_local(addr, local_id, new_name)`.
5. Set the function's prototype: `UPDATE funcs SET signature = '...' WHERE address = addr;`.
6. Plate-comment the function: `INSERT INTO comments (address, comment, source) VALUES (addr, 'summary', 'plate');`.
7. Tag progress: `INSERT INTO function_tag_mappings (func_addr, tag_name) VALUES (addr, 'progress:annotated');`.
8. `SELECT save_database();`.

If the decompilation now "looks like source", drop the `seed` tag and move up.

## Phase 3 — Walk Up the Graph

Find callers of the just-annotated function, prioritise by fan-in:

```sql
-- Direct callers, ranked by how many of their callees are already annotated
WITH annotated AS (
  SELECT func_addr FROM function_tag_mappings WHERE tag_name = 'progress:annotated'
),
caller_progress AS (
  SELECT ce.src_func_addr AS caller,
         COUNT(*) AS total_callees,
         SUM(CASE WHEN ce.dst_func_addr IN (SELECT func_addr FROM annotated) THEN 1 ELSE 0 END) AS done_callees
  FROM callgraph_edges ce
  GROUP BY ce.src_func_addr
)
SELECT printf('0x%X', cp.caller) AS addr,
       f.name,
       cp.done_callees, cp.total_callees,
       round(100.0 * cp.done_callees / NULLIF(cp.total_callees, 0), 1) AS pct
FROM caller_progress cp
JOIN funcs f ON f.address = cp.caller
WHERE cp.done_callees >= cp.total_callees
  AND cp.caller NOT IN (SELECT func_addr FROM annotated)
ORDER BY cp.total_callees DESC
LIMIT 20;
```

Functions whose callees are *all* annotated are the next-best candidates — every variable and call site they reference now has a name and type.

## Phase 4 — Cross-Function Struct Recovery

Pick a struct candidate (a parameter type that recurs across many functions but is still `void *` or `undefined4 *`):

```sql
-- Pointer parameters that probably represent a shared structure
SELECT param_type, COUNT(DISTINCT func_addr) AS funcs_using
FROM function_params
WHERE param_type LIKE '%void *%' OR param_type LIKE '%undefined% *%'
GROUP BY param_type
ORDER BY funcs_using DESC
LIMIT 20;
```

For each function that takes that pointer, find the field-offset accesses in `pseudocode` / `ctree_v_derefs` / `decomp_tokens`. Combine offsets across functions to discover the full layout:

```sql
-- All deref sites in functions that take the candidate parameter type
WITH candidates AS (
  SELECT DISTINCT func_addr FROM function_params WHERE param_type = 'void *'
)
SELECT printf('0x%X', d.func_addr) AS func,
       printf('0x%X', d.ea) AS site,
       d.expr
FROM ctree_v_derefs d
JOIN candidates c ON c.func_addr = d.func_addr
ORDER BY d.func_addr, d.ea
LIMIT 100;
```

(`ctree_v_derefs` is mnemonic-derived, so it surfaces patterns like `[ebx+0x10]` — the offsets are what you correlate.)

Once you've collected offsets, build the struct via `parse_decls()`:

```sql
SELECT parse_decls('typedef struct {
  uint32_t magic;     // offset 0
  uint32_t flags;     // offset 4
  char     tag[8];    // offset 8
  uint16_t version;   // offset 16 (observed in CalleeC at 0x401234)
} Session;');
```

Apply via `apply_type_param`:

```sql
INSERT INTO apply_type_param (func_addr, ordinal, type_name)
SELECT func_addr, 0, 'Session *'
FROM function_params
WHERE param_type = 'void *' AND ordinal = 0
  AND func_addr IN (SELECT func_addr FROM function_tag_mappings WHERE tag_name = 'progress:annotated');
```

Re-decompile each affected function and verify the new field accesses look right:

```sql
-- libghidra live mode: reuses cache until the native freshness token changes, then refreshes automatically
SELECT decompile(<callee_addr>);  -- substitute each affected callee

-- inside a batched script, drop the cache first so the re-decompile sees fresh data:
--   SELECT cache_invalidate('pseudocode');
--   SELECT decompile(<callee_addr>);
```

If a field access still looks wrong, refine the struct (`UPDATE type_members SET member_type = ...` or `INSERT INTO type_members ...`) and re-decompile.

## Progress Tracking Convention

Use `function_tags` + `function_tag_mappings` (no idasql-style `netnode_kv` exists in ghidrasql). Suggested tag taxonomy:

| Tag | Meaning |
|---|---|
| `progress:seed` | In the current campaign's seed set |
| `progress:reviewed` | Function has been read and understood |
| `progress:typed` | Parameters and locals have been retyped |
| `progress:annotated` | Function is named, prototyped, plate-commented |
| `recovered:struct:<Name>` | A function contributed evidence to recovering struct `<Name>` |
| `crypto`, `network`, `parser`, `state-machine` | Analytical buckets |

Cross-session resume:

```sql
-- "What's left to annotate?"
SELECT printf('0x%X', f.address) AS addr, f.name, f.size
FROM funcs f
LEFT JOIN function_tag_mappings m
       ON m.func_addr = f.address AND m.tag_name = 'progress:annotated'
WHERE m.func_addr IS NULL
ORDER BY f.size DESC
LIMIT 30;
```

## Critical Rules

- **Save often.** `SELECT save_database();` after every function. Headless saves can occasionally not persist — losing one annotated function is recoverable, losing thirty is not.
- **One function at a time.** Concurrent decompiler reads against the same host can deadlock. Don't fork annotation work across processes.
- **Iterate types, don't perfect them up front.** A struct with 3 known fields is more useful than a guess at all 12. Re-decompile, observe new offsets, refine.
- **Callees first, callers second.** Every annotated callee makes its callers easier — never the reverse.
- **Tag progress immediately.** A tag that's already in the database is more durable than a memory of "I think I did this one yesterday".

## Additional Resources

- Struct-recovery worked example: [references/struct-recovery.md](references/struct-recovery.md)
