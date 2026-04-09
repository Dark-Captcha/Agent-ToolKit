# JavaScript Coding Standards

> **Version:** 1.0.0 | **Status:** Active | **Updated:** 2026-04-10

Strict mode always. No implicit coercion. Explicit over clever.

---

## Quick Reference

| Rule       | Do                          | Don't                          |
| ---------- | --------------------------- | ------------------------------ |
| Mode       | `"use strict"` or ESM       | Sloppy mode                    |
| Equality   | `===` and `!==`             | `==` and `!=`                  |
| Variables  | `const` by default, `let`   | `var`                          |
| Modules    | ESM (`import`/`export`)     | CommonJS unless Node.js requires it |
| Async      | `async/await`               | Callback chains, `.then()`     |
| Logging    | Structured logger           | `console.log` in production    |
| Null check | Optional chaining `?.`      | Nested `if` checks             |
| Types      | JSDoc annotations on public API | Untyped public functions    |

---

## File Structure

| Order | Section        | Required      |
| ----- | -------------- | ------------- |
| 1     | `@fileoverview` | Yes          |
| 2     | Imports        | Yes           |
| 3     | Constants      | If any        |
| 4     | Implementation | Yes           |
| 5     | Exports        | If not inline |

---

## Modules

| Context | Use | Reason |
| ------- | --- | ------ |
| Browser | ESM | Native support, tree-shaking |
| Node.js (new) | ESM with `"type": "module"` in `package.json` | Modern standard |
| Node.js (legacy/compatibility) | CommonJS | When ESM is not supported by dependencies |

When using ESM:

```javascript
import { readFile } from "node:fs/promises";
import express from "express";
import { validate } from "./utils.js";
```

When forced into CommonJS:

```javascript
const { readFile } = require("node:fs/promises");
const express = require("express");
```

Never mix `require` and `import` in the same file.

---

## Imports

Group with blank lines between:

| Group    | Order | Example                                     |
| -------- | ----- | ------------------------------------------- |
| Node.js  | 1     | `import { readFile } from "node:fs/promises";` |
| External | 2     | `import express from "express";`            |
| Internal | 3     | `import { validate } from "./utils.js";`    |

| Rule           | Good                         | Bad                              |
| -------------- | ---------------------------- | -------------------------------- |
| Named imports  | `import { foo } from "mod";` | `import * as mod from "mod";`    |
| File extension | `from "./mod.js";`           | `from "./mod";` (ambiguous)      |
| Node prefix    | `from "node:fs";`            | `from "fs";` (ambiguous in ESM)  |

---

## Naming

| Item        | Convention        | Example              |
| ----------- | ----------------- | -------------------- |
| Classes     | `PascalCase`      | `UserService`        |
| Functions   | `camelCase`       | `findUser`, `isValid` |
| Constants   | `SCREAMING_SNAKE` | `MAX_RETRIES`        |
| Variables   | `camelCase`       | `userId`, `isActive` |
| Files       | `kebab-case`      | `user-service.js`    |
| Private     | Leading `_` or `#` | `_cache`, `#secret` |

| Prefix/Suffix | Use For | Example |
| ------------- | ------- | ------- |
| `is*`, `has*`, `can*` | Booleans | `isActive`, `hasPermission` |
| `create*` | Factories | `createUser()` |
| `handle*` | Event handlers | `handleClick()` |

---

## JSDoc Types

Use JSDoc for type information on public API. This enables editor intellisense without a build step:

```javascript
/**
 * @param {string} email
 * @param {FindOptions} [options]
 * @returns {Promise<User | null>}
 * @throws {ValidationError} If email invalid
 */
export async function findByEmail(email, options) {}
```

```javascript
/**
 * @typedef {Object} FindOptions
 * @property {number} [page]
 * @property {number} [pageSize]
 */
```

---

## Error Handling

```javascript
export class AppError extends Error {
  /** @param {string} message @param {string} code @param {number} [statusCode] */
  constructor(message, code, statusCode = 500) {
    super(message);
    this.name = this.constructor.name;
    this.code = code;
    this.statusCode = statusCode;
  }
}
```

| Rule | Detail |
| ---- | ------ |
| Custom error classes | Extend `Error` with `code` and `statusCode` |
| Catch at boundaries | Route handlers, event listeners — not in business logic |
| Never swallow | Always log or re-throw |
| Specific catches | Check `instanceof` before handling |

---

## Async

| Rule | Detail |
| ---- | ------ |
| Always async/await | Never use `.then()` chains |
| Parallel when independent | `Promise.all([a(), b(), c()])` |
| Explicit void | `void fireAndForget()` for intentional floating promises |
| Add timeouts | `Promise.race` with `setTimeout` rejection for external calls |
| Error handling | `try/catch` around await, not `.catch()` |

---

## Classes vs Functions

| Use Classes | Use Functions |
| ----------- | ------------- |
| Stateful services | Stateless utilities |
| Private state with `#` fields | Pure transformations |
| Complex lifecycle | Simple operations |

Prefer composition over inheritance. Use `#private` fields (ES2022) over naming conventions.

---

## Testing

| Rule | Detail |
| ---- | ------ |
| Framework | Vitest or Jest |
| Structure | `describe` > `it("should ...")` |
| Pattern | Arrange / Act / Assert |
| Assertions | Specific: `toEqual`, `toHaveLength`, not `toBeTruthy` |
| No globals in tests | Import explicitly from test framework |

---

## Forbidden Patterns

| Pattern          | Why                   | Alternative              |
| ---------------- | --------------------- | ------------------------ |
| `var`            | Hoisting, leaking     | `const`, `let`           |
| `==`, `!=`       | Implicit coercion     | `===`, `!==`             |
| `eval()`         | Security risk         | Safe alternatives        |
| `with`           | Scope confusion       | Explicit references      |
| `arguments`      | Not a real array      | Rest params `...args`    |
| `console.log`    | Unstructured          | Logger in production     |
| `new Function()` | Same as eval          | Direct code              |
| Implicit globals | Missing `const`/`let` | Always declare variables |

| Exception      | Allowed When               |
| -------------- | -------------------------- |
| `console.log`  | Development / debugging    |
| `==` with null | `value == null` checks both null and undefined |

---

## Pre-Commit Checklist

```bash
npx eslint .
npx prettier --check .
npm test
```

| Check                       | Status |
| --------------------------- | ------ |
| ESLint passes               | [ ]    |
| Prettier passes             | [ ]    |
| Tests pass                  | [ ]    |
| No `var` declarations       | [ ]    |
| No `==` (except null check) | [ ]    |
| Public functions have JSDoc | [ ]    |
| `"use strict"` or ESM      | [ ]    |
