---
name: disassembly
description: "Inspect code structure through ghidrasql tables for functions, instructions, blocks, CFG, loops, switches, dominators, and tail calls."
allowed-tools:
  - Bash
  - Read
  - Glob
  - Grep
---

# Disassembly

## Trigger Intents

Use this skill when the user asks about:
- instructions, operands, raw code layout
- function boundaries and sizes
- blocks and CFG edges
- loops, switch tables, dominators, tail calls, function chunks
- control flow without needing pseudocode first

Route to:
- `decompiler` when higher-level reconstruction is needed
- `xrefs` for cross-function tracing
- `analysis` for ranking and triage

## Performance Contract

| Surface | Predicate | Pushdown? | Note |
|---|---|---|---|
| `funcs` | any | Indexed by `address` | Cheap |
| `blocks` | `WHERE func_addr = X` | **Yes** | Per-function CFG build |
| `cfg_edges` | `WHERE func_addr = X` | **Yes** | Per-function CFG build |
| `instructions` | `WHERE address BETWEEN A AND B` | **No (post-filter scan)** | **Has no `func_addr` column.** Cache build over every instruction in the program. Filter narrows the result, not the work. |
| `loops`, `switch_tables`, `dominators`, `post_dominators`, `tail_calls`, `function_chunks` | `WHERE func_addr = X` | Per-table | Most are per-function — verify via `PRAGMA table_xinfo` |

For function-scoped questions where you'd otherwise join `instructions` against `funcs`, prefer the **`disasm_blocks`** and **`disasm_calls`** views — they materialise the same join and push `func_addr`.

## Do This First

Pick the function and inspect its shape:

```sql
SELECT name, printf('0x%X', address) AS addr, size
FROM funcs
ORDER BY size DESC
LIMIT 10;
```

Per-function block layout (cheap, pushed):

```sql
SELECT printf('0x%X', start_ea) AS start, printf('0x%X', end_ea) AS end, in_degree, out_degree
FROM blocks
WHERE func_addr = 0x401000
ORDER BY start_ea;
```

Per-function call sites (cheap, pushed via the view):

```sql
SELECT printf('0x%X', ea) AS site, callee_name, printf('0x%X', callee_addr) AS callee
FROM disasm_calls
WHERE func_addr = 0x401000
ORDER BY ea;
```

Instruction spot-check by tight range (post-filter scan — keep the range narrow):

```sql
SELECT printf('0x%X', address) AS addr, mnemonic, operands
FROM instructions
WHERE address BETWEEN 0x401020 AND 0x401060
ORDER BY address;
```

For a function-scoped instruction slice, join against `funcs` (the scan still happens, but the result is tight):

```sql
SELECT printf('0x%X', i.address) AS addr, i.mnemonic, i.operands
FROM funcs f
JOIN instructions i ON i.address >= f.address AND i.address < f.end_ea
WHERE f.address = 0x401000
ORDER BY i.address;
```

## Critical Rules

- `instructions` has **no `func_addr` column** — confirm with `PRAGMA table_xinfo('instructions');`. Always tightly bound by `address` range.
- `blocks` and `cfg_edges` *do* take `func_addr` and push it down — use them for control-flow questions.
- For "what does this function call?" prefer `disasm_calls` over scanning `instructions` for `CALL` opcodes — `disasm_calls` resolves callee names from `funcs`/`names`.
- Hand off to `decompiler` once instruction-level evidence is enough.

## CFG and Loop Surfaces

```sql
-- Edges (kind values come from Ghidra: CONDITIONAL_JUMP, FALL_THROUGH, CALL, UNCONDITIONAL_JUMP, ...)
SELECT printf('0x%X', src_start_ea) AS src,
       printf('0x%X', dst_start_ea) AS dst,
       edge_kind
FROM cfg_edges
WHERE func_addr = 0x401000
ORDER BY src_start_ea, dst_start_ea;

-- Discover the live edge_kind enum on this binary (don't hard-code — values come from libghidra)
SELECT DISTINCT edge_kind FROM cfg_edges WHERE func_addr = 0x401000;

-- Loops: header_ea, latch_ea, range, depth, kind, block_count
SELECT printf('0x%X', header_ea) AS header, loop_kind, depth, block_count
FROM loops
WHERE func_addr = 0x401000;

-- Switch tables (jump tables recovered by the decompiler)
SELECT printf('0x%X', instr_ea) AS site,
       printf('0x%X', table_ea) AS table,
       min_case, max_case, case_count,
       printf('0x%X', default_ea) AS default_target
FROM switch_tables
WHERE func_addr = 0x401000;
```

## Dominator Surfaces

```sql
-- Per-function dominator tree (depth-tagged)
SELECT printf('0x%X', node_ea) AS node,
       printf('0x%X', idom_ea) AS idom,
       depth, is_entry
FROM dominators
WHERE func_addr = 0x401000
ORDER BY depth, node_ea;

-- Post-dominators (mirrored shape)
SELECT printf('0x%X', node_ea) AS node,
       printf('0x%X', ipdom_ea) AS ipdom,
       depth, is_exit
FROM post_dominators
WHERE func_addr = 0x401000;
```

## Tail Calls and Function Chunks

```sql
-- Tail calls: jumps treated as calls because they leave the function
SELECT printf('0x%X', call_site) AS site,
       printf('0x%X', dst_addr) AS dst,
       printf('0x%X', dst_func_addr) AS dst_func,
       tail_kind
FROM tail_calls
WHERE src_func_addr = 0x401000;

-- Function chunks: discontiguous body fragments belonging to one function
SELECT chunk_id,
       printf('0x%X', start_ea) AS start,
       printf('0x%X', end_ea) AS end,
       chunk_kind, is_primary
FROM function_chunks
WHERE func_addr = 0x401000
ORDER BY start_ea;
```

## Useful Views

| View | What it adds |
|---|---|
| `disasm_calls` | Every call site with attributed callee (`func_addr`, `ea`, `callee_addr`, `callee_name`, `kind`) — prefer over `instructions` scans |
| `disasm_blocks` | Block layout with computed `size` |
| `disasm_v_leaf_funcs` | Functions that make no calls (mnemonic-derived; complementary to `ctree_v_leaf_funcs`) |
| `disasm_v_call_chains` | Recursive call-chain projection from `disasm_calls` |
| `cfg_edges_detailed` | `cfg_edges` enriched with attributes |
| `loop_summary` | One row per loop with kind/depth/block_count |
| `switch_summary` | One row per switch with case range |
| `dominator_tree`, `post_dominator_tree` | Hierarchical dominator projections |
| `function_chunks_detailed` | Chunks with computed metadata |
| `function_frame_layout`, `stack_var_layout`, `register_var_summary` | Frame/stack/register layouts |

## Failure and Recovery

- **Query takes very long.** You probably scanned `instructions` without a tight `address` range. Switch to `disasm_blocks` or `disasm_calls` (per-function) or narrow the range to one function.
- **`edge_kind` value is unfamiliar.** It comes verbatim from libghidra's `RefType` taxonomy (`CONDITIONAL_JUMP`, `FALL_THROUGH`, `CALL`, `UNCONDITIONAL_JUMP`, `COMPUTED_JUMP`, `INDIRECTION`, etc.). Run `SELECT DISTINCT edge_kind FROM cfg_edges WHERE func_addr = X` to enumerate what's present.
- **`switch_tables` is empty.** Either the function has no switch, or the decompiler didn't recover one. Cross-check via `decomp_tokens` for `case` keywords.
