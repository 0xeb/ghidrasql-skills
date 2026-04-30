# Type Workflows

## Import Declarations

```sql
SELECT parse_decls('
typedef enum { CMD_INIT=0, CMD_RUN=1, CMD_STOP=2 } CmdType;
typedef struct {
  CmdType type;
  int     flags;
} Command;
');
```

Returns the count of newly created types (0 if everything already existed) or `-1` on parse error. Standard C only; partial failures are silent.

## Apply via UPDATE

```sql
UPDATE funcs
SET signature = 'int processCommand(Command *cmd, Result *out)'
WHERE address = 0x401200;
```

## Apply via the apply_type_* views (INSERT-only)

| View | Required columns | Underlying write |
|---|---|---|
| `apply_type_data` | `address`, `type_name` | `UPDATE data_items SET data_type = type_name WHERE address = ...` |
| `apply_type_param` | `func_addr`, `ordinal`, `type_name` | `UPDATE function_params SET param_type = type_name WHERE func_addr = ... AND ordinal = ...` |
| `apply_type_local` | `func_addr`, `local_id`, `type_name` | Calls `set_local_type(func_addr, local_id, type_name)` |

```sql
INSERT INTO apply_type_data  (address, type_name)             VALUES (0x404000, 'IMAGE_DOS_HEADER');
INSERT INTO apply_type_param (func_addr, ordinal, type_name)  VALUES (0x401200, 0, 'Command *');
INSERT INTO apply_type_local (func_addr, local_id, type_name) VALUES (0x401200, 'arg2', 'Result *');
```

## Verify

```sql
-- Pseudocode reflects the new signature.
-- In one-shot mode, the next query already sees fresh data.
-- In a batched script, drop the cache first:
--   SELECT cache_invalidate('pseudocode');
SELECT text FROM pseudocode WHERE func_addr = 0x401200;

-- Parameters carry the new types
SELECT ordinal, param_name, param_type
FROM function_params
WHERE func_addr = 0x401200;
```

## Build a Struct from Scratch

The cleanest path is `parse_decls()` — Ghidra's CParser computes offsets and alignment for you:

```sql
SELECT parse_decls('typedef struct {
  uint32_t magic;
  uint32_t flags;
  char     tag[8];
} Session;');

SELECT member_name, member_type, offset, size
FROM type_members
WHERE parent_type_name = 'Session'
ORDER BY offset;
```

Manual two-step path if you want to assemble row-by-row. Note that `INSERT INTO type_members` requires `parent_type_id` (not `parent_type_name`) and **ignores `offset`** — layout is auto-computed from existing members:

```sql
-- 1. Create the empty type
INSERT INTO types (name, kind) VALUES ('Session', 'struct');

-- 2. Add members in order; size is honoured, offset is auto
INSERT INTO type_members (parent_type_id, member_name, member_type, size)
  SELECT type_id, 'magic', 'uint32_t', 4 FROM types WHERE name = 'Session';
INSERT INTO type_members (parent_type_id, member_name, member_type, size)
  SELECT type_id, 'flags', 'uint32_t', 4 FROM types WHERE name = 'Session';
INSERT INTO type_members (parent_type_id, member_name, member_type, size)
  SELECT type_id, 'tag', 'char[8]', 8 FROM types WHERE name = 'Session';

-- 3. Verify layout
SELECT member_name, member_type, offset, size
FROM type_members
WHERE parent_type_name = 'Session'
ORDER BY offset;
```

## Build an Enum

```sql
INSERT INTO type_enums (name, width, is_signed) VALUES ('Status', 4, 0);

-- Add values keyed by type_id resolved from the previous insert
INSERT INTO type_enum_members (type_id, name, value)
  SELECT type_id, 'STATUS_OK',     0 FROM type_enums WHERE name = 'Status';
INSERT INTO type_enum_members (type_id, name, value)
  SELECT type_id, 'STATUS_FAILED', 1 FROM type_enums WHERE name = 'Status';

SELECT value_name, value, comment
FROM types_enum_values
WHERE type_name = 'Status'
ORDER BY value;
```

## Cross-Reference Types and Functions

```sql
-- Functions whose return type or any parameter matches a pattern
SELECT f.name, printf('0x%X', f.address) AS addr, f.signature
FROM funcs f
WHERE f.signature LIKE '%Session%'
ORDER BY f.name;

-- Members with a specific type (e.g. find every struct with a Command pointer)
SELECT parent_type_name, member_name, offset
FROM type_members
WHERE member_type LIKE '%Command%';
```
