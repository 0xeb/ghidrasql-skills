---
name: types
description: "Import, inspect, and apply Ghidra types through ghidrasql — structs, unions, enums, typedefs, and function signatures."
allowed-tools:
  - Bash
  - Read
  - Glob
  - Grep
---

# Types

## Trigger Intents

Use this skill when the user asks to:
- create or import structs, enums, unions, typedefs
- inspect type members or enum values
- apply a function signature or parameter type
- recover or refine data types

Route to:
- `annotations` for the broader rename/comment workflow that wraps a type apply
- `decompiler` to verify the effect in pseudocode
- `re-source` for iterative cross-function struct recovery

## Writable Type Surface

| Table | UPDATE columns | INSERT (required positional) | DELETE |
|---|---|---|---|
| `types` | `name` | `name`, `kind` (size, declaration optional) | yes |
| `type_members` | `member_name`, `member_type`, `comment` | `parent_type_id`, `member_name`, `member_type` | yes |
| `type_enums` | `name` | `name` (width default 4, is_signed default 0) | yes |
| `type_enum_members` | `name`, `value`, `comment` | `type_id`, `name`, `value` | yes |
| `type_unions` | `name` | `name` (size, declaration optional) | yes |
| `type_aliases` | `name`, `target_type` | `name`, `target_type` | yes |
| `signatures` | **`prototype` only** | – | – |
| `function_params` | `param_name`, `param_type` | – | – |

**Important:** UPDATE on `kind`, `size`, `declaration`, `width`, `is_signed`, `target_type.declaration`, etc. is **not implemented** — those columns are read-only post-INSERT. To change a struct's layout or kind, delete and re-create (or use `parse_decls()`).

INSERT-only views: `apply_type_data`, `apply_type_param`, `apply_type_local` — see [references/type-workflows.md](references/type-workflows.md).

## Do This First

Inspect the existing type surface:

```sql
SELECT name, kind, size
FROM types
ORDER BY name
LIMIT 50;
```

