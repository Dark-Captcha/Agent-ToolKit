# Agent-ToolKit

> **Version:** 1.0.1 | **Updated:** 2026-05-26

Universal rules, coding standards, and domain knowledge for AI coding agents. Works across any tool that reads markdown instructions.

---

| #   | Section                                 |
| --- | --------------------------------------- |
| 1   | [Structure](#structure)                 |
| 2   | [Design Principles](#design-principles) |
| 3   | [Loading Strategy](#loading-strategy)   |
| 4   | [Usage](#usage)                         |
| 5   | [Adding Content](#adding-content)       |
| 6   | [License](#license)                     |

---

## Structure

```text
Agent-ToolKit/
├── rules.md                 <- Universal behavior rules (always load)
├── standards/               <- Language-specific coding standards (load per language)
│   ├── rust.md
│   ├── typescript.md
│   ├── python.md
│   ├── javascript.md
│   └── mojo.md
├── knowledge/               <- Domain-specific reference material (load on demand)
│   └── oxc.md
├── agents/                  <- Agent definitions (tools with agent support)
│   ├── document-writer.md
│   └── experiment-runner.md
├── CLAUDE.md                <- Claude Code router (references rules.md, no inline mirror)
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

| Layer     | Files                   | When to load                             | Token cost      |
| --------- | ----------------------- | ---------------------------------------- | --------------- |
| Rules     | `rules.md`              | Every session, every tool                | ~500 tokens     |
| Standards | `standards/{lang}.md`   | When writing code in that language       | ~1K tokens each |
| Knowledge | `knowledge/{domain}.md` | When working in that domain              | Varies (large)  |
| Agents    | `agents/*.md`           | Tools with agent support, per-capability | ~3K tokens each |

---

## Usage

`rules.md` is the single source of truth. Wire each tool to it — prefer **symlink/reference** over copy. Only copy when the tool cannot follow links or read additional files.

| Tool        | Instruction file         | Wiring                                                                                   | Agent directory                                        |
| ----------- | ------------------------ | ---------------------------------------------------------------------------------------- | ------------------------------------------------------ |
| Claude Code | `~/.claude/CLAUDE.md`    | **Symlink** to `Agent-ToolKit/CLAUDE.md` — that file references `rules.md` via Bootstrap | Symlink `~/.claude/agents/` to `Agent-ToolKit/agents/` |
| Cursor      | `.cursor/rules/core.mdc` | Copy `rules.md` (Cursor needs frontmatter)                                               | N/A                                                    |
| Codex       | `AGENTS.md`              | Symlink to `Agent-ToolKit/rules.md` if possible, else copy                               | N/A                                                    |
| Windsurf    | `.windsurfrules`         | Symlink to `Agent-ToolKit/rules.md` if possible, else copy                               | N/A                                                    |
| Cline       | `.clinerules`            | Symlink to `Agent-ToolKit/rules.md` if possible, else copy                               | N/A                                                    |

### Claude Code setup (one-time)

```bash
ln -s ~/Documents/Agent-ToolKit/CLAUDE.md ~/.claude/CLAUDE.md
ln -s ~/Documents/Agent-ToolKit/agents   ~/.claude/agents
```

`Agent-ToolKit/CLAUDE.md` is a thin router: it tells the agent to read `rules.md` at session start, then lists on-demand triggers for language standards and knowledge files. No rules are duplicated inside it — edit `rules.md` and every tool that points here picks up the change.

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
