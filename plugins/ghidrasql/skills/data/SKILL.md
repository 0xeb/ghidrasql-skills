---
name: data
description: "Query strings, bytes, data items, memory blocks, and relocations through ghidrasql."
allowed-tools:
  - Bash
  - Read
  - Glob
  - Grep
---

# Data

## Trigger Intents

Use this skill when the user asks to:
- search strings
- inspect bytes or hexdumps
- explore data items and typed globals
- inspect memory blocks or relocations
- trace data-oriented evidence in a binary

Route to:
- `xrefs` when string or data references matter (use `string_refs` view)
- `analysis` for higher-level suspiciousness or summarization
- `annotations` if the next step is naming or typing the data
- `debugger` for byte patches via `UPDATE memory_bytes`

## Performance Contract

| Surface | Predicate | Pushdown? | Notes |
|---|---|---|---|
| `strings` | any | Indexed | Cheap |
| `data_items` | any | Indexed | Cheap (sub-second for tens of thousands of rows) |
| `memory_bytes` | range | **No (post-filter scan)** | Always tightly bound |
| `memory_blocks`, `segments` | any | Indexed | Cheap, small cardinality |
| `relocations` | any | Indexed | Cheap |

Composition views: `memory_layout`, `memory_hexdump`, `memory_byte_detail`, `memory_byte_items`, `typed_data_items`, `relocation_map`, `string_hotspots`, `string_refs`.

## Do This First

Lightest useful surface — strings:

```sql
SELECT printf('0x%X', address) AS addr, length, type, encoding, content
FROM strings
WHERE content LIKE '%password%'
ORDER BY address
LIMIT 50;
```

`type` and `encoding` come from Ghidra. Run `SELECT DISTINCT type, encoding FROM strings;` to enumerate what's present on this binary.

## Common Patterns

Typed globals (only data items that have a Ghidra-recognised type):

```sql
SELECT printf('0x%X', address) AS addr, name, data_type, size
FROM data_items
WHERE data_type IS NOT NULL AND data_type != ''
ORDER BY address
LIMIT 50;
```

Discover the actual `data_type` values on this binary (don't hard-code — the set varies by binary, processor, and Ghidra version):

```sql
SELECT DISTINCT data_type
FROM data_items
WHERE data_type IS NOT NULL AND data_type != ''
LIMIT 50;
```

Typical results include Ghidra primitives (`byte`, `word`, `dword`, `undefined1/2/4/8`, `pointer`, `string`, `unicode`, `TerminatedCString`, `TerminatedUnicode`). Format-specific structures (PE, ELF, Mach-O, etc.) appear when Ghidra recognises the container — names depend on the binary in front of you. Don't hard-code — enumerate live.

Hexdump for a specific address:

```sql
SELECT *
FROM memory_hexdump
WHERE address = 0x403000;
```

Memory blocks (per-segment metadata; perm flags as `is_read`, `is_write`, `is_exec`):

```sql
SELECT printf('0x%X', start_ea) AS start,
       printf('0x%X', end_ea) AS end,
       name, block_class, size,
       is_read, is_write, is_exec
FROM memory_blocks
ORDER BY start_ea;
```

Relocations (note: table is named `relocations`, not `relocation_items`):

```sql
SELECT printf('0x%X', address) AS at,
       printf('0x%X', target_addr) AS target,
       reloc_type, width, symbol_name
FROM relocations
ORDER BY address;
```

Create a typed data item:

```sql
INSERT INTO data_items (address, data_type)
VALUES (0x403000, 'dword');
```

## String References

Functions that reference a string — use the `string_refs` view, which pre-attributes the function context:

```sql
SELECT printf('0x%X', func_addr) AS func, func_name, string_value
FROM string_refs
WHERE string_value LIKE '%error%'
ORDER BY func_addr
LIMIT 50;
```

For "which strings are referenced most", use `string_hotspots`.

## When `imports` is Empty

Some PE binaries (packed, stripped, or unusually crafted) parse with **`imports` empty** but their IAT entries surface as `data_items` named `PTR_<API>_<address>`.

```sql
SELECT printf('0x%X', address) AS addr, name
FROM data_items
WHERE name LIKE 'PTR_%'
ORDER BY name
LIMIT 50;
```

If this fallback returns rows, route the agent to think of those as imports for fan-in / call-target analysis.

## Critical Rules

- Keep `memory_bytes` queries tightly bounded — it's a post-filter scan over the whole address space.
- Prefer `strings`, `data_items`, `memory_hexdump`, `memory_blocks`, `relocations` over raw byte scans.
- When a string matters semantically, hand off to `string_refs` (read it as a `data` query, then route to `xrefs` if you need callers).

## Additional Resources

- Memory and data surfaces: [references/data-surfaces.md](references/data-surfaces.md)
