# Global Instructions — Agent-ToolKit

> **Source:** `/home/gnusocute/Documents/Agent-ToolKit`
> **Layout:** rules (always) · standards (per-language, on-demand) · knowledge (per-domain, on-demand) · agents (`~/.claude/agents/`)
>
> If you edit `rules.md`, re-sync this file. The rules section below is a verbatim mirror of `rules.md` v1.1.0.

---

## Source layout

```
/home/gnusocute/Documents/Agent-ToolKit/
├── rules.md                 ← inlined below (always loaded)
├── standards/               ← read on demand, per language
│   ├── rust.md              v1.0.0
│   ├── typescript.md        v1.0.0
│   ├── python.md            v1.0.0
│   └── javascript.md        v1.0.0
├── knowledge/               ← read on demand, per domain
│   └── oxc.md               v2.0.0 (verified against OXC v0.135.0)
├── agents/                  ← symlinked into ~/.claude/agents/
│   ├── document-writer.md   v1.0.0
│   └── experiment-runner.md v1.0.0
├── CLAUDE.md                ← this file
├── README.md                v1.0.1
└── LICENSE                  (MIT)
```

---

## On-demand loading

Read the matching file before producing code or guidance in that area. Do not rely on prior knowledge — load the current rules from disk.

| Trigger                                             | File to read first                                                    |
| --------------------------------------------------- | --------------------------------------------------------------------- |
| Writing or modifying **Rust** code                  | `/home/gnusocute/Documents/Agent-ToolKit/standards/rust.md`           |
| Writing or modifying **TypeScript** code            | `/home/gnusocute/Documents/Agent-ToolKit/standards/typescript.md`     |
| Writing or modifying **JavaScript** code            | `/home/gnusocute/Documents/Agent-ToolKit/standards/javascript.md`     |
| Writing or modifying **Python** code                | `/home/gnusocute/Documents/Agent-ToolKit/standards/python.md`         |
| Working on/with **oxc** (oxidation compiler)        | `/home/gnusocute/Documents/Agent-ToolKit/knowledge/oxc.md`            |
| Writing or reformatting **technical documentation** | `/home/gnusocute/Documents/Agent-ToolKit/agents/document-writer.md`   |
| Running **autonomous optimization experiments**     | `/home/gnusocute/Documents/Agent-ToolKit/agents/experiment-runner.md` |

Read once per session per file. If the task spans multiple languages, load each standard before writing in that language.

---

## Agents

| Agent             | File                          | Invoke via | Purpose                                               |
| ----------------- | ----------------------------- | ---------- | ----------------------------------------------------- |
| document-writer   | `agents/document-writer.md`   | Agent tool | Format/rewrite technical docs per project conventions |
| experiment-runner | `agents/experiment-runner.md` | Agent tool | Autonomous iterative metric optimization              |

Sourced from Agent-ToolKit (symlinked into `~/.claude/agents/`).

---

## Tool integration

| Tool        | Agent-ToolKit usage                                    |
| ----------- | ------------------------------------------------------ |
| Claude Code | Copy `rules.md` into `CLAUDE.md`. Append language std. |
| Codex       | Copy `rules.md` into `AGENTS.md`. Append language std. |
| Cursor      | Import `rules.md` as `.cursor/rules/*.mdc`.            |
| Windsurf    | Copy `rules.md` into `.windsurfrules`.                 |
| Cline       | Copy `rules.md` into `.clinerules`.                    |

---

# Core Rules

> **Version:** 1.1.0 | **Status:** Active | **Updated:** 2026-05-26

Imperative. No exceptions unless stated.

---