Discover the real `kind` values on this binary (don't hard-code — values come from Ghidra):

```sql
SELECT DISTINCT kind FROM types;
```

For new declarations, `parse_decls()`:

```sql
SELECT parse_decls('typedef struct { int flags; char tag[8]; } Session;');
```

Returns the count of newly created types (0 if everything already existed) or `-1` on parse error. Standard C only — no preprocessor, no `#include`. Verify by re-reading `types`.

## type_family() classifier

`type_family(decl)` returns exactly one of these 10 strings:

```
aggregate | enum | alias | function | pointer | boolean
floating  | integral | void | unknown
```

```sql
SELECT name, type_family(declaration) AS fam
FROM types
WHERE name LIKE '%Context%';
```

Two helpers built on the same logic:

- `type_is_pointer(decl)` returns `0`/`1`.
- `type_strip_cv(decl)` removes `const`/`volatile`/`restrict`/`mutable`, preserves `*` and `&`.

## Common Patterns

Apply a function signature (writes through `funcs.signature`):

```sql
UPDATE funcs
SET signature = 'int processCommand(Command *cmd, Result *out)'
WHERE address = 0x401200;
```

If `__cdecl` (or any explicit calling-convention prefix) causes a vague `set_function_signature` error, drop it and retry. Calling-convention parsing in Ghidra can reject otherwise-valid prototypes.

Inspect struct members. The `type_members` table column is `parent_type_name` (and `parent_type_id`); the `types_members` view exposes a friendlier resolved alias `type_name` plus extra metadata (`mt_is_struct`, ...):

```sql
-- Direct table
SELECT parent_type_name, member_name, member_type, offset, size, comment
FROM type_members
WHERE parent_type_name = 'Session'
ORDER BY offset;

-- View with resolved type_name + extra metadata
SELECT type_name, member_name, member_type, offset
FROM types_members
WHERE type_name = 'Session'
ORDER BY offset;
```

Add a member. **INSERT requires `parent_type_id` (not `parent_type_name`)** — look it up from `types` first. The C++ insertable handler reads `parent_type_id` (column 0), `member_name` (column 2), `member_type` (column 3), `size` (column 5). Other columns including `offset` are ignored on INSERT — Ghidra computes offset from existing layout:

```sql
INSERT INTO type_members (parent_type_id, member_name, member_type, size)
  SELECT type_id, 'last_seen', 'time_t', 4
  FROM types WHERE name = 'Session';
```

For non-trivial layouts, use `parse_decls()` instead — it goes through Ghidra's CParser which handles offsets and alignment.

Update a member's name, type, or comment (offset and size are read-only post-INSERT):

```sql
UPDATE type_members
SET member_type = 'uint32_t',
    comment = 'flags bitfield'
WHERE parent_type_name = 'Session' AND member_name = 'flags';
```

Create an enum and add values:

```sql
INSERT INTO type_enums (name, width, is_signed) VALUES ('CmdType', 4, 0);
INSERT INTO type_enum_members (type_id, name, value)
  SELECT type_id, 'CMD_INIT', 0 FROM type_enums WHERE name = 'CmdType';
INSERT INTO type_enum_members (type_id, name, value)
  SELECT type_id, 'CMD_RUN', 1 FROM type_enums WHERE name = 'CmdType';

SELECT value_name, value, comment
FROM types_enum_values
WHERE type_name = 'CmdType'
ORDER BY value;
```

Explore signatures (read-mostly — only `prototype` is writable):

```sql
SELECT name, prototype, return_type, calling_convention, is_variadic
FROM signatures
WHERE name LIKE '%parse%';
```

## Apply-Type Views (INSERT-only shorthand)

Three views with INSTEAD OF INSERT triggers for compositional type application:

```sql
-- Apply a global data type at an address
INSERT INTO apply_type_data  (address, type_name)             VALUES (0x404000, 'IMAGE_DOS_HEADER');

-- Apply a parameter type by ordinal
INSERT INTO apply_type_param (func_addr, ordinal, type_name)  VALUES (0x401200, 0, 'Command *');

-- Apply a local-variable type by local_id (calls set_local_type underneath)
INSERT INTO apply_type_local (func_addr, local_id, type_name) VALUES (0x401200, 'arg2', 'Result *');
```

UPDATE/DELETE on these views is not implemented — they are write-paths only.

## parse_decls() Semantics

- Standard C only. No `#include`, no `#define`, no comments-as-pragmas.
- Standard fixed-width types resolve correctly: `uint8_t`, `uint16_t`, `uint32_t`, `int8_t`, `size_t`, `ptrdiff_t`. (Older builds mis-mapped `uint8_t`/`uint16_t`/`uint32_t`; current builds resolve them correctly.)
- Returns count of newly created types or `-1` on error. **Partial failures are silent** — verify by re-reading `types` after the call.
- Re-running `parse_decls()` for an already-imported declaration returns `0`, not an error. It is idempotent for the no-op case.

```sql
-- Bulk import with verification:
SELECT parse_decls('
typedef enum { CMD_INIT=0, CMD_RUN=1, CMD_STOP=2 } CmdType;
typedef struct {
  CmdType type;
  int     flags;
  char    tag[8];
} Command;
');

SELECT name, kind FROM types WHERE name IN ('Command', 'CmdType');
```

## Critical Rules

- Use standard C declarations with `parse_decls()` — preprocessor directives are not supported.
- Import declarations **before** applying signatures or member types that depend on them.
- Re-read `pseudocode` after type changes that affect decompilation. In one-shot mode, the next query already sees the updated decompilation; inside a batched script, drop the cache first: `SELECT cache_invalidate('pseudocode'); SELECT decompile(0xX);`.
- **`signatures` writes only `prototype`** — to rename a function, write `funcs.name`. Don't expect `UPDATE signatures SET name = ...` to work.

## Failure and Recovery

- **`parse_decls()` returned 0 but the type didn't appear.** Either the declaration was already imported, or the parser silently rejected it. Re-read `types`; if absent, simplify the declaration (drop forward declarations, qualifiers) and retry.
- **`apply_type_param` INSERT raises 'apply_type_param requires func_addr, ordinal, and type_name'.** All three must be non-NULL; `type_name` must be non-empty.
- **`UPDATE decomp_lvars SET type = 'wchar_t[128]'` fails.** Use the `apply_type_local` view or `set_local_type()` SQL function — the direct UPDATE path does not always re-parse array types.
- **Pointer-return signature update fails with `Can't parse name: *fn`.** Ghidra's CParser tokenises `<type> *fn(...)` as a name `*fn` instead of a return-type pointer. Workaround: write the asterisk on the type, not the name — `char* fn(...)`, `void** fn(...)`. Same trap on parameter lists; prefer `T* name` over `T *name` when authoring prototypes.
- **Applied type over-propagates to other locals.** The decompiler reuses register/stack slots across source-level variables, so applying a type at one site can affect unrelated ones rendered with the same storage. Keep recovered structs on prototypes/parameters; for individual locals, back out to a neutral type (`char *`, `int`) and verify each apply in `pseudocode`.

## Additional Resources

- Type import and application patterns: [references/type-workflows.md](references/type-workflows.md)
