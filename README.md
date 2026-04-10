# Agent-ToolKit

> **Version:** 1.0.0 | **Status:** Active | **Updated:** 2026-04-10

Universal rules, coding standards, and domain knowledge for AI coding agents. Works across any tool that reads markdown instructions.

---

| #   | Section                                    |
| --- | ------------------------------------------ |
| 1   | [Structure](#structure)                    |
| 2   | [Design Principles](#design-principles)    |
| 3   | [Loading Strategy](#loading-strategy)      |
| 4   | [Usage](#usage)                            |
| 5   | [Adding Content](#adding-content)          |
| 6   | [License](#license)                        |

---

## Structure

```text
Agent-ToolKit/
├── rules.md                 <- Universal behavior rules (always load)
├── standards/               <- Language-specific coding standards (load per language)
│   ├── rust.md
│   ├── typescript.md
│   ├── python.md
│   └── javascript.md
├── knowledge/               <- Domain-specific reference material (load on demand)
│   └── oxc.md
├── agents/                  <- Agent definitions (tools with agent support)
│   └── document-writer.md
├── README.md
└── LICENSE
```

---

## Design Principles

| Principle                      | Detail                                                             |
| ------------------------------ | ------------------------------------------------------------------ |
| Rules-dense, not example-dense | AI follows short rules. Long tutorials get ignored after line 200. |
| Load only what the task needs  | Rules always. Standards per-language. Knowledge per-domain.        |
| Universal across tools         | Plain markdown. No tool-specific format inside content files.      |
| Strict, imperative, no filler  | Every line earns its place. No prose that restates a table.        |

---

## Loading Strategy

| Layer     | Files                  | When to load                       | Token cost      |
| --------- | ---------------------- | ---------------------------------- | --------------- |
| Rules     | `rules.md`             | Every session, every tool          | ~500 tokens     |
| Standards | `standards/{lang}.md`  | When writing code in that language | ~1K tokens each |
| Knowledge | `knowledge/{domain}.md` | When working in that domain       | Varies (large)  |
| Agents    | `agents/*.md`          | Tools with agent support, per-capability | ~3K tokens each |

---

## Usage

Copy `rules.md` into your tool's instruction file. Append the relevant language standard. Copy agent files if the tool supports custom agents.

| Tool    | Instruction file                       | Agent directory      |
| ------- | -------------------------------------- | -------------------- |
| Claude Code | `CLAUDE.md` or `~/.claude/CLAUDE.md` | `.claude/agents/` |
| Cursor  | `.cursor/rules/*.mdc`                  | N/A                  |
| Codex   | `AGENTS.md`                            | N/A                  |
| Windsurf | `.windsurfrules`                      | N/A                  |
| Cline   | `.clinerules`                          | N/A                  |

---

## Adding Content

### New Language Standard

Create `standards/{language}.md`. Follow the structure of existing standards: version header, quick reference table, rules as tables, minimal code examples, forbidden patterns, pre-commit checklist.

### New Domain Knowledge

Create `knowledge/{domain}.md`. Add an Invariants or Critical Rules section at the top for rules AI must always follow. Organize by dependency layer when the domain has a layered architecture.

### New Agent

Create `agents/{name}.md`. Follow the document-writer format: version header, role definition, behavior tables, checklist. Only universal agents belong here. Project-specific agents belong in their project.

---

## License

MIT License - see [LICENSE](LICENSE)
