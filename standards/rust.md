# Rust Coding Standards

> **Version:** 1.0.0 | **Status:** Active | **Updated:** 2026-04-10

Safe by default. Explicit over implicit. No panics in production.

---

| #   | Section                                      |
| --- | -------------------------------------------- |
| 1   | [Quick Reference](#quick-reference)          |
| 2   | [File Structure](#file-structure)            |
| 3   | [Imports](#imports)                          |
| 4   | [Naming](#naming)                            |
| 5   | [Types](#types)                              |
| 6   | [Functions](#functions)                      |
| 7   | [Error Handling](#error-handling)            |
| 8   | [Async](#async)                              |
| 9   | [Unsafe](#unsafe)                            |
| 10  | [Testing](#testing)                          |
| 11  | [Documentation](#documentation)             |
| 12  | [Forbidden Patterns](#forbidden-patterns)    |
| 13  | [Pre-Commit Checklist](#pre-commit-checklist) |

---

## Quick Reference

| Rule      | Do                           | Don't                              |
| --------- | ---------------------------- | ---------------------------------- |
| Errors    | `?` operator, `Result<T, E>` | `unwrap()`, `expect()`, `panic!()` |
| Imports   | Explicit items               | Wildcards `use foo::*`             |
| IDs       | Newtypes `UserId(u64)`       | Raw primitives                     |
| Logging   | `tracing` or `log`           | `println!()`                       |
| Strings   | `&str` for params            | `&String`                          |
| Iteration | Iterators                    | Index loops                        |
| Unsafe    | Documented, minimal          | Scattered, undocumented            |

---

## File Structure

| Order | Section               | Required    |
| ----- | --------------------- | ----------- |
| 1     | Module docs (`//!`)   | Yes         |
| 2     | Imports               | Yes         |
| 3     | Constants             | If any      |
| 4     | Types                 | If any      |
| 5     | Implementations       | If any      |
| 6     | Trait Implementations | If any      |
| 7     | Private helpers       | If any      |
| 8     | Tests                 | Recommended |

---

## Imports

Group with blank lines between:

| Group    | Order | Example                          |
| -------- | ----- | -------------------------------- |
| std      | 1     | `use std::collections::HashMap;` |
| External | 2     | `use serde::Serialize;`          |
| Crate    | 3     | `use crate::error::Result;`      |
| Parent   | 4     | `use super::Config;`             |

| Rule          | Good                          | Bad                       |
| ------------- | ----------------------------- | ------------------------- |
| Explicit      | `use std::io::{Read, Write};` | `use std::io::*;`         |
| Top of file   | Imports at top                | Inline `use` in functions |
| Group related | `use std::io::{Read, Write};` | Separate `use` for each   |

---

## Naming

| Item          | Convention        | Example                 |
| ------------- | ----------------- | ----------------------- |
| Types, Traits | `PascalCase`      | `UserId`, `HttpClient`  |
| Functions     | `snake_case`      | `find_user`, `is_valid` |
| Constants     | `SCREAMING_SNAKE` | `MAX_CONNECTIONS`       |
| Modules       | `snake_case`      | `user_service`          |
| Lifetimes     | Short lowercase   | `'a`, `'de`             |
| Type params   | Single uppercase  | `T`, `E`, `Item`        |

| Prefix/Suffix   | Use For              | Example          |
| --------------- | -------------------- | ---------------- |
| `is_*`, `has_*` | Boolean getters      | `is_empty()`     |
| `with_*`        | Builder methods      | `with_timeout()` |
| `into_*`        | Consuming conversion | `into_inner()`   |
| `as_*`          | Borrowing conversion | `as_str()`       |
| `try_*`         | Fallible operation   | `try_from()`     |

---

## Types

Use newtypes for IDs, domain concepts, units, and validated data:

```rust
#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash)]
pub struct UserId(u64);
```

| Type         | Derives                                           |
| ------------ | ------------------------------------------------- |
| Value types  | `Debug, Clone, Copy, PartialEq, Eq, Hash`         |
| Data structs | `Debug, Clone, PartialEq, Serialize, Deserialize` |
| Error types  | `Debug, Error` (thiserror)                        |
| Config       | `Debug, Clone, Default, Serialize, Deserialize`   |

Use `Arc<Inner>` pattern for shared ownership across threads.

---

## Functions

| Guideline                | Good                      | Bad                         |
| ------------------------ | ------------------------- | --------------------------- |
| Borrow when possible     | `fn process(data: &[u8])` | `fn process(data: Vec<u8>)` |
| Use `&str` not `&String` | `fn parse(s: &str)`       | `fn parse(s: &String)`      |
| Accept `impl Trait`      | `fn read(r: impl Read)`   | `fn read(r: &mut File)`     |

Prefer explicit temporaries over nested chains:

```rust
let request = Request::new(user_id, command);
let response = connection.send(request).await?;
let result = response.into_result()?;
```

---

## Error Handling

Use `thiserror` for library errors, `anyhow` for application errors:

```rust
#[derive(Error, Debug)]
pub enum Error {
    #[error("not found: {resource} with id {id}")]
    NotFound { resource: &'static str, id: String },

    #[error("io error: {0}")]
    Io(#[from] std::io::Error),
}

pub type Result<T> = std::result::Result<T, Error>;
```

Always propagate with `?`. Add context with `.map_err()`:

```rust
let file = File::open(path)
    .map_err(|e| Error::invalid_input(format!("cannot open {}: {}", path.display(), e)))?;
```

---

## Async

| Rule                      | Detail                                                            |
| ------------------------- | ----------------------------------------------------------------- |
| No blocking in async      | Use `tokio::fs` not `std::fs`. Use `spawn_blocking` for CPU work. |
| Parallel when independent | `tokio::join!` for concurrent operations.                         |
| Always add timeouts       | Wrap external calls with `timeout()`.                             |
| Use `select!` for racing  | First-completed wins.                                             |

---

## Unsafe

Every `unsafe` block requires:

| Requirement    | Detail                         |
| -------------- | ------------------------------ |
| Safety comment | Explain why it is safe         |
| Minimal scope  | Smallest possible unsafe block |
| Safe wrapper   | Public API must be safe        |

| Allowed                | Not Allowed             |
| ---------------------- | ----------------------- |
| FFI bindings           | Convenience             |
| Measured hot paths     | Premature optimization  |
| Low-level abstractions | Avoiding borrow checker |

---

## Testing

| Rule       | Detail                                                                             |
| ---------- | ---------------------------------------------------------------------------------- |
| Location   | `#[cfg(test)] mod tests` in file, integration tests in `tests/`                    |
| Naming     | `test_parse_valid_input_returns_ok` — descriptive                                  |
| Assertions | Always include context: `assert!(result.is_ok(), "Expected Ok, got {:?}", result)` |
| Async      | Use `#[tokio::test]`                                                               |

---

## Documentation

| Section       | When              |
| ------------- | ----------------- |
| Brief         | Always (one line) |
| `# Arguments` | Function params   |
| `# Returns`   | Return value      |
| `# Errors`    | Error conditions  |
| `# Safety`    | Unsafe functions  |
| `# Examples`  | Usage examples    |

---

## Forbidden Patterns

| Pattern               | Why            | Alternative      |
| --------------------- | -------------- | ---------------- |
| `unwrap()`            | Panics         | `?`, `ok_or()`   |
| `expect()`            | Panics         | `?` with context |
| `panic!()`            | Crashes        | Return `Result`  |
| `use foo::*`          | Unclear        | Explicit imports |
| `println!()`          | Unstructured   | `tracing`        |
| Undocumented `unsafe` | Unclear safety | Document it      |

| Exception  | Allowed When                  |
| ---------- | ----------------------------- |
| `unwrap()` | Tests only                    |
| `expect()` | Tests, or proven can't fail   |
| `panic!()` | Tests, or truly unrecoverable |

---

## Pre-Commit Checklist

```bash
cargo fmt
cargo clippy -- -D warnings
cargo test
```

| Check                         | Status |
| ----------------------------- | ------ |
| `cargo fmt` passes            | [ ]    |
| `cargo clippy -D warnings`    | [ ]    |
| `cargo test` passes           | [ ]    |
| No `unwrap()` in prod code    | [ ]    |
| All public items documented   | [ ]    |
| All unsafe has safety comment | [ ]    |
