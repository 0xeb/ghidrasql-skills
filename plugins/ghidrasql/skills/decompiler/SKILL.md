---
name: decompiler
description: "Decompile functions with ghidrasql and work with pseudocode, locals, parameters, and ctree pattern views safely."
allowed-tools:
  - Bash
  - Read
  - Glob
  - Grep
---

# Decompiler

## Trigger Intents

Use this skill when the user asks for:
- pseudocode for a function
- local variables / parameter inspection
- decompiler-token analysis
- ctree-style AST-pattern queries (calls inside loops, leaf functions, call chains)
- decompiler-backed cleanup
- re-decompile after a mutation

Route to:
- `annotations` for persistent edits
- `types` for declaration import or type recovery
- `xrefs` for caller / callee expansion from a decompiled target

## func_addr Contract

**Decompiler-backed tables MUST be filtered by `func_addr`.**

| Table | Predicate | Without it |
|---|---|---|
| `pseudocode` | `WHERE func_addr = X` | Decompiles every function in the program |
| `decomp_lvars` | `WHERE func_addr = X` | Same — full-program decompile |
| `decomp_tokens` | `WHERE func_addr = X` | Same |
| `decomp_comments` | `WHERE func_addr = X` | Same |

Per-function decompile typically takes ~500 ms. Without the filter, the cost scales with the function count — minutes per query for thousands of functions.

The `ctree*` views also compose over per-function surfaces and are best filtered the same way.

## Do This First

Direct full-text:

```sql
SELECT decompile(0x401000);
```

Structured rows:

```sql
SELECT text FROM pseudocode WHERE func_addr = 0x401000;
SELECT local_id, name, type, storage FROM decomp_lvars WHERE func_addr = 0x401000;
SELECT text, kind, var_name, var_type FROM decomp_tokens WHERE func_addr = 0x401000;
SELECT ordinal, param_name, param_type FROM function_params WHERE func_addr = 0x401000;
```

## Cache and Refresh

`decompile()` is **1-arg only — there is no force-flag**. With a libghidra live source, table materialisation can be reused across one-shot `/query` calls while the native freshness token is unchanged. Writes through ghidrasql and external Ghidra/libghidra edits change Ghidra's modification number, while program switches change `program_id`, so the next query invalidates before reading. Custom sources without freshness tracking still invalidate on every one-shot query.

The cache_invalidate dance mostly matters **inside a batched script** (`-f script.sql`, multi-statement REPL input) or when you want to force one surface to rebuild:

```sql
-- inside a -f script:
SELECT rename_local(0x401000, 'arg0', 'configHandle');
SELECT cache_invalidate('pseudocode');   -- needed because the cache was built earlier in this batch
SELECT cache_invalidate('decomp_lvars');
SELECT decompile(0x401000);
```

`cache_invalidate('<table>')` is cheaper than `refresh_database()` when only one surface is stale. Use `refresh_database()` when you want to force a full source refresh regardless of revision.

## Concurrency Caveat

**Do not run two ghidrasql clients hitting decompiler tables on the same host in parallel.** Overlapping requests against `pseudocode`/`decomp_lvars`/`decomp_comments` can deadlock the host on a fair `ReentrantReadWriteLock`. Symptoms: queries stop returning, threads in `getComments` / `decompileFunction` are parked. Workaround: serialise decompiler-backed work, or kill `java.exe` and restart if already stuck.

## Common Patterns

Read pseudocode:

```sql
SELECT text FROM pseudocode WHERE func_addr = 0x401000;
```

Inspect locals (`local_id` is `arg0`, `arg1`, ..., `var_<N>`, `local_<N>` etc., `storage` looks like `Stack[0x4]:4` or a register name):

```sql
SELECT local_id, name, type, storage
FROM decomp_lvars
WHERE func_addr = 0x401000;
```

Inspect tokens (structured decompiler output with semantic info):

```sql
SELECT text, kind, var_name, var_type
FROM decomp_tokens
WHERE func_addr = 0x401000;
```

Token kinds: `keyword`, `variable`, `type`, `function`, `parameter`, `global`, `const`, `comment`, `default`, `error`, `special`. Filter to variable references only:

