# Struct Recovery — Worked Example

A representative bottom-up struct recovery against a single shared structure.

The example below is written as if you were running it as a single batched script (`-f script.sql` or a multi-statement REPL input), so the `cache_invalidate` calls between writes and re-reads matter. If you instead issue each statement as its own `/query` call (one-shot mode), the cache is rebuilt on every query and the explicit invalidations become no-ops.

## Setup

Three functions all take the same `void *` pointer (call it `s`):

```sql
SELECT func_addr, ordinal, param_type
FROM function_params
WHERE ordinal = 0 AND param_type = 'void *'
ORDER BY func_addr;
```

Suppose `0x401100`, `0x401200`, `0x401300`.

## Step 1 — Collect deref offsets per function

```sql
SELECT printf('0x%X', func_addr) AS func,
       printf('0x%X', ea) AS site,
       expr
FROM ctree_v_derefs
WHERE func_addr IN (0x401100, 0x401200, 0x401300)
ORDER BY func_addr, ea;
```

Hand-correlate the operand expressions. Imagine the result:

```
0x401100  0x401120  [EBX]            ← s->[0]
0x401100  0x401128  [EBX+0x4]        ← s->[4]
0x401200  0x401210  [EAX+0x10]       ← s->[16]
0x401200  0x401218  [EAX+0x14]       ← s->[20]
0x401300  0x401320  [ESI+0x4]        ← s->[4]  (corroborates 0x401100)
0x401300  0x401328  [ESI+0x8]        ← s->[8]
```

Offsets observed across the three functions: 0, 4, 8, 16, 20.

## Step 2 — Inspect each access type

For each site, look at the surrounding pseudocode to infer types. Use `decomp_tokens` for the immediate context:

```sql
SELECT line, column, text, kind, var_type
FROM decomp_tokens
WHERE func_addr = 0x401100
  AND line BETWEEN
      (SELECT MAX(line) FROM decomp_tokens WHERE func_addr = 0x401100 AND text LIKE '%0x4%') - 2
  AND (SELECT MAX(line) FROM decomp_tokens WHERE func_addr = 0x401100 AND text LIKE '%0x4%') + 2
ORDER BY line, column;
```

Or pull the whole pseudocode and read the relevant block:

```sql
SELECT decompile(0x401100);
```

Suppose:

- `[s+0]` is compared against a magic constant (`0x12345678`) — `uint32_t magic`.
- `[s+4]` flows into a bitfield comparison — `uint32_t flags`.
- `[s+8]` is read with `mov al, [s+8]` and used as an index — `uint8_t kind` (next 7 bytes likely padding or related).
- `[s+16]` is dereferenced as a pointer to a string — `char *name`.
- `[s+20]` is used as a length — `uint32_t name_len`.

## Step 3 — Define the struct

```sql
SELECT parse_decls('typedef struct {
  uint32_t magic;
  uint32_t flags;
  uint8_t  kind;
  uint8_t  pad[7];
  char    *name;
  uint32_t name_len;
} Session;');

-- Verify
SELECT name, kind, size FROM types WHERE name = 'Session';
SELECT member_name, member_type, offset, size
FROM type_members
WHERE parent_type_name = 'Session'
ORDER BY offset;
```

## Step 4 — Apply to the three functions

```sql
INSERT INTO apply_type_param (func_addr, ordinal, type_name) VALUES (0x401100, 0, 'Session *');
INSERT INTO apply_type_param (func_addr, ordinal, type_name) VALUES (0x401200, 0, 'Session *');
INSERT INTO apply_type_param (func_addr, ordinal, type_name) VALUES (0x401300, 0, 'Session *');

-- Force re-decompile
SELECT cache_invalidate('pseudocode');
SELECT decompile(0x401100);
SELECT decompile(0x401200);
SELECT decompile(0x401300);
```

## Step 5 — Verify accesses now read like source

The pseudocode for `0x401100` should now show `s->magic`, `s->flags` instead of `*(int *)s` and `*(int *)(s+4)`. If a field still appears as `s[ofs]`, the struct size is wrong or a field type is mismatched. Refine and re-apply:

```sql
UPDATE type_members
SET member_type = 'uint16_t'
WHERE parent_type_name = 'Session' AND member_name = 'name_len';

SELECT cache_invalidate('pseudocode');
SELECT decompile(0x401100);
```

## Step 6 — Tag progress

```sql
INSERT OR IGNORE INTO function_tags (name, comment) VALUES ('recovered:struct:Session', '');
INSERT INTO function_tag_mappings (func_addr, tag_name)
SELECT addr, 'recovered:struct:Session'
FROM (VALUES (0x401100), (0x401200), (0x401300)) AS t(addr);
```

## Step 7 — Find more functions that take Session *

After applying, the struct propagates to functions that already had typed parameters of the same shape. Ask:

```sql
-- Who else takes a Session pointer now?
SELECT printf('0x%X', func_addr) AS func, ordinal, param_type
FROM function_params
WHERE param_type LIKE '%Session%';

-- Who passes a Session pointer to others (call-site evidence)?
SELECT printf('0x%X', src_func_addr) AS caller,
       printf('0x%X', dst_func_addr) AS callee,
       dst_func_name
FROM callgraph_edges
WHERE src_func_addr IN (0x401100, 0x401200, 0x401300)
ORDER BY callee;
```

The callees are the next iteration's seed set. Re-enter Phase 2 of the methodology with them.

## Why iterative instead of "build the perfect struct first"

Each iteration produces:

1. Better-readable pseudocode → easier to find the next field.
2. More typed parameters in callers → more functions become tractable.
3. Tag progress in `function_tags` → durable across sessions.

A struct guessed in one shot is almost always wrong on size or alignment. Pseudocode after a wrong struct apply is *worse* than pseudocode without any struct. Apply small, verify, refine.
