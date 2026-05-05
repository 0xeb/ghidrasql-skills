---
name: debugger
description: "Manage breakpoints and patch bytes through ghidrasql — the breakpoints table and memory_bytes single-byte UPDATE."
allowed-tools:
  - Bash
  - Read
  - Glob
  - Grep
---

# Debugger

ghidrasql exposes Ghidra's breakpoint metadata table and a single-byte patch path through `memory_bytes.value`. This is **not** an active debugger control plane — Ghidra itself is not a runtime debugger in the JTAG sense — but it covers static breakpoint definition for tooling that consumes the project (e.g. running it under a real debugger later) and direct byte patching.

## Trigger Intents

Use this skill when the user asks to:
- list, add, modify, or delete breakpoints
- patch bytes at an address
- inspect or audit existing patches
- review `bookmarks` (lightweight notes that often double as patch markers)

Route to:
- `disassembly` for context around a breakpoint or patch site
- `annotations` to document a patch via a comment or bookmark
- `analysis` for triage that uncovered the patch

## What ghidrasql Does NOT Provide

- No live debugger control. There is no `step`, no `continue`, no register read/write.
- No watchpoint. The closest analogue is a `breakpoints.type` value reflecting the kind, but ghidrasql does not exercise it.
- No multi-byte atomic patch. `memory_bytes` writes one byte at a time. For multi-byte patches, batch UPDATEs in a single transaction-like script.

## Breakpoints

`breakpoints` is a writable virtual table. 11 columns:

| Column | Type | Writable? |
|---|---|---|
| `address` | int | – (key) |
| `enabled` | int | UPDATE |
| `type` | int | UPDATE |
| `type_name` | text | – (computed from `type`) |
| `size` | int | UPDATE |
| `flags` | int | – |
| `pass_count` | int | – |
| `condition` | text | UPDATE |
| `group` | text | UPDATE |
| `loc_type` | int | – |
| `loc_type_name` | text | – (computed from `loc_type`) |

INSERT signature (positional argv):

```sql
INSERT INTO breakpoints (address, enabled, type, size, condition, group)
VALUES (0x401234, 1, 0, 1, '', 'review');
```

Required: `address`. Defaults: `enabled = 1`, `type = 0`, `size = 1`, `condition = ''`, `group = ''`.

Common queries:

```sql
-- List every breakpoint with its decoded type and location names
SELECT printf('0x%X', address) AS at, enabled, type_name, size, condition, group, loc_type_name
FROM breakpoints
ORDER BY address;

-- Disable a breakpoint without removing it
UPDATE breakpoints SET enabled = 0 WHERE address = 0x401234;

-- Add a conditional breakpoint
UPDATE breakpoints
SET condition = 'EAX == 0',
    group = 'crashes'
WHERE address = 0x401234;

-- Delete
DELETE FROM breakpoints WHERE address = 0x401234;
```

To discover the actual `type` and `loc_type` integer values used by Ghidra on this binary, consult `type_name` / `loc_type_name`:

```sql
SELECT DISTINCT type, type_name FROM breakpoints;
SELECT DISTINCT loc_type, loc_type_name FROM breakpoints;
```

Empty until at least one breakpoint exists — set one through the GUI or via INSERT first.

## Byte Patching via memory_bytes

`memory_bytes.value` is the **single writable byte column**. Each UPDATE patches one address:

```sql
-- Single-byte patch (NOP an instruction byte)
UPDATE memory_bytes SET value = 0x90 WHERE address = 0x401234;

-- Verify
SELECT printf('0x%X', address) AS at, value, ascii, is_printable
FROM memory_bytes
WHERE address = 0x401234;
```

Multi-byte patch (NOP a 5-byte instruction):

```sql
UPDATE memory_bytes SET value = 0x90 WHERE address = 0x401234;
UPDATE memory_bytes SET value = 0x90 WHERE address = 0x401235;
UPDATE memory_bytes SET value = 0x90 WHERE address = 0x401236;
UPDATE memory_bytes SET value = 0x90 WHERE address = 0x401237;
UPDATE memory_bytes SET value = 0x90 WHERE address = 0x401238;
SELECT save_database();
```

There is no separate `WriteBytes` / `PatchBytes` SQL surface — `memory_bytes.value` is the path.

## Patch Inventory

To find bytes that have been patched (i.e. differ from the original image), join `memory_bytes` against itself by `source_kind`:

```sql
-- Discover what source_kind values exist on this binary
SELECT DISTINCT source_kind FROM memory_bytes WHERE source_kind IS NOT NULL LIMIT 10;
```

Document each patch with a bookmark so the patch survives rediscovery:

```sql
INSERT INTO bookmarks (address, type, category, comment)
VALUES (0x401234, 'Note', 'patch', 'NOPed magic check at the entry to authValidate');
```

## Performance and Safety

- `memory_bytes` is a **post-filter scan**. Each UPDATE costs roughly the cost of a full byte cache build; avoid loops that patch dozens of bytes one at a time without the guard `WHERE address = X` already in place.
- Patches are persisted only when you call `SELECT save_database();` (or run with `--shutdown save`, the default in managed mode).
- For read-only audit sessions, run the host with `--readonly --shutdown discard` so accidental UPDATEs cannot persist.

## Failure and Recovery

- **`UPDATE memory_bytes` returned without error but the byte didn't change.** Re-read the same address and check `program_revision()` / `cache_stats()`. Libghidra live sources refresh automatically when Ghidra's native modification number or program identity changes; inside a batched script, drop the cache first (`SELECT cache_invalidate('memory_bytes');`) and then re-read. If the address is in a non-writable segment (e.g. `.rdata` mapped read-only), Ghidra accepts the write into its program model but a runtime debugger may refuse to apply it.
- **Breakpoint did not appear.** Check that the address is inside a known function (`SELECT * FROM funcs WHERE address <= 0xN AND end_ea > 0xN;`); breakpoints set on data are often dropped by Ghidra.
