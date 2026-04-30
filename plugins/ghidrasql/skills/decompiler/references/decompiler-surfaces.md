# Decompiler Surfaces

## Primary Reads

Full-function text:

```sql
SELECT decompile(0x401000);
```

Structured pseudocode (single row, the entire function in `text`):

```sql
SELECT text FROM pseudocode WHERE func_addr = 0x401000;
```

`pseudocode` returns at most one row per function. The `text` column contains the entire decompilation as a single string.

## Local Variables

```sql
SELECT local_id, name, type, storage
FROM decomp_lvars
WHERE func_addr = 0x401000;
```

`local_id` is a stable Ghidra identifier (`arg0`, `arg1`, ..., `var_24`, `local_8`, ...). `storage` looks like `"Stack[0x4]:4"` for stack locals or a register name (`"EAX:4"`) for register-allocated locals.

Use the exact `local_id` you just read for writes:

```sql
SELECT rename_local(0x401000, 'arg0', 'configHandle');
SELECT set_local_type(0x401000, 'arg0', 'HANDLE');

-- Or via direct UPDATE (works for name; may fail for type on array locals):
UPDATE decomp_lvars
SET name = 'configHandle'
WHERE func_addr = 0x401000 AND local_id = 'arg0';
```

## Decompiler Tokens

`decomp_tokens` is the decompiler's tokenized output — the closest thing ghidrasql exposes to a real AST. Each row is one syntactic token.

```sql
SELECT text, kind, var_name, var_type, var_storage, line, column
FROM decomp_tokens
WHERE func_addr = 0x401000
ORDER BY token_index;
```

Token `kind` values:

`keyword`, `variable`, `type`, `function`, `parameter`, `global`, `const`, `comment`, `default`, `error`, `special`.

Filter to a specific kind:

```sql
SELECT DISTINCT var_name, var_type
FROM decomp_tokens
WHERE func_addr = 0x401000 AND kind = 'variable';
```

## Parameters

```sql
SELECT ordinal, param_name, param_type, storage
FROM function_params
WHERE func_addr = 0x401000
ORDER BY ordinal;
```

Writable: `param_name`, `param_type`. Use the exact `ordinal` from the read.

## Decompiler Comments

`decomp_comments` carries comments anchored at decompiler tokens (separate from raw-address `comments`):

```sql
SELECT address, comment, source
FROM decomp_comments
WHERE func_addr = 0x401000;
```

## ctree* Views

See the main skill file for the full table. Critical reminder: `ctree*` views are **not** Ghidra's real PcodeAST — they are SQL compositions over `call_edges`, `loops`, `blocks`, `funcs`, `instructions`, and `decomp_lvars`. Useful for pattern catalogs (`ctree_v_calls_in_loops`, `ctree_v_leaf_funcs`, `ctree_v_call_chains`) but not for syntactically exact AST matching. For that, use `decomp_tokens`.

## Reminder

Decompiler-backed tables are on-demand surfaces. Keep them tightly scoped by `func_addr`. In one-shot mode, each query rebuilds the relevant tables; inside a batched script, drop the cache after a write (`SELECT cache_invalidate('pseudocode');`) before re-reading.
