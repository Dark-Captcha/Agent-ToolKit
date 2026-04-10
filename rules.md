# Core Rules

> **Version:** 1.0.0 | **Status:** Active | **Updated:** 2026-04-10

Imperative. No exceptions unless stated.

---

## Setup

- Use CLI for all config changes.
- Hand-edit only files with no CLI support (scripts, metadata).

---

## Before Writing Code

| Rule                  | Detail                                                                                        |
| --------------------- | --------------------------------------------------------------------------------------------- |
| Read first            | Read existing code and trace dependencies before proposing changes.                           |
| Cite sources          | Name the files you read and what you learned from each.                                       |
| State approach        | Explain what the code must do and why this approach is correct for this specific problem.     |
| Trace logic           | Walk through at least one concrete input and expected output before writing.                  |
| Name edge cases       | List what inputs will break it and what assumptions must hold.                                |
| Verify patterns       | Never apply a pattern just because it looks familiar. Confirm it fits the actual constraints. |
| Ask, do not guess     | If the logic is genuinely unclear from the request, ask. Do not guess and write.              |
| Wait for confirmation | On non-trivial changes, state your plan and wait before implementing.                         |

---

## Code

| Rule                       | Detail                                                                                   |
| -------------------------- | ---------------------------------------------------------------------------------------- |
| Explicit imports           | Import explicitly at the top. Never use wildcards or inline paths.                       |
| Read before editing        | Read existing code and cross-references before modifying.                                |
| Small changes              | Make small changes. Keep working code. Match existing style.                             |
| No unrequested refactoring | Never refactor code the user did not ask to change.                                      |
| Propagate errors           | Propagate errors with context. Never swallow errors or use catch-all handlers.           |
| Complete implementation    | Write complete, working code. Never use placeholders, stubs, TODOs, or empty bodies.     |
| No fake completeness       | If a full implementation requires more than one response, say so and ask where to start. |

---

## After Writing Code

| Rule           | Detail                                                                                   |
| -------------- | ---------------------------------------------------------------------------------------- |
| Self-review    | Check every function for: error handling, security issues, edge cases, style violations. |
| Run tests      | Run tests after every change. Fix failures before moving on. Never skip.                 |
| Verify logic   | Do not assume a working-looking output means the logic is correct. Trace it.             |
| Security check | Check for injection, XSS, hardcoded secrets, unsafe patterns. Fix immediately if found.  |

---

## Dependencies

- Prefer stdlib and existing deps.
- Justify any new package. Never add one for trivial tasks.

---

## Debug

Debug in this order:

| Order | Check                 |
| ----- | --------------------- |
| 1     | Usage errors          |
| 2     | Config                |
| 3     | Docs                  |
| 4     | Version compatibility |

- Never downgrade a library as a first step.
- Never repeat a failed approach. Diagnose why it failed first.

---

## Git

- Never commit, amend, or rebase unless asked.
- Never skip hooks or bypass signing unless asked.

---

## Communication

| Rule             | Detail                                                              |
| ---------------- | ------------------------------------------------------------------- |
| Ask when unclear | Do not assume. Ask specific numbered questions.                     |
| Flag wrong tasks | If a request is illogical or will break things, say so immediately. |
| No repetition    | Never repeat a failed approach. Never confirm what did not work.    |
| Act on decisions | Once the user decides, act. Never re-offer rejected alternatives.   |
| Be concise       | Lead with the answer or action, not the reasoning. Skip filler.     |
