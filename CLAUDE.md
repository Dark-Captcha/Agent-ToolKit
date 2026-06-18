# Global Instructions — Agent-ToolKit

> **Source:** `~/Documents/Agent-ToolKit`
> Routing only. Rules live in [`rules.md`](rules.md). Language standards live in [`standards/`](standards/).

---

## Bootstrap

**Read [`rules.md`](rules.md) before responding to the first user message of every session.** Those rules are always in force. They are referenced here, not inlined, so this file stays a single source of routing.

---

## Source layout

```
~/Documents/Agent-ToolKit/
├── rules.md                 # always-loaded behavioral rules (read at bootstrap)
├── standards/               # language-specific, on-demand
│   ├── rust.md
│   ├── typescript.md
│   ├── python.md
│   ├── javascript.md
│   └── mojo.md
├── knowledge/               # domain-specific, on-demand
│   └── oxc.md               # (verified against OXC v0.135.0)
├── agents/                  # symlinked into ~/.claude/agents/
│   ├── document-writer.md
│   └── experiment-runner.md
├── CLAUDE.md                # this file (symlinked into ~/.claude/CLAUDE.md)
├── README.md
└── LICENSE
```

File versions are not pinned here — read the file itself if a version matters.

---

## On-demand loading

Read the matching file the first time the task touches that area. Do not rely on prior knowledge — load the current content from disk.

| Trigger                                             | File to read first                                      |
| --------------------------------------------------- | ------------------------------------------------------- |
| Writing or modifying **Rust** code                  | `~/Documents/Agent-ToolKit/standards/rust.md`           |
| Writing or modifying **TypeScript** code            | `~/Documents/Agent-ToolKit/standards/typescript.md`     |
| Writing or modifying **JavaScript** code            | `~/Documents/Agent-ToolKit/standards/javascript.md`     |
| Writing or modifying **Python** code                | `~/Documents/Agent-ToolKit/standards/python.md`         |
| Writing or modifying **Mojo** code                  | `~/Documents/Agent-ToolKit/standards/mojo.md`           |
| Working on/with **oxc** (oxidation compiler)        | `~/Documents/Agent-ToolKit/knowledge/oxc.md`            |
| Writing or reformatting **technical documentation** | `~/Documents/Agent-ToolKit/agents/document-writer.md`   |
| Running **autonomous optimization experiments**     | `~/Documents/Agent-ToolKit/agents/experiment-runner.md` |

Read once per session per file. If the task spans multiple languages, load each standard before writing in that language.

---

## Agents

| Agent             | File                          | Invoke via | Purpose                                               |
| ----------------- | ----------------------------- | ---------- | ----------------------------------------------------- |
| document-writer   | `agents/document-writer.md`   | Agent tool | Format/rewrite technical docs per project conventions |
| experiment-runner | `agents/experiment-runner.md` | Agent tool | Autonomous iterative metric optimization              |

Sourced from Agent-ToolKit (symlinked into `~/.claude/agents/`).
