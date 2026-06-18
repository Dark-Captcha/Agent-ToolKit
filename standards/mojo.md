# Mojo Coding Standards

> **Version:** 1.2.0 | **Updated:** 2026-06-18 | **Mojo:** 1.0.0b3

Explicit over implicit. Comptime over runtime where possible. `raises` for errors, `Optional` for absence. No `let`, no `class`, no wildcards.

> Derived from real usage across `chrono-mojo`, `crypto-mojo`, `socket-mojo`, `tls-mojo`, `logging-mojo`, `http-client-mojo` — and from each project's `.probe/SYNTAX.md` (verified-on-disk language facts, not memory).

---

| #   | Section                                                |
| --- | ------------------------------------------------------ |
| 1   | [Quick Reference](#quick-reference)                    |
| 2   | [Project Layout](#project-layout)                      |
| 3   | [File Structure](#file-structure)                      |
| 4   | [Imports](#imports)                                    |
| 5   | [Naming](#naming)                                      |
| 6   | [Types](#types)                                        |
| 7   | [Functions](#functions)                                |
| 8   | [Parameters & Compile-Time](#parameters--compile-time) |
| 9   | [Memory & Ownership](#memory--ownership)               |
| 10  | [Error Handling](#error-handling)                      |
| 11  | [FFI](#ffi)                                            |
| 12  | [Testing](#testing)                                    |
| 13  | [Documentation](#documentation)                        |
| 14  | [Forbidden Patterns](#forbidden-patterns)              |
| 15  | [Pre-Commit Checklist](#pre-commit-checklist)          |
| 16  | [Mojo 1.0.0b3 Gotchas](#mojo-100b3-gotchas)            |

---

## Quick Reference

| Rule            | Do                                  | Don't                               |
| --------------- | ----------------------------------- | ----------------------------------- |
| Function decl   | `def`                               | `fn` (not used in practice)         |
| Type decl       | `struct`, `trait`                   | `class`                             |
| Bindings        | `var`                               | `let` (deprecated)                  |
| Constants       | `comptime NAME = ...` at module     | Runtime `var` for constants         |
| Errors          | `raises` + `raise Error("msg")`     | Sentinel returns, swallowed errors  |
| Absence         | `Optional[T]`                       | Magic-value returns                 |
| Imports         | Explicit, absolute, top of file     | Wildcards, inline imports           |
| Hot inline      | `@always_inline`                    | Manual unrolling                    |
| Move ownership  | `value^`                            | Implicit copies on heavy types      |
| Move semantics  | `mut self` for mutators             | Re-assigning fields via copy        |
| Param ownership | `def f(var x: T)` to take ownership | `def f(owned x: T)` (removed)       |
| Destructor      | `def __del__(deinit self)`          | `def __del__(owned self)` (removed) |
| FFI             | `external_call["name", Ret](args)`  | Linking shims, wrapper libraries    |
| Stdlib I/O      | `std.io.FileDescriptor`, `std.os.*` | `external_call["write"/"getenv"]`   |

---

## Project Layout

The de-facto layout that emerged across all surveyed projects:

```
pkg-mojo/                   # repo name = <pkg>-mojo
├── pixi.toml               # required: workspace + tasks + deps
├── pixi.lock               # committed
├── .gitignore              # required
├── README.md               # required: what is this, install, quick example
├── LICENSE                 # required
├── ARCHITECTURE.md         # map of the system (add once layers stabilize)
├── ROADMAP.md              # future direction
├── CHANGELOG.md            # per-release notes, from v0.1.0 onward
├── PERF.md                 # if performance is a feature (socket, crypto)
├── AUDIT.md                # if security-critical (crypto, tls)
├── docs/                   # promote here when single files aren't enough
│   ├── 01-purpose.md
│   ├── ...
│   └── adr/                # Architecture Decision Records (one per file)
│       └── 0001-<decision>.md
├── pkg/                    # package dir matches project name (sans -mojo)
│   ├── __init__.mojo       # public re-exports
│   ├── <thing>.mojo        # flat public types, one concern per file
│   ├── <group>/            # subdir ONLY for a real set/family
│   │   ├── __init__.mojo
│   │   └── <member>.mojo
│   ├── _core/              # foundation — zero internal deps (optional)
│   └── _internal/          # private helpers for upper layers
├── tests/
│   ├── run_tests.mojo      # aggregator entry point
│   ├── __init__.mojo
│   └── <group>/            # mirrors package when it grows
│       └── test_<thing>.mojo
├── benchmarks/             # optional; named `bench/` in some projects
├── examples/               # optional
└── .probe/                 # scratch area for verified-on-disk language facts
    ├── SYNTAX.md           #   — Mojo version quirks PROVEN by `mojo run`,
    └── *.mojo, *.py, *.txt #     plus KAT vectors / reference impls
```

**Multi-word package names** use `snake_case` for the directory and repo uses `kebab-case`: `http-client-mojo/` → `http_client/` package. Single-word is plain: `chrono-mojo/` → `chrono/`.

**`.probe/`** holds _probed, not remembered_ language facts and verification scaffolding (4 of 6 surveyed projects use it). Anything checked-in here should be reproducible by running the file — that's the rule. Listed in `.gitignore`? No — committed, because regression in a Mojo nightly is the thing it protects against.

### Scaffold

A new Mojo project starts with this exact shape, all dirs created **empty** except for `.probe/SYNTAX.md` (port from any existing project, then re-verify):

```
new-pkg-mojo/
├── pixi.toml                # set name, channel, mojo-compiler pin
├── new_pkg/                 # empty — package dir
├── tests/                   # empty — will hold run_tests.mojo aggregator
├── benchmarks/              # empty — for `pixi run bench`
├── examples/                # empty — for end-user-facing demos
└── .probe/
    └── SYNTAX.md            # copied from a sibling project, then verified
```

Empty dirs are committed (drop a `.gitkeep` if your VCS strips them). This is the `http-client-mojo` shape — pre-built scaffold so the first file you add already knows where to live.

### Project Documentation

Six files cover most of what you need. Add the next tier only when the project genuinely earns it.

| Tier | File                 | When to add                                                         |
| ---- | -------------------- | ------------------------------------------------------------------- |
| 1    | `README.md`          | Always — what is this, install, quick example                       |
| 1    | `LICENSE`            | Always — without it the code is legally unusable                    |
| 1    | `.gitignore`         | Always                                                              |
| 2    | `ARCHITECTURE.md`    | Once layers stabilize — map of the system, layers, dep direction    |
| 2    | `ROADMAP.md`         | Once direction matters — future work, milestones                    |
| 2    | `CHANGELOG.md`       | From the first tagged release onward — per-version notes            |
| 3    | `PERF.md`            | When performance is a feature (network, crypto, SIMD)               |
| 3    | `AUDIT.md`           | When security-critical (crypto, auth, TLS, parsers)                 |
| 3    | `SECURITY.md`        | When accepting external vuln reports (GitHub auto-detects)          |
| 3    | `TESTING.md`         | When test methodology is non-obvious (differential, property, KAT)  |
| 4    | `CONTRIBUTING.md`    | When accepting external PRs                                         |
| 4    | `CODE_OF_CONDUCT.md` | When project goes public and community grows                        |
| 5    | `docs/`              | When single files no longer fit — promote to a structured directory |
| 5    | `docs/adr/`          | One file per major decision (Architecture Decision Record pattern)  |

**Doc naming choices that matter:**

| Question                                  | Pick                                                                                                                              |
| ----------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------- |
| `ARCHITECTURE.md` vs `DESIGN.md`?         | `ARCHITECTURE.md` for the system map. `DESIGN.md` only if you need a separate per-decision rationale doc — better as `docs/adr/`. |
| `ROADMAP.md` vs `STATE.md` / `STATUS.md`? | `ROADMAP.md`. `STATE.md` rots in 24h; git log + ROADMAP cover present + future.                                                   |
| Where do release notes live?              | `CHANGELOG.md` from the first tag. Use Keep-a-Changelog format.                                                                   |

**Anti-pattern doc names — banned:**

| Name                       | Why it's bad                                | Replace with                                  |
| -------------------------- | ------------------------------------------- | --------------------------------------------- |
| `NOTES.md`                 | Bucket name, attracts junk                  | A specific doc — `ARCHITECTURE`, `AUDIT`, ... |
| `IDEAS.md`                 | Same                                        | GitHub Issues / Discussions                   |
| `TODO.md`                  | Overlaps `ROADMAP.md`                       | `ROADMAP.md` or issues                        |
| `MISC.md`, `INFO.md`       | Says nothing                                | Name what's actually inside, or delete        |
| `STATUS.md` (solo project) | Rots fast — present becomes past per commit | Skip; `git log` + `ROADMAP.md` cover it       |

**Minimal recommended set for a Mojo library (solo, perf-critical):**

```
README.md, LICENSE, ARCHITECTURE.md, ROADMAP.md, CHANGELOG.md  ← core 5
+ PERF.md     (if perf is a feature)
+ AUDIT.md    (if security-critical)
+ docs/adr/   (when >5 major decisions need recording)
```

Stop at 6–7 docs. The 8th is usually a bucket file masquerading as content.

---

| Layer            | Role                                                                                                                      |
| ---------------- | ------------------------------------------------------------------------------------------------------------------------- |
| `pkg/__init__`   | Public API surface — re-exports only, no logic                                                                            |
| `pkg/*.mojo`     | Public types, flat at the top, one concern per file                                                                       |
| `pkg/<group>/`   | Subdir when family forms (formats, protocols, locales, countries)                                                         |
| `pkg/_core/`     | Foundation primitives — zero internal deps. Split from `_internal` only when there's a real dependency layer between them |
| `pkg/_internal/` | Private helpers shared across upper layers. May import from `_core` only                                                  |

| Rule                                                                                                                       |
| -------------------------------------------------------------------------------------------------------------------------- |
| Imports point **downward** only: `_core` ← `_internal` ← public ← features. Never cycle.                                   |
| `_` prefix = private. Outside code does not import these.                                                                  |
| Flat top has a soft ceiling around **20 files**. Beyond that, group into subdirs.                                          |
| Subdir for a real set (formats, countries) OR a family with shared root noun mid-grow.                                     |
| **Shared root noun rule:** 2+ files sharing a stem (`datetime`, `offset_datetime`, `zoned_datetime`) → grouping candidate. |
| One file = one concern. Filename should let a reader guess ~80% of the contents.                                           |
| Banned bucket names: `utils/`, `helpers/`, `common/`, `misc/`, `types/`.                                                   |

---

## File Structure

| Order | Section                           | Required |
| ----- | --------------------------------- | -------- |
| 1     | Module header (`#` comment block) | Yes      |
| 2     | Imports                           | Yes      |
| 3     | `comptime` constants              | If any   |
| 4     | Type aliases                      | If any   |
| 5     | Structs / traits                  | If any   |
| 6     | Module-private helpers (`def _`)  | If any   |
| 7     | Public functions                  | If any   |

Module headers are written as `#` line comments at the top of the file — **not** triple-quoted docstrings. State purpose, key algorithms/RFCs, and any honesty notes (verification status, test seam).

---

## Imports

Absolute paths from the package root. No wildcards. Group with blank lines:

| Group        | Order | Example                                       |
| ------------ | ----- | --------------------------------------------- |
| Standard lib | 1     | `from sys.ffi import external_call`           |
| Third-party  | 2     | (rare)                                        |
| Local (pkg)  | 3     | `from chrono._core.civil import YearMonthDay` |

| Rule                | Good                                              | Bad                             |
| ------------------- | ------------------------------------------------- | ------------------------------- |
| Explicit            | `from chrono.enums import Weekday, Month`         | `from chrono.enums import *`    |
| Absolute            | `from chrono._core.units import SECONDS_PER_HOUR` | Deep relative `from .....units` |
| Top of file         | All imports after the module header               | Imports inside functions        |
| Group related items | One `from X import (a, b, c)` block per source    | Many lines from same module     |

---

## Naming

| Item            | Convention        | Example                              |
| --------------- | ----------------- | ------------------------------------ |
| Types, Traits   | `PascalCase`      | `Date`, `Duration`, `Subscriber`     |
| Functions       | `snake_case`      | `days_since_epoch_from_date`         |
| Methods         | `snake_case`      | `total_milliseconds`, `from_seconds` |
| Variables       | `snake_case`      | `days_since_epoch`                   |
| Constants       | `SCREAMING_SNAKE` | `SECONDS_PER_DAY`, `INT32_MAX`       |
| Private items   | Leading `_`       | `_checked_scale`, `_BLOCK_LEN`       |
| Modules / files | `snake_case`      | `offset_datetime.mojo`               |
| Parameters      | `snake_case`      | `clock: ClockId = ClockId.REALTIME`  |

| Prefix/Suffix    | Use For                            | Example                   |
| ---------------- | ---------------------------------- | ------------------------- |
| `is_*`, `has_* ` | Boolean predicates                 | `is_leap_year()`          |
| `from_*`         | Static fallible/total constructors | `from_days_since_epoch()` |
| `to_*`           | Lossless conversion                | `to_iso_string()`         |
| `as_*`           | Borrowed view                      | `as_str()`                |
| `try_*`          | Fallible variant of total op       | `try_parse()`             |
| `_*`             | Private / module-internal          | `_apply_timeout()`        |

---

## Types

`struct` for concrete types, `trait` for contracts. No `class`. Declare trait conformance inline:

```mojo
struct Date(Comparable, TrivialRegisterPassable):
    var _days_since_epoch: Int32
```

| Pattern                                                     | Use For                                                                 |
| ----------------------------------------------------------- | ----------------------------------------------------------------------- |
| `struct Foo(TrivialRegisterPassable)`                       | Small POD-like values; passed in registers                              |
| `struct Foo(Comparable)`                                    | Ordered + equatable (implies equality; no `EqualityComparable`)         |
| `trait Foo(Copyable, Movable, ImplicitlyDeletable)`         | Behavior contract for owned types                                       |
| `trait Foo`                                                 | Bare contract for borrowed-only use                                     |
| Parameterized: `struct Logger[S: Subscriber]`               | Generic over a trait-constrained type                                   |
| Generic struct field                                        | `var sub: Self.S` — **`Self.S`, not bare `S`**                          |
| Phantom param: `Instant[clock: ClockId = ClockId.REALTIME]` | Type-level discriminant                                                 |
| Nullable pointer                                            | `Optional[UnsafePointer[T, MutAnyOrigin]]` (pointer itself is non-null) |

Single-Int storage with decoded accessors is the canonical pattern for time/ID values — compare/sort/hash become integer ops; SIMD-friendly column layouts come free.

**`UnsafePointer` mutable origin is `MutAnyOrigin`** (not `MutableAnyOrigin`). **`Stringable` trait is gone** — expose `name() -> StaticString` or `write_to[W: Writer](self, mut w: W)`. **`@register_passable("trivial")` is removed** — use `TrivialRegisterPassable` trait conformance.

---

## Functions

All callables use `def`. Constructors take `out self`. Methods mutate via `mut self`:

```mojo
def __init__(out self, year: Int, month: Int, day: Int) raises:
    if month < 1 or month > 12:
        raise Error("chrono.Date: month out of range (1..12), got " + String(month))
    ...

@always_inline
def year(self) -> Int:
    return date_from_days_since_epoch(Int(self._days_since_epoch)).year
```

| Rule                      | Detail                                                    |
| ------------------------- | --------------------------------------------------------- |
| Constructor receiver      | `def __init__(out self, ...)`                             |
| Read-only method receiver | `def method(self, ...)`                                   |
| Mutating method receiver  | `def method(mut self, ...)`                               |
| Hot small methods         | `@always_inline` (accessors, register-trivial wrappers)   |
| Static factory            | `@staticmethod def from_xxx(...) raises -> Self`          |
| Keyword-only args         | `def fn(*, year: Int, month: Int)` for >2 same-typed args |
| Explicit temporaries      | Break chained calls into named `var` steps                |

---

## Parameters & Compile-Time

`[...]` is the compile-time parameter list; `(...)` is the runtime argument list. Push everything constant to `comptime`.

```mojo
comptime SECONDS_PER_DAY = SECONDS_PER_HOUR * HOURS_PER_DAY  # 86_400
comptime NANOS_PER_DAY   = SECONDS_PER_DAY * NANOS_PER_SECOND

comptime if CompilationTarget.is_macos():
    var rc = external_call["clock_gettime", Int32](_CLOCK_MONOTONIC_MACOS, ptr)
else:
    var rc = external_call["clock_gettime", Int32](_CLOCK_MONOTONIC_LINUX, ptr)
```

| Pattern                                   | Use For                                          |
| ----------------------------------------- | ------------------------------------------------ |
| `comptime NAME = expr`                    | Constants derived from other constants           |
| `comptime if CompilationTarget.is_...():` | Platform-specific code paths (zero runtime cost) |
| `struct Foo[T: Trait, *, dim: Int = 4]`   | Generics + named comptime knobs                  |
| `struct Foo[N: Int]`                      | Size-parameterized (buffers, SIMD widths)        |

| Rule                                                                              |
| --------------------------------------------------------------------------------- |
| If a value is known at build time, it goes through `comptime` — never runtime.    |
| Derive constants from each other (`B = A * factor`) so relationships can't drift. |
| Keep magic numbers off the call site — name them as `comptime` and reuse.         |

---

## Memory & Ownership

| Convention                 | Meaning                                                    |
| -------------------------- | ---------------------------------------------------------- |
| `self`                     | Borrowed receiver (read-only)                              |
| `mut self`                 | Mutable borrow of receiver                                 |
| `out self`                 | Receiver being initialized (constructors)                  |
| `var arg: T`               | Param: function takes ownership (replaces removed `owned`) |
| `value^`                   | Move expression — transfer ownership                       |
| `ref [origin] T`           | Borrowed reference with explicit origin                    |
| `def __del__(deinit self)` | Destructor (replaces `__del__(owned self)`)                |
| `def close(deinit self)`   | Consuming method that destroys `self` at the end           |

```mojo
def __init__(out self, *, var bytes_value: List[UInt8]):
    self._bytes = bytes_value^   # explicit move into field

def __del__(deinit self):
    _ = external_call["close", Int32](self.fd)
```

| Rule                                                                      |
| ------------------------------------------------------------------------- |
| Move (`^`) when handing off ownership of heavy values.                    |
| Implement `__del__` on any struct that owns a resource (fd, ptr, region). |
| Use `ref [self._field]` returns instead of copying out internal buffers.  |

---

## Error Handling

`raises` + `raise Error("msg")` is the default. `Optional[T]` for "may be absent" return values. No `Result`-style typed errors.

```mojo
def __init__(out self, year: Int, month: Int, day: Int) raises:
    if month < 1 or month > 12:
        raise Error("chrono.Date: month out of range (1..12), got " + String(month))

def next_completion(mut self) -> Optional[Completion]:
    if self._head == self._tail:
        return None
    ...
```

| Rule                 | Detail                                                              |
| -------------------- | ------------------------------------------------------------------- |
| Prefix every error   | `"pkg.module: <what's wrong> (<expected>), got <actual>"`           |
| Validate at boundary | Constructors, parsers, public entry points                          |
| Propagate inward     | Let `raises` bubble; do not wrap-and-rethrow without adding context |
| `Optional[T]`        | "Maybe present" — lookups, queue pops                               |
| Never swallow        | Catch only to add context, then re-raise                            |

---

## FFI

`external_call["symbol", ReturnType](args...)` from `sys.ffi`. No linker shims, no wrapper libs. Bind directly.

```mojo
from sys.ffi import external_call

var rc = external_call["clock_gettime", Int32](clockid, timespec.unsafe_ptr())

@always_inline
def _errno() -> Int32:
    return external_call["__errno_location", UnsafePointer[Int32, MutAnyOrigin]]()[]
```

| Rule                                                                                                                                                                                                                |
| ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Pin symbol name and return type as comptime strings/types.                                                                                                                                                          |
| Branch platform constants via `comptime if CompilationTarget.is_macos()` etc.                                                                                                                                       |
| Normalize errno conventions at the FFI boundary, not in callers.                                                                                                                                                    |
| Document the syscall/libc contract in the module header (clockid layout, etc.).                                                                                                                                     |
| **Prefer stdlib over FFI when both exist.** `external_call["write", ...]` conflicts with stdlib's own declaration — use `std.io.FileDescriptor(2).write(s)` for stderr, `std.os.getenv("VAR")`, `std.os.isatty(2)`. |

---

## Testing

| Convention     | Pattern                                                               |
| -------------- | --------------------------------------------------------------------- |
| Location       | `tests/` at project root, mirrors package shape as it grows           |
| Entry point    | `tests/run_tests.mojo` aggregates and reports total failures          |
| Per-file shape | `def run() raises -> Int:` returning failure count (0 == pass)        |
| Helpers        | `def _check(...) raises -> Int:` local to each test module            |
| Assertions     | `if got != want: print("FAIL ...", got, want); f += 1`                |
| Naming         | `test_<unit>.mojo`, descriptive: `test_parse_valid_input_round_trips` |
| Pixi task      | `test = "mojo run -I . tests/run_tests.mojo"`                         |

Layered tests for spec-driven code: **differential** (against an oracle), **property** (round-trip), **edge** (range/leap/boundary).

---

## Documentation

| Form                    | Use For                                                                   |
| ----------------------- | ------------------------------------------------------------------------- |
| Module header (`#`)     | Purpose, algorithms, RFCs, honesty/verification notes — top of every file |
| Docstring (`"""..."""`) | Public types and non-trivial public methods                               |
| Inline `#` comments     | Subtle invariants, RFC section refs, why-not-what                         |
| **Skip docstrings on**  | Trivial accessors, obvious wrappers                                       |

```mojo
# Date — a civil (proleptic Gregorian) calendar date, stored as a single Int32
# day count since 1970-01-01. Year/month/day decoded on demand via _core/civil.
#
# Facets: tier T0 | safety sound | quantum n/a.
# Honesty: from spec; verified differential vs Python `datetime`.

def __init__(out self, year: Int, month: Int, day: Int) raises:
    """Validating construction; raises on an out-of-range field."""
```

---

## Forbidden Patterns

| Pattern                             | Why                          | Alternative                                 |
| ----------------------------------- | ---------------------------- | ------------------------------------------- |
| `from x import *`                   | Hidden namespace             | Explicit imports                            |
| `class Foo:`                        | Not idiomatic; use `struct`  | `struct Foo`                                |
| `let x = ...`                       | Removed keyword              | `var x = ...`                               |
| `fn ...`                            | Removed keyword              | `def ...`                                   |
| `alias FOO = ...` at module scope   | Deprecated, warns            | `comptime FOO = ...`                        |
| `@parameter if`                     | Deprecated, warns            | `comptime if`                               |
| `@register_passable("trivial")`     | Removed                      | `struct Foo(TrivialRegisterPassable)`       |
| `def f(owned x: T)`                 | Removed in param lists       | `def f(var x: T)`                           |
| `def __del__(owned self)`           | Removed                      | `def __del__(deinit self)`                  |
| Module-level mutable `var`          | Error: globals not supported | Pass state down; `comptime` for constants   |
| `Stringable` / `EqualityComparable` | Traits gone                  | `Writer`-protocol / `Comparable` implies eq |
| `MutableAnyOrigin`                  | Not the name                 | `MutAnyOrigin`                              |
| `s[a:b]` byte-slicing               | Rejected by compiler         | `s[byte=a:b]`                               |
| `external_call["write", ...]`       | Conflicts with stdlib decl   | `std.io.FileDescriptor(fd).write(s)`        |
| `external_call["getenv"/"isatty"]`  | Stdlib already provides      | `std.os.getenv(...)`, `std.os.isatty(...)`  |
| Sentinel returns (`-1`, `""`)       | Ambiguous; easy to ignore    | `raises` or `Optional[T]`                   |
| Swallowed errors                    | Bugs hide                    | Add context, re-raise                       |
| Bucket-named folders (`utils/`)     | Attracts junk                | Name by role                                |
| Inline imports                      | Hard to grep                 | Top-of-file imports                         |
| Magic numbers in code               | Drifts, opaque               | `comptime NAME = ...` in shared file        |
| Per-call FFI symbol lookup helpers  | Unnecessary indirection      | `external_call` directly                    |
| Bare `S` for generic field          | Compiler wants `Self.S`      | `var sub: Self.S`                           |

---

## Pre-Commit Checklist

```bash
pixi run test
mblack chrono/ tests/   # if mblack configured
```

| Check                                                                      | Status |
| -------------------------------------------------------------------------- | ------ |
| `pixi run test` passes                                                     | [ ]    |
| No wildcards in imports                                                    | [ ]    |
| Module header present on every new file                                    | [ ]    |
| All public types/non-trivial methods have docstrings                       | [ ]    |
| Constants live in shared `comptime` modules                                | [ ]    |
| FFI calls platform-gated via `comptime if`                                 | [ ]    |
| `raises` / `Optional[T]` used over sentinel returns                        | [ ]    |
| Imports point downward (`_core` ← `_internal` ← rest)                      | [ ]    |
| No removed forms: `fn`, `let`, `owned`, `@register_passable`, `Stringable` | [ ]    |
| Stdlib used over `external_call` where it exists                           | [ ]    |

---

## Mojo 1.0.0b3 Gotchas

The current Mojo nightly has churn. Anything below is **probed, not remembered** — verify against your own `.probe/SYNTAX.md` before relying on it.

### Removed / replaced

| Old form                          | New form                                   |
| --------------------------------- | ------------------------------------------ |
| `fn name(...)`                    | `def name(...)`                            |
| `let x = ...`                     | `var x = ...`                              |
| `alias NAME = ...` (module-level) | `comptime NAME = ...`                      |
| `@parameter if`                   | `comptime if`                              |
| `@register_passable("trivial")`   | `struct Foo(TrivialRegisterPassable)`      |
| `def f(owned x: T)`               | `def f(var x: T)`                          |
| `def __del__(owned self)`         | `def __del__(deinit self)`                 |
| `MutableAnyOrigin`                | `MutAnyOrigin`                             |
| `s[a:b]` (byte slice on String)   | `s[byte=a:b]`                              |
| `Stringable` trait                | `name() -> StaticString` or `Writer` proto |
| `EqualityComparable`              | `Comparable` already implies equality      |

### Quirks worth remembering

| Form                                        | Note                                                               |
| ------------------------------------------- | ------------------------------------------------------------------ |
| Module-level mutable `var`                  | **Error** — globals not supported; pass state down                 |
| Generic struct field referencing type param | `var sub: Self.S` — `Self.` prefix required                        |
| Trait base for owned types                  | `trait T(Copyable, Movable, ImplicitlyDeletable)`                  |
| Trait method body                           | `...` ellipsis (declaration only)                                  |
| `Optional[UnsafePointer[T, O]]`             | The way to model nullable pointer (pointer is non-null)            |
| `comptime if` inside `@always_inline`       | DCE verified — erased branch produces no output                    |
| `String(some_struct)`                       | Requires Writer-protocol; fails with long candidate list otherwise |

### Stdlib supersedes FFI for these

| Need         | Use                                                             |
| ------------ | --------------------------------------------------------------- |
| Write to fd  | `from std.io import FileDescriptor; FileDescriptor(2).write(s)` |
| Read env var | `from std.os import getenv` → `String` (empty if unset)         |
| isatty       | `from std.os import isatty; isatty(2) -> Bool`                  |

> When this section gets stale (new Mojo version), update from `.probe/SYNTAX.md` — don't update from memory.
