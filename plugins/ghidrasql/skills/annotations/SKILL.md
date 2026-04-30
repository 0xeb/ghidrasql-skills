---
name: annotations
description: "Apply persistent ghidrasql annotations such as names, comments, signatures, and local-variable edits."
allowed-tools:
  - Bash
  - Read
  - Glob
  - Grep
---

# Annotations

## Trigger Intents

Use this skill when the user asks to:
- rename a function, symbol, parameter, or local
- add comments
- apply or adjust a function signature
- tag functions
- apply types via the `apply_type_*` views
- persist cleanup into the Ghidra database

Route to:
- `decompiler` to inspect locals and pseudocode before editing
- `types` when a mutation depends on declarations or type imports
- `re-source` when the campaign expands beyond a single function

## Mandatory Mutation Loop

1. Read the current function or target rows.
2. Resolve exact write coordinates (`func_addr`, `local_id`, `ordinal`, `address`).
3. Batch the writes.
4. Re-read the affected surfaces.
5. `SELECT save_database();` once at the end.
6. To end the session: `curl -X POST http://127.0.0.1:8081/shutdown` (managed mode applies the launch-time `--shutdown save` policy; proxy mode leaves the upstream host running).

There is no SQL `shutdown(...)` function.

## Two save-time gotchas to know about

1. **Verify saves explicitly.** `save_database()` returning `1` is not always proof the change persisted — in some configurations the save is silently dropped on reopen. For critical edits, save → shut the host down → reconnect with `--readonly --no-analyze` → re-query the affected rows and compare against the post-write values. See the connect skill's mutation-loop section for the full recipe.
2. **GUI host stalls under concurrent decompiler reads.** Two ghidrasql clients simultaneously hitting `pseudocode`/`decomp_lvars`/`decomp_comments` on the same host can deadlock on a fair `ReentrantReadWriteLock`. Serialise decompiler-backed work — do not run two parallel annotation pipelines against one host.

## Writable Surface (verified per-table)

| Table | UPDATE columns | INSERT | DELETE |
|---|---|---|---|
| `funcs` | `name`, `signature` | – | – |
| `names` | `name` | yes | yes |
| `comments` | `comment`, `repeatable`, `source` | yes | yes |
| `data_items` | `name`, `data_type` | yes | yes |
| `bookmarks` | `type`, `category`, `comment` | yes | yes |
| `decomp_lvars` | `name`, `type` | – | – |
| `decomp_comments` | `comment`, `source` | yes | yes |
| `function_params` | `param_name`, `param_type` | – | – |
| `function_tags` | `comment` | yes | yes |
| `function_tag_mappings` | – | yes | yes |
| `signatures` | **`prototype` only** (not `name`) | – | – |
| `breakpoints` | `enabled`, `type`, `size`, `condition`, `group` | yes | yes |
| `memory_bytes` | `value` (single-byte patch) | – | – |

Type tables (`types`, `type_members`, `type_enums`, `type_enum_members`, `type_unions`, `type_aliases`) support INSERT and DELETE; UPDATE is mostly limited to `name` (full per-table matrix in the `types` skill).

INSERT-only views: `apply_type_data`, `apply_type_param`, `apply_type_local` — INSTEAD OF INSERT triggers that wrap the underlying type write.

## Canonical Mutation Surfaces

Function rename and signature:

```sql
UPDATE funcs
SET name = 'parseConfig',
    signature = 'bool parseConfig(const char *path, Config *out)'
WHERE address = 0x4011F0;
```

Parameter rename or type:

```sql
UPDATE function_params
SET param_name = 'path',
    param_type = 'const char *'
WHERE func_addr = 0x4011F0 AND ordinal = 0;
```

Decompiler local rename or type — **prefer the SQL helpers** (direct RPC, no re-decompilation between statements):

```sql
SELECT rename_local(0x4011F0, '<local_id>', 'configFile');
SELECT set_local_type(0x4011F0, '<local_id>', 'FILE *');
```

Alternative via UPDATE (works for the *name*; type rewrites can fail on array locals — see Gotchas):

```sql
UPDATE decomp_lvars
SET name = 'configFile'
WHERE func_addr = 0x4011F0 AND local_id = '<local_id>';
```

Apply a type via the dedicated view (compositional — works even when the agent only knows the type name):

```sql
INSERT INTO apply_type_data  (address, type_name)               VALUES (0x404000, 'IMAGE_DOS_HEADER');
INSERT INTO apply_type_param (func_addr, ordinal, type_name)    VALUES (0x4011F0, 0, 'const char *');
INSERT INTO apply_type_local (func_addr, local_id, type_name)   VALUES (0x4011F0, 'arg2', 'FILE *');
```

