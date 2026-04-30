# Data Surfaces

## strings

| Column | Type | Notes |
|---|---|---|
| `address` | int | String start address |
| `length` | int | Length in characters (not bytes for unicode) |
| `type` | text | Ghidra string-type label — discover via `SELECT DISTINCT type FROM strings;` |
| `encoding` | text | Encoding label — discover via `SELECT DISTINCT encoding FROM strings;` |
| `content` | text | Decoded string text |

Helpers:

```sql
-- Search by content (case-insensitive LIKE)
SELECT printf('0x%X', address) AS addr, content
FROM strings
WHERE content LIKE '%CryptAcquireContext%'
LIMIT 20;

-- Strings used by a specific function (via string_refs view)
SELECT s.string_value
FROM string_refs s
WHERE s.func_addr = 0x401000;

-- Hottest strings (most-referenced)
SELECT * FROM string_hotspots ORDER BY ref_count DESC LIMIT 20;
```

## data_items

| Column | Type | Writable? | Notes |
|---|---|---|---|
| `address` | int | – (key) | |
| `name` | text | UPDATE | Symbol name at this address |
| `data_type` | text | UPDATE | Ghidra type string — discover via `SELECT DISTINCT data_type FROM data_items;` |
| `size` | int | – | Byte size of the typed cell |
| `value_repr` | text | – | Human-readable rendering (e.g. `"hello"`, `0x12345678`) |
| `segment_name` | text | – | Containing segment name |
| `is_string` | int | – | 1 if Ghidra classifies this as a string |
| `is_initialized` | int | – | Expected to reflect whether the underlying segment is initialised. Verify live before relying on it — some backends always report `1`. |

INSERT/DELETE supported. Use `apply_type_data` view for compositional INSERT-as-type-application:

```sql
INSERT INTO apply_type_data (address, type_name) VALUES (0x404000, 'IMAGE_DOS_HEADER');
```

## memory_bytes

| Column | Type | Writable? | Notes |
|---|---|---|---|
| `address` | int | – (key) | |
| `value` | int | UPDATE (single-byte patch) | |
| `segment_name` | text | – | |
| `func_addr` | int | – | Containing function (NULL outside funcs) |
| `source_kind` | text | – | Origin classification |
| `item_addr` | int | – | Containing data item start |
| `item_offset` | int | – | Offset within the containing item |
| `is_printable` | int | – | 1 if `value` is printable ASCII |
| `ascii` | text | – | Single-character string when printable |

`memory_bytes` is a **post-filter scan** — always specify a tight `address` range. For byte patching, see the `debugger` skill.

## memory_blocks

| Column | Type | Notes |
|---|---|---|
| `start_ea`, `end_ea` | int | Half-open range |
| `name` | text | Block name |
| `block_class` | text | `"INITIALIZED"`, `"OVERLAY"`, `"BIT_MAPPED"`, etc. (verify via `SELECT DISTINCT block_class`) |
| `perm` | int | Composite permission bits |
| `bitness` | int | 16/32/64 |
| `size` | int | `end_ea - start_ea` |
| `is_read`, `is_write`, `is_exec` | int | Decomposed permission bits |

## relocations

| Column | Type | Notes |
|---|---|---|
| `address` | int | Where the reloc fires |
| `target_addr` | int | Target address (may be 0 for unresolved) |
| `reloc_type` | text | Ghidra reloc type label |
| `width` | int | Bytes affected |
| `symbol_name` | text | Associated symbol if any |

The `relocation_map` view enriches this with module / segment context.

## memory_hexdump

Returns a one-row dump per requested address (block layout + bytes formatted as hex/ASCII columns):

```sql
SELECT * FROM memory_hexdump WHERE address = 0x403000;
```

Related composition views:

- `memory_layout` — high-level block layout
- `memory_byte_detail` — per-byte detail with item attribution
- `memory_byte_items` — group bytes by their containing data item
- `typed_data_items` — `data_items` filtered to typed entries

## Stripped / packed PE fallback

If `SELECT COUNT(*) FROM imports;` returns 0 (the binary is stripped/packed/unusual), Ghidra often surfaces IAT entries as `data_items` named `PTR_<API>_<addr>`. Use them as proxy import rows:

```sql
SELECT printf('0x%X', address) AS addr, name
FROM data_items
WHERE name LIKE 'PTR_%'
ORDER BY name;
```

When this fallback is needed, you'll see `imports = 0` rows but a non-trivial count of `data_items` matching `name LIKE 'PTR_%'`.
