# ghidrasql-skills

Claude Code skills for `ghidrasql`, a live SQL interface for Ghidra analysis and annotation workflows.

## Prerequisites

- A Ghidra project with the `LibGhidraHost` extension installed and running when you need live queries
- `ghidrasql` available on your PATH or callable from a known location
- Verify: `ghidrasql --help`

## Installation

```bash
/plugin marketplace add 0xeb/ghidrasql-skills
```

## Compatibility

- `SKILL.md` is the canonical contract for each skill.
- `agents/openai.yaml` adds optional Codex app metadata and does not replace `SKILL.md`.

## Skills

| Skill | Description | When to Use |
|-------|-------------|-------------|
| `connect` | Bootstrap sessions, REPL, and HTTP usage | Starting work, checking connectivity, routing to the right skill |
| `analysis` | High-level binary triage and query workflows | Audits, summaries, "what does this binary do?" questions |
| `annotations` | Persistent renames, comments, signatures, and save workflow | Cleaning up code, adding comments, applying structured annotations |
| `decompiler` | Pseudocode, locals, parameters, and decompiler-backed workflows | Decompiling functions, local-variable cleanup, read-after-write verification |
| `disassembly` | Functions, instructions, blocks, CFG, and raw code structure | Instruction-level analysis, block queries, code-shape inspection |
| `data` | Strings, memory bytes, typed data, and hexdumps | String hunting, data-item recovery, byte inspection |
| `grep` | Name and pattern lookup across symbols and types | Finding functions, names, imports, exports, and type names |
| `xrefs` | Callers, callees, call graph, and string-reference tracing | Dependency analysis, import usage, call-chain questions |
| `types` | Type import, type exploration, and signature application | Structs, enums, typedefs, signatures, and declaration parsing |
| `functions` | SQL function catalog and function-oriented usage patterns | Looking up `ghidrasql` helper functions and choosing the right one |

## Links

- [ghidrasql](https://github.com/0xeb/ghidrasql)
- [libghidra](https://github.com/0xeb/libghidra)
- [libxsql](https://github.com/0xeb/libxsql)

## Author

**Elias Bachaalany** ([@0xeb](https://github.com/0xeb))

## License

This project and all its contents — including skill definitions, reference documentation, and configuration files — are licensed under the [Mozilla Public License 2.0](LICENSE).
