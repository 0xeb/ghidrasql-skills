# Write Workflow

## Minimal Safe Sequence

```sql
-- Read first: resolve exact write coordinates
SELECT text FROM pseudocode WHERE func_addr = 0x4011F0;
SELECT local_id, name, type FROM decomp_lvars WHERE func_addr = 0x4011F0;
SELECT ordinal, param_name, param_type FROM function_params WHERE func_addr = 0x4011F0;
```

Batch the writes:

```sql
UPDATE funcs SET name = 'parseConfig' WHERE address = 0x4011F0;
UPDATE funcs SET signature = 'bool parseConfig(const char *path, Config *out)' WHERE address = 0x4011F0;
UPDATE function_params SET param_name = 'path' WHERE func_addr = 0x4011F0 AND ordinal = 0;
SELECT rename_local(0x4011F0, 'arg2', 'configFile');
INSERT INTO comments (address, comment, source) VALUES (0x4011F0, 'Parses config file.', 'plate');
```

Verify and save:

```sql
SELECT text FROM pseudocode WHERE func_addr = 0x4011F0;
SELECT save_database();
```

## Apply-type Views (INSERT-only)

Three view-shaped helpers for type application. All use INSTEAD OF INSERT triggers:

| View | Required columns | What it does |
|---|---|---|
| `apply_type_data` | `address`, `type_name` | `UPDATE data_items SET data_type = type_name WHERE address = ...` |
| `apply_type_param` | `func_addr`, `ordinal`, `type_name` | `UPDATE function_params SET param_type = type_name WHERE func_addr = ... AND ordinal = ...` |
| `apply_type_local` | `func_addr`, `local_id`, `type_name` | Calls `set_local_type(func_addr, local_id, type_name)` |

```sql
INSERT INTO apply_type_data  (address, type_name)              VALUES (0x404000, 'IMAGE_DOS_HEADER');
INSERT INTO apply_type_param (func_addr, ordinal, type_name)   VALUES (0x4011F0, 0, 'const char *');
INSERT INTO apply_type_local (func_addr, local_id, type_name)  VALUES (0x4011F0, 'arg2', 'FILE *');
```

Each `INSERT` aborts (RAISE) if a required column is missing. UPDATE/DELETE on these views is not implemented.

## Function Tags

Two-table model: `function_tags` is the catalog, `function_tag_mappings` is the join.

```sql
-- 1. Create the tag (idempotent if it already exists)
INSERT OR IGNORE INTO function_tags (name, comment) VALUES ('reviewed', 'manually reviewed');

-- 2. Map a function to the tag
INSERT INTO function_tag_mappings (func_addr, tag_name) VALUES (0x4011F0, 'reviewed');

-- 3. Find every reviewed function
SELECT f.name, printf('0x%X', f.address) AS addr
FROM funcs f
JOIN function_tag_mappings m ON m.func_addr = f.address
WHERE m.tag_name = 'reviewed';

-- 4. Delete a tag (cascades through every mapping)
DELETE FROM function_tags WHERE name = 'reviewed';
```

Suggested tag-naming convention for cross-session progress tracking (used by the `re-source` skill):

- `progress:reviewed`
- `progress:typed`
- `progress:annotated`
- `recovered:struct:Foo`
- `crypto`, `network`, `parser`, `state-machine` (analytical buckets)

## All Writable Tables Outside the Type System

- `funcs` (UPDATE name, signature)
- `names` (INSERT/UPDATE name/DELETE)
- `comments` (INSERT/UPDATE/DELETE; columns: address, comment, repeatable, source)
- `decomp_comments` (INSERT/UPDATE/DELETE; columns: func_addr, address, comment, source)
- `data_items` (INSERT/UPDATE name+data_type/DELETE)
- `bookmarks` (INSERT/UPDATE/DELETE; columns: address, type, category, comment)
- `decomp_lvars` (UPDATE name, type)
- `function_params` (UPDATE param_name, param_type)
- `signatures` (UPDATE prototype only)
- `breakpoints` (INSERT/UPDATE enabled+type+size+condition+group/DELETE)
- `function_tags`, `function_tag_mappings`
- `memory_bytes` (UPDATE value — single-byte patch; see the `debugger` skill)

Use exact row identifiers (`func_addr`, `local_id`, `ordinal`, `address`, `tag_name`) from a read query before mutating. Never guess them.

## Session End

Save first, then stop the server. There is no SQL `shutdown(...)` — set the exit policy at launch (`--shutdown save|discard|none`, default `save`) and trigger the actual stop via HTTP:

```bash
curl -X POST http://127.0.0.1:8081/shutdown
```

Wait for `java.exe` to fully exit before reusing the project directory. Force-killing leaves orphaned `*.lock` / `*.lock~` files behind.