| #   | Section                                     |
| --- | ------------------------------------------- |
| 1   | [Setup](#setup)                             |
| 2   | [Before Writing Code](#before-writing-code) |
| 3   | [Code](#code)                               |
| 4   | [After Writing Code](#after-writing-code)   |
| 5   | [Dependencies](#dependencies)               |
| 6   | [Debug](#debug)                             |
| 7   | [Git](#git)                                 |
| 8   | [Communication](#communication)             |

---

## Setup

| Rule           | Detail                                                        |
| -------------- | ------------------------------------------------------------- |
| Use CLI        | Use CLI for all config changes.                               |
| Hand-edit only | Hand-edit only files with no CLI support (scripts, metadata). |

---

## Before Writing Code

| Rule                | Detail                                                                                               |
| ------------------- | ---------------------------------------------------------------------------------------------------- |
| Read first          | Read existing code and trace dependencies before proposing changes.                                  |
| Cite sources        | Name the files you read and what you learned from each.                                              |
| Surface assumptions | State assumptions before implementing. If multiple interpretations exist, present them.              |
| State approach      | Explain what the code must do and why this approach is correct for this specific problem.            |
| Define success      | Convert the request into verifiable success criteria before coding.                                  |
| Prefer simple plan  | Use the simplest approach that solves the request. Name tradeoffs when a heavier approach is needed. |
| Trace logic         | Walk through at least one concrete input and expected output before writing.                         |
| Name edge cases     | List what inputs will break it and what assumptions must hold.                                       |
| Verify patterns     | Never apply a pattern just because it looks familiar. Confirm it fits the actual constraints.        |
| Ask, do not guess   | If the logic is genuinely unclear from the request, ask. Do not guess and write.                     |
| Confirm when needed | For ambiguous, risky, or behavior-changing work, state the plan and wait. For clear work, proceed.   |

---

## Code

| Rule                       | Detail                                                                                          |
| -------------------------- | ----------------------------------------------------------------------------------------------- |
| Explicit imports           | Import explicitly at the top. Never use wildcards or inline paths.                              |
| Read before editing        | Read existing code and cross-references before modifying.                                       |
| Small changes              | Make small changes. Keep working code. Match existing style.                                    |
| No speculative code        | Do not add flexibility, configuration, abstractions, or future-proofing that was not asked for. |
| Trace every line           | Every changed line must directly support the user's request.                                    |
| No unrequested refactoring | Never refactor code the user did not ask to change.                                             |
| Clean own orphans          | Remove only unused imports, variables, functions, or files made obsolete by your changes.       |
| Propagate errors           | Propagate errors with context. Never swallow errors or use catch-all handlers.                  |
| Complete implementation    | Write complete, working code. Never use placeholders, stubs, TODOs, or empty bodies.            |
| No fake completeness       | If a full implementation requires more than one response, say so and ask where to start.        |

---

## After Writing Code

| Rule           | Detail                                                                                                                 |
| -------------- | ---------------------------------------------------------------------------------------------------------------------- |
| Self-review    | Check every function for: error handling, security issues, edge cases, style violations.                               |
| Test the claim | For bugs, reproduce the failure. For validation, test invalid inputs. For refactors, verify behavior before and after. |
| Run tests      | Run tests after every change. Fix failures before moving on. Never skip.                                               |
| Verify logic   | Do not assume a working-looking output means the logic is correct. Trace it.                                           |
| Security check | Check for injection, XSS, hardcoded secrets, unsafe patterns. Fix immediately if found.                                |
| Report gaps    | If tests cannot be run, say why and describe the closest verification performed.                                       |

---

## Dependencies

| Rule              | Detail                                                         |
| ----------------- | -------------------------------------------------------------- |
| Prefer stdlib     | Use stdlib and existing deps before adding new ones.           |
| Justify additions | Never add a package for trivial tasks. State why it is needed. |

---

## Debug

Debug in this order:

| Order | Check                 |
| ----- | --------------------- |
| 1     | Usage errors          |
| 2     | Config                |
| 3     | Docs                  |
| 4     | Version compatibility |

| Rule             | Detail                                                        |
| ---------------- | ------------------------------------------------------------- |
| No downgrades    | Never downgrade a library as a first step.                    |
| No blind retries | Never repeat a failed approach. Diagnose why it failed first. |

---

## Git

| Rule            | Detail                                           |
| --------------- | ------------------------------------------------ |
| No auto-commits | Never commit, amend, or rebase unless asked.     |
| No skipping     | Never skip hooks or bypass signing unless asked. |

---

## Communication

| Rule              | Detail                                                                                           |
| ----------------- | ------------------------------------------------------------------------------------------------ |
| Ask when unclear  | Do not assume. Ask specific numbered questions.                                                  |
| Name confusion    | If something is unclear, say exactly what is unclear before asking.                              |
| Flag wrong tasks  | If a request is illogical or will break things, say so immediately.                              |
| Push back plainly | If the requested approach is overcomplicated or likely wrong, say so and offer the simpler path. |
| No repetition     | Never repeat a failed approach. Never confirm what did not work.                                 |
| Act on decisions  | Once the user decides, act. Never re-offer rejected alternatives.                                |
| Be concise        | Lead with the answer or action, not the reasoning. Skip filler.                                  |
