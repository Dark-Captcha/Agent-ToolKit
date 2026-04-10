# Python Coding Standards

> **Version:** 1.0.0 | **Status:** Active | **Updated:** 2026-04-10

Readability counts. Explicit over implicit. Type hints on public API. Fail fast with clear errors.

---

## Quick Reference

| Rule       | Do                                   | Don't                             |
| ---------- | ------------------------------------ | --------------------------------- |
| Style      | PEP 8 (Ruff / Black)                 | Inconsistent formatting           |
| Types      | Annotations (PEP 484, 585, 604)      | Untyped public APIs               |
| Docstrings | Google style (Args, Returns, Raises) | Missing or inconsistent docs      |
| Imports    | Explicit, grouped (stdlib/3rd/local) | `from foo import *`               |
| Errors     | Specific exceptions, propagate       | Bare `except`, swallow errors     |
| Iteration  | Iterators, comprehensions            | Manual index loops when avoidable |
| Constants  | Module-level `UPPER_SNAKE`           | Magic numbers in code             |

---

## File Structure

| Order | Section              | Required                        |
| ----- | -------------------- | ------------------------------- |
| 1     | Module docstring     | Yes (PEP 257)                   |
| 2     | Imports              | Yes                             |
| 3     | Constants            | If any                          |
| 4     | Types / Type aliases | If any                          |
| 5     | Public API           | If any                          |
| 6     | Private helpers      | If any                          |
| 7     | Tests                | Recommended (often in `tests/`) |

---

## Imports

Group with blank lines between (PEP 8):

| Group        | Order | Example                                  |
| ------------ | ----- | ---------------------------------------- |
| Standard lib | 1     | `import os` / `from pathlib import Path` |
| Third-party  | 2     | `import httpx`                           |
| Local        | 3     | `from .errors import AppError`           |

| Rule               | Good                           | Bad                                       |
| ------------------ | ------------------------------ | ----------------------------------------- |
| Explicit           | `from os import path, environ` | `from os import *`                        |
| Top of file        | All imports after docstring    | Imports inside functions                  |
| Absolute preferred | `from mypkg.utils import foo`  | Deep relative `from ....utils import foo` |
| No unused          | Remove or use                  | Unused imports                            |

---

## Naming

| Item            | Convention         | Example                    |
| --------------- | ------------------ | -------------------------- |
| Modules         | `lower_with_under` | `user_service`             |
| Classes         | `CapWords`         | `HttpClient`, `UserId`     |
| Functions, vars | `lower_with_under` | `find_user`, `max_retries` |
| Constants       | `UPPER_WITH_UNDER` | `DEFAULT_TIMEOUT`          |
| Private         | Leading `_`        | `_internal_helper`         |
| Type vars       | `CapWords`         | `T`, `KT`, `VT`            |

| Prefix/Suffix   | Use For            | Example          |
| --------------- | ------------------ | ---------------- |
| `is_*`, `has_*` | Boolean predicates | `is_valid()`     |
| `_*`            | Internal / private | `_parse_token()` |

---

## Types

Use modern syntax (Python 3.10+):

```python
def parse_id(value: str | int) -> int: ...
def process(items: list[str], lookup: dict[str, int]) -> list[tuple[str, int]]: ...
```

| Pattern                   | Use For                                       |
| ------------------------- | --------------------------------------------- |
| `NewType("UserId", int)`  | Domain IDs, validated concepts                |
| `TypedDict`               | Structured dicts, `**kwargs` typing (PEP 692) |
| `@dataclass(frozen=True)` | Immutable structured data                     |
| `class Status(str, Enum)` | Enumerations                                  |

---

## Functions

Use Google style docstrings. Types in the signature make docstring types optional, but descriptions are still required:

```python
def find_user(user_id: UserId, include_deleted: bool = False) -> User | None:
    """Load a user by ID from the store.

    Args:
        user_id: The user's unique identifier.
        include_deleted: If True, include soft-deleted users.

    Returns:
        The user if found, otherwise None.

    Raises:
        AppError: When the backing store is unavailable.
    """
```

| Rule                       | Detail                                 |
| -------------------------- | -------------------------------------- |
| Immutable defaults         | Use `None`, never mutable `[]` or `{}` |
| Keyword-only for many args | `def fn(a: int, *, b: int, c: int)`    |
| Explicit temporaries       | Break chained calls into named steps   |

---

## Error Handling

```python
class AppError(Exception):
    """Base exception for this application."""

class NotFoundError(AppError):
    """Resource was not found."""
```

| Rule             | Detail                                 |
| ---------------- | -------------------------------------- |
| Be specific      | `except ValueError` not bare `except`  |
| Chain exceptions | `raise ... from e` when re-raising     |
| Do not swallow   | Log and re-raise, or handle explicitly |

---

## Async

| Rule                      | Detail                                              |
| ------------------------- | --------------------------------------------------- |
| Do not block event loop   | Use `asyncio.to_thread()` for blocking I/O          |
| Parallel when independent | `asyncio.gather(a(), b())`                          |
| Always add timeouts       | `async with asyncio.timeout(30)`                    |
| Use async libraries       | `httpx`, `aiofiles` instead of blocking equivalents |

---

## Testing

| Rule       | Detail                                                          |
| ---------- | --------------------------------------------------------------- |
| Framework  | pytest                                                          |
| Naming     | `test_parse_valid_input_returns_ok` — descriptive               |
| Assertions | Specific with messages: `assert actual == expected, "mismatch"` |
| Async      | `@pytest.mark.asyncio`                                          |

---

## Forbidden Patterns

| Pattern                  | Why                | Alternative                |
| ------------------------ | ------------------ | -------------------------- |
| `from x import *`        | Unclear namespace  | Explicit imports           |
| Bare `except:`           | Hides bugs         | `except SpecificError`     |
| Swallowing errors        | Hard to debug      | Log and re-raise or handle |
| Mutable default args     | Shared state       | `None` + assign in body    |
| `assert` for validation  | Disabled with `-O` | Explicit checks + `raise`  |
| Public API without types | Hard to use safely | Annotate public API        |

| Exception        | Allowed When                              |
| ---------------- | ----------------------------------------- |
| `assert`         | Invariants in tests only                  |
| Broad catch      | Top-level handler that logs and re-raises |
| `# type: ignore` | With comment explaining why               |

---

## Pre-Commit Checklist

```bash
ruff check .
ruff format --check .
mypy .
pytest
```

| Check                    | Status |
| ------------------------ | ------ |
| Ruff (lint) passes       | [ ]    |
| Ruff (format) passes     | [ ]    |
| mypy (or pyright) passes | [ ]    |
| pytest passes            | [ ]    |
| No bare `except`         | [ ]    |
| Public API documented    | [ ]    |
| Public API typed         | [ ]    |
