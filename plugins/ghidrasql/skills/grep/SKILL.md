---
name: grep
description: "Search ghidrasql entities by name or pattern across functions, symbols, imports, exports, types, and strings."
allowed-tools:
  - Bash
  - Read
  - Glob
  - Grep
---

# Grep

ghidrasql does not have a `grep()` SQL function or a `grep` table — this skill is purely about applying SQLite `LIKE` patterns against the right entity table. The skill name reflects intent, not a special surface.

## Trigger Intents

Use this skill when the user asks to:
- find functions by name fragment
- locate a symbol, import, export, or type by pattern
- search candidates before deeper analysis

Route to:
- `xrefs` once a target symbol is found
- `decompiler` when the next step is reading a function
- `types` when the target is a type or member name
- `data` when searching strings (`strings.content LIKE ...`)

## LIKE Semantics

SQLite's `LIKE` is **case-insensitive by default** (`PRAGMA case_sensitive_like` defaults to off). `%` matches any substring, `_` matches a single character. For literal `%` or `_`, escape with `ESCAPE '\\'`:

```sql
SELECT name FROM funcs WHERE name LIKE '%a\_b%' ESCAPE '\\';
```

For regex-shape matching, SQLite has `REGEXP` only if a regex extension is loaded — ghidrasql does not load one. Use `LIKE` patterns or fall back to substring tests with `INSTR(...)`.

## Do This First

Function name search:

```sql
SELECT name, printf('0x%X', address) AS addr
FROM funcs
WHERE name LIKE '%config%'
ORDER BY name
LIMIT 50;
```

Symbol search (`namespace` carries the Ghidra namespace path — `Global`, `kernel32.dll`, etc.):

```sql
SELECT printf('0x%X', address) AS addr, name, namespace, symbol_kind
FROM names
WHERE name LIKE '%socket%'
ORDER BY name
LIMIT 50;
```

## Common Patterns

Imports:

```sql
SELECT printf('0x%X', address) AS addr, name, module
FROM imports
WHERE name LIKE '%Crypt%'
ORDER BY name;
```

**If `SELECT COUNT(*) FROM imports;` returns 0** (some PE binaries — packed, stripped, unusual — leave `imports` empty and surface IAT entries as `data_items`):

```sql
SELECT printf('0x%X', address) AS addr, name
FROM data_items
WHERE name LIKE 'PTR_%Crypt%'
ORDER BY name;
```

Use this fallback whenever `imports` is empty but the binary clearly imports OS APIs.

Exports:

```sql
SELECT printf('0x%X', address) AS addr, name
FROM exports
WHERE name LIKE '%Init%'
ORDER BY name;
```

Type names:

```sql
SELECT name, kind
FROM types
WHERE name LIKE '%Context%'
ORDER BY name;
```

Type members across all parents:

```sql
SELECT parent_type_name, member_name, member_type
FROM type_members
WHERE member_name LIKE '%handle%'
ORDER BY parent_type_name, member_name;
```

Strings (route to `data` skill for the full surface):

```sql
SELECT printf('0x%X', address) AS addr, content
FROM strings
WHERE content LIKE '%license%'
LIMIT 30;
```

## Multi-Surface Search

For "find every entity matching X across functions, symbols, types, and strings", the `jump_entities` view is a unified entity catalog you can pattern-match against:

```sql
SELECT entity_kind, name, address
FROM jump_entities
WHERE name LIKE '%Crypt%'
ORDER BY entity_kind, name
LIMIT 50;
```

For full-text relevance ranking, use the `search_*` SQL functions (see the `functions` skill):

```sql
SELECT name, search_score(name, 'crypt context') AS score
FROM funcs
WHERE search_match(name, 'crypt context') = 1
ORDER BY score DESC
LIMIT 20;
```

## Critical Rules

- Use this skill to narrow the target set quickly. Once you have a real address or name, hand off — don't keep searching.
- **There is no `grep()` function and no `grep` table.** Anything that *looked* like one in older docs is just a `WHERE name LIKE '...'` query; use the catalogs above.
- For substring searches that need to be case-sensitive, run `PRAGMA case_sensitive_like = 1;` first (per session).