```sql
SELECT text, kind, var_name, var_type, var_storage
FROM decomp_tokens
WHERE func_addr = 0x401000 AND kind = 'variable';
```

Inspect parameters:

```sql
SELECT ordinal, param_name, param_type
FROM function_params
WHERE func_addr = 0x401000
ORDER BY ordinal;
```

## ctree Views — pattern catalog over CFG + decompiler

ghidrasql exposes 16 `ctree*` views. **They are not Ghidra's real PcodeAST.** They are a SQL composition over `call_edges`, `loops`, `blocks`, `funcs`, `instructions`, and `decomp_lvars` that simulates an AST-shape catalog (`cot_call`, `cit_for`, `cit_while`, `cit_do`, `cit_if`, `cit_return`). They are useful for **pattern queries** ("calls inside loops", "leaf functions", "call chains") but not for syntactically exact AST traversal — for that, use `decomp_tokens` (which IS from the real decompiler).

| View | Composes over | Useful for |
|---|---|---|
| `ctree` | call_edges + loops + blocks (out_degree > 1) + funcs | Master catalog used by the rest |
| `ctree_call_args` | call_edges with attributed callee | Per-call argument inspection |
| `ctree_v_calls` | ctree filtered to `cot_call` | Every call in a function with callee resolution |
| `ctree_v_loops` | ctree filtered to `cit_for`/`cit_while`/`cit_do` | Loop list per function |
| `ctree_v_ifs` | ctree filtered to `cit_if` | Branch points per function |
| `ctree_v_signed_ops` | instructions filtered by mnemonic (`imul`/`idiv`/`sar`/`sal`/`sub`/`add`) | Arithmetic-of-interest hits |
| `ctree_v_comparisons` | instructions filtered to `cmp`/`test` | Comparison hits |
| `ctree_v_assignments` | instructions filtered to `mov`/`lea` | Assignment hits |
| `ctree_v_derefs` | instructions where operand contains `[` | Pointer-dereference hits |
| `ctree_v_calls_in_loops` | ctree_v_calls JOIN loops on EA range | "What gets called inside a loop body?" |
| `ctree_v_calls_in_ifs` | ctree_v_calls JOIN if_nodes | "What gets called from a conditional branch?" |
| `ctree_v_leaf_funcs` | funcs LEFT JOIN ctree_v_calls (no callees) | Bottom-up annotation entry points |
| `ctree_v_call_chains` | RECURSIVE over ctree_v_calls | Reachability with cycle guard |
| `ctree_v_returns` | funcs (synthetic return at end_ea) | Return-point markers |
| `ctree_lvars` | decomp_lvars with derived stable index | Stable per-function variable index for cross-references |

Worked examples:

```sql
-- Every call site inside a loop in this function
SELECT printf('0x%X', ea) AS site, callee_name, loop_op, printf('0x%X', loop_id) AS loop_header
FROM ctree_v_calls_in_loops
WHERE func_addr = 0x401000;

-- Leaf functions (good seed set for bottom-up annotation)
SELECT printf('0x%X', address) AS addr, name
FROM ctree_v_leaf_funcs
ORDER BY name
LIMIT 50;

-- Call chains rooted at a function (cycle-guarded recursion)
SELECT printf('0x%X', current_func) AS current, depth, path
FROM ctree_v_call_chains
WHERE root_func = 0x401000 AND depth <= 5
ORDER BY depth;
```

## Failure and Recovery

- **Query is slow or appears stuck.** Confirm `WHERE func_addr = X` is present. Without it you triggered a full-program decompile.
- **Locals do not look writable.** Query `decomp_lvars` again and use the exact `local_id` from the result (don't synthesise `arg0`, query for it).
- **Mutation happened but `pseudocode` still looks stale.** Check `program_revision()` and `cache_stats()`. Libghidra live sources refresh after Ghidra's native modification number or the program identity changes; inside a batch or when forcing a rebuild: `SELECT cache_invalidate('pseudocode'); SELECT decompile(0xX);` and re-read.
- **Two parallel decompiler queries hung the host.** Kill `java.exe`, delete `*.lock` / `*.lock~`, restart the host, run sequentially.

## Additional Resources

- Decompiler surfaces and ctree compositions: [references/decompiler-surfaces.md](references/decompiler-surfaces.md)
