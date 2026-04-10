# TypeScript Coding Standards

> **Version:** 1.0.0 | **Status:** Active | **Updated:** 2026-04-10

Type everything. Avoid `any`. Explicit over implicit.

---

| #   | Section                                       |
| --- | --------------------------------------------- |
| 1   | [Quick Reference](#quick-reference)           |
| 2   | [File Structure](#file-structure)             |
| 3   | [Imports](#imports)                           |
| 4   | [Naming](#naming)                             |
| 5   | [Types](#types)                               |
| 6   | [Functions](#functions)                       |
| 7   | [Error Handling](#error-handling)             |
| 8   | [Async](#async)                               |
| 9   | [Classes](#classes)                           |
| 10  | [Testing](#testing)                           |
| 11  | [Documentation](#documentation)               |
| 12  | [Forbidden Patterns](#forbidden-patterns)     |
| 13  | [Pre-Commit Checklist](#pre-commit-checklist) |

---

## Quick Reference

| Rule      | Do                     | Don't              |
| --------- | ---------------------- | ------------------ |
| Types     | Explicit types         | `any`              |
| Imports   | Named imports          | `import *`         |
| Null      | Optional chaining `?.` | Nested `if` checks |
| Async     | `async/await`          | Callback chains    |
| Logging   | Structured logger      | `console.log`      |
| Equality  | `===` and `!==`        | `==` and `!=`      |
| Variables | `const` by default     | `let` everywhere   |

### Strict Config

```json
{
  "compilerOptions": {
    "strict": true,
    "noImplicitAny": true,
    "strictNullChecks": true,
    "noUncheckedIndexedAccess": true
  }
}
```

---

## File Structure

| Order | Section         | Required      |
| ----- | --------------- | ------------- |
| 1     | `@fileoverview` | Yes           |
| 2     | Imports         | Yes           |
| 3     | Constants       | If any        |
| 4     | Types           | If any        |
| 5     | Implementation  | Yes           |
| 6     | Exports         | If not inline |

---

## Imports

Group with blank lines between:

| Group    | Order | Example                                   |
| -------- | ----- | ----------------------------------------- |
| Node.js  | 1     | `import { readFile } from "fs/promises";` |
| External | 2     | `import { z } from "zod";`                |
| Internal | 3     | `import { User } from "../types.js";`     |

| Rule               | Good                            | Bad                           |
| ------------------ | ------------------------------- | ----------------------------- |
| Named imports      | `import { foo } from "mod";`    | `import * as mod from "mod";` |
| Type-only          | `import type { T } from "mod";` | Mixed with values             |
| File extensions    | `from "./mod.js";`              | `from "./mod";`               |
| No default exports | `export { MyClass };`           | `export default MyClass;`     |

---

## Naming

| Item              | Convention        | Example               |
| ----------------- | ----------------- | --------------------- |
| Types/Interfaces  | `PascalCase`      | `User`, `HttpClient`  |
| Functions/Methods | `camelCase`       | `findUser`, `isValid` |
| Constants         | `SCREAMING_SNAKE` | `MAX_RETRIES`         |
| Variables         | `camelCase`       | `userId`, `isActive`  |
| Files/Folders     | `kebab-case`      | `user-service.ts`     |

| Prefix/Suffix         | Use For         | Example                     |
| --------------------- | --------------- | --------------------------- |
| `is*`, `has*`, `can*` | Booleans        | `isActive`, `hasPermission` |
| `create*`             | Factories       | `createUser()`              |
| `handle*`             | Event handlers  | `handleClick()`             |
| `*Params`             | Parameter types | `CreateUserParams`          |
| `*Result`             | Return types    | `SearchResult`              |

---

## Types

| Use Interface         | Use Type Alias        |
| --------------------- | --------------------- |
| Object shapes         | Unions, intersections |
| Extendable contracts  | Computed types        |
| Class implementations | Branded types         |

Use branded types for type-safe IDs:

```typescript
type UserId = string & { readonly __brand: "UserId" };
```

Use discriminated unions for result types:

```typescript
type Result<T> =
  | { type: "success"; data: T }
  | { type: "error"; error: string };
```

Use const objects instead of enums:

```typescript
export const UserStatus = {
  Active: "active",
  Inactive: "inactive",
} as const;

export type UserStatus = (typeof UserStatus)[keyof typeof UserStatus];
```

---

## Functions

| Rule                  | Detail                                             |
| --------------------- | -------------------------------------------------- |
| Explicit return types | Always declare return type on public functions     |
| Parameter objects     | Use an interface for 3+ params                     |
| Early returns         | Guard clauses at top, main logic below             |
| Explicit temporaries  | Break chained calls into named steps for debugging |

---

## Error Handling

Extend `Error` for custom types with `code` and `statusCode`. Use the Result pattern for expected failures:

```typescript
type Result<T, E = Error> =
  | { success: true; data: T }
  | { success: false; error: E };
```

Catch at boundaries (route handlers, event listeners), not in business logic. Let errors propagate.

---

## Async

| Rule                      | Detail                                                   |
| ------------------------- | -------------------------------------------------------- |
| Always async/await        | Never use `.then()` chains                               |
| Parallel when independent | `Promise.all([a(), b(), c()])`                           |
| Explicit void             | `void fireAndForget()` for intentional floating promises |
| Always add timeouts       | `Promise.race` with timeout for external calls           |

---

## Classes

| Use Classes          | Use Functions          |
| -------------------- | ---------------------- |
| Stateful services    | Stateless utilities    |
| Dependency injection | Pure functions         |
| Complex lifecycle    | Simple transformations |

Prefer composition over inheritance. Use constructor injection for dependencies.

---

## Testing

| Rule       | Detail                                                |
| ---------- | ----------------------------------------------------- |
| Framework  | Vitest preferred                                      |
| Structure  | `describe` > `it("should ...")`                       |
| Pattern    | Arrange / Act / Assert                                |
| Assertions | Specific: `toEqual`, `toHaveLength`, not `toBeTruthy` |
| Mocking    | `vi.fn()`, `vi.mocked()`                              |

---

## Documentation

Use JSDoc on public functions:

```typescript
/**
 * Finds a user by email.
 *
 * @param email - Email to search
 * @returns User if found, null otherwise
 * @throws {ValidationError} If email invalid
 */
```

Use `/** comment */` on interface properties.

---

## Forbidden Patterns

| Pattern       | Why             | Alternative       |
| ------------- | --------------- | ----------------- |
| `any`         | No type safety  | `unknown` + guard |
| `@ts-ignore`  | Hides errors    | Fix the type      |
| `import *`    | Unclear imports | Named imports     |
| `console.log` | Unstructured    | Logger            |
| `eval()`      | Security risk   | Safe alternatives |
| `==`, `!=`    | Type coercion   | `===`, `!==`      |
| `var`         | Hoisting issues | `const`, `let`    |

| Exception          | Allowed When              |
| ------------------ | ------------------------- |
| `any`              | Missing third-party types |
| `@ts-expect-error` | With explanation comment  |
| `console.log`      | Development only          |

---

## Pre-Commit Checklist

```bash
npx tsc --noEmit
npx eslint .
npm test
```

| Check                           | Status |
| ------------------------------- | ------ |
| TypeScript compiles             | [ ]    |
| ESLint passes                   | [ ]    |
| Tests pass                      | [ ]    |
| No `any` types                  | [ ]    |
| No `@ts-ignore`                 | [ ]    |
| All public functions have JSDoc | [ ]    |
| Explicit return types           | [ ]    |
