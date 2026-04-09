# Agent-ToolKit

> **Version:** 1.0.0 | **Status:** Active | **Updated:** 2026-04-10

Universal rules, coding standards, and domain knowledge for AI coding agents. Works across Claude Code, Cursor, Codex, and any tool that reads markdown instructions.

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
│   └── oxc/
│       ├── reference.md
│       └── handbook.md
├── agents/                  <- Claude Code agent definitions (tool-specific)
│   └── document-writer.md
├── README.md
└── LICENSE
```

---

## Design Principles

| Principle | Detail |
| --------- | ------ |
| Rules-dense, not example-dense | AI follows short rules. Long tutorials get ignored after line 200. |
| Load only what the task needs | Rules always. Standards per-language. Knowledge per-domain. |
| Universal across tools | Plain markdown. No tool-specific format inside content files. |
| Strict, imperative, no filler | Every line earns its place. No prose that restates a table. |

---

## Loading Strategy

| Layer | Files | When to load | Token cost |
| ----- | ----- | ------------ | ---------- |
| Rules | `rules.md` | Every session, every tool | ~500 tokens |
| Standards | `standards/{lang}.md` | When writing code in that language | ~1K tokens each |
| Knowledge | `knowledge/{domain}/*` | When working in that domain | Varies (large) |
| Agents | `agents/*.md` | Claude Code only, per-capability | ~3K tokens each |

---

## Usage

### Claude Code

Copy `rules.md` content into your project's `CLAUDE.md` or global `~/.claude/CLAUDE.md`. Append the relevant language standard. Copy agent files to `.claude/agents/`.

### Cursor

Copy `rules.md` content into `.cursor/rules/core.mdc` with frontmatter:

```yaml
---
description: Core coding rules
globs: ["**/*"]
alwaysApply: true
---
```

Add language standards as separate `.mdc` files with appropriate glob patterns.

### Codex (OpenAI)

Copy `rules.md` content into the root `AGENTS.md`. Place language standards in relevant subdirectories.

### Other Tools

Copy `rules.md` into the tool's instruction file (`.windsurfrules`, `.clinerules`, etc.). Append language standards as needed.

---

## Adding Content

### New Language Standard

Create `standards/{language}.md`. Follow the structure of existing standards: version header, quick reference table, rules as tables, minimal code examples, forbidden patterns, pre-commit checklist.

### New Domain Knowledge

Create `knowledge/{domain}/` with reference files. Add a Critical Rules Summary section at the top of each file (~80 lines) for the rules AI must always follow. Full reference below for on-demand reading.

### New Agent

Create `agents/{name}.md` following Claude Code agent format. Only universal agents belong here. Project-specific agents belong in their project's `.claude/agents/`.

---

## License

MIT License - see [LICENSE](LICENSE)