Comment insertion (the `source` column controls the comment kind: `'plate'`, `'pre'`, `'post'`, `'eol'`, `'repeatable'`):

```sql
INSERT INTO comments (address, comment, source)
VALUES (0x4011F0, 'Reads and parses the configuration file.', 'plate');
```

Decompiler-attached comments (anchored at decompiler tokens, not raw addresses):

```sql
INSERT INTO decomp_comments (func_addr, address, comment, source)
VALUES (0x4011F0, 0x4012A0, 'fail path: invalid magic', 'pre');
```

Bookmark (defaults to `type='Analysis'` if not specified):

```sql
INSERT INTO bookmarks (address, type, category, comment)
VALUES (0x4011F0, 'Note', 'review', 'check overflow handling');
```

Function tags (catalog first, then map):

```sql
INSERT INTO function_tags (name, comment)         VALUES ('reviewed', 'manually reviewed');
INSERT INTO function_tag_mappings (func_addr, tag_name) VALUES (0x4011F0, 'reviewed');
```

Tag deletion cascades through both tables: `DELETE FROM function_tags WHERE name = 'reviewed';` removes every mapping.

## Gotchas

- **`signatures` is read-mostly.** Only `prototype` is writable. To rename the function itself, `UPDATE funcs SET name = ...`. To change the calling convention, write a full prototype string into either `funcs.signature` or `signatures.prototype`.
- **`__cdecl` in a prototype string can fail with a vague diagnostic.** If `set_function_signature at 0xN` errors without a parse detail, drop the calling-convention prefix and retry — the signature without `__cdecl` typically applies cleanly.
- **Renaming an array local via `UPDATE decomp_lvars SET name = ...` may fail with "unable to resolve writable data type 'wchar_t[128]'".** The UPDATE path re-applies the current type during the rename, and Ghidra refuses arrays in some contexts. Fall back to:
  ```sql
  SELECT rename_local(0x4011F0, '<local_id>', 'configFile');
  ```
  which goes through a direct RPC and ignores the type.
- **Unsigned C type aliases (`uint8_t`/`uint16_t`/`uint32_t`/`unsigned`/`unsigned int`)** are mis-mapped in older builds of `parse_decls()`. Current builds resolve them correctly; if you hit a width mismatch, suspect an outdated build.
- **Caches can hold stale rows inside a batched script.** Materialisation is query-scoped in normal one-shot mode (each `/query` rebuilds the tables it touches), so a fresh read after a write already reflects the write. Inside a `-f` script or multi-statement REPL input, however, the cache persists across statements — drop just the table you wrote to before reading it again:
  ```sql
  SELECT cache_invalidate('decomp_lvars');
  ```
  Cheaper than `refresh_database()` when only one surface is stale.
- **Pointer-return signature parser is whitespace-sensitive.** Ghidra's CParser tokenises `<type> *fn(...)` as a name `*fn` rather than a return type. Use `<type>* fn(...)` form when authoring prototypes — `char* foo`, `void** bar`, `int* baz`. Same trap applies to multi-statement parameter lists. Verify the apply landed by re-reading `funcs.signature` or `function_params`.
- **Type apply can over-propagate through reused decompiler temporaries.** Ghidra shares storage between source-level variables that happen to use the same register/stack slot. Applying a type via `apply_type_local` (or a direct UPDATE on `decomp_lvars`) at one site can change unrelated variables. Mitigation: keep the recovered type on prototypes/parameters, back the local out to a neutral type (`char *`, `int`) at the questionable site, and verify each step in `pseudocode` before continuing.

## Verification Pattern

```sql
SELECT text FROM pseudocode WHERE func_addr = 0x4011F0;
SELECT local_id, name, type FROM decomp_lvars WHERE func_addr = 0x4011F0;
SELECT ordinal, param_name, param_type FROM function_params WHERE func_addr = 0x4011F0;
SELECT save_database();
```

## Failure and Recovery

- **Mutation did not show up:** query the exact target table again, re-read `pseudocode`, verify you used the canonical `local_id` / `ordinal` / `address`. If you're inside a batched script and the row is correct in `decomp_lvars` but `pseudocode` looks stale, `SELECT cache_invalidate('pseudocode'); SELECT decompile(0xX);`.
- **Multiple changes needed:** put them in one script and `save_database()` once at the end.
- **Save reported success but the change is gone after reopen:** the save was silently dropped. Re-apply the mutation, run the save → shutdown → reconnect verification recipe again, and confirm the post-reopen value matches before continuing.

## Additional Resources

- Write workflow and supported surfaces: [references/write-workflow.md](references/write-workflow.md)
