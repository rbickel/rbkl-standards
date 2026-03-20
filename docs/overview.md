# Overview & Philosophy

← [Back to README](../README.md)

## Mission

The RBKL standards wiki exists to answer one question for every engineer and AI coding agent working in the organization:

> "What is the right way to build this at RBKL?"

A shared baseline of standards reduces the cognitive overhead of every pull request review, keeps our attack surface consistent, and lets coding agents produce code that fits our stack without requiring manual correction.

---

## Primary Languages: TypeScript on Node.js (backend) + React (frontend)

RBKL standardizes on **TypeScript** for all backend services (Node.js 20 LTS) and all frontend applications (React 18). JavaScript without TypeScript is not permitted in production code.

### Why TypeScript + Node.js for the backend?

| Property | Benefit |
|---|---|
| Static typing | Catches bugs at compile time; self-documenting interfaces |
| Shared language with frontend | Engineers move between layers with minimal context switch |
| NPM ecosystem | Largest package ecosystem; vast Azure SDK and tooling support |
| Async I/O by default | Efficient for high-concurrency HTTP and event-driven workloads |
| Fastify + Pino | Production-grade, high-performance HTTP framework with structured logging |
| Strong Azure SDK support | First-class SDKs for Entra ID, Key Vault, Service Bus, etc. |

### Why React for the frontend?

| Property | Benefit |
|---|---|
| Industry standard | Largest hiring pool; most tooling support |
| Component model | Composable, testable UI with React Testing Library |
| Vite build tooling | Fast HMR, tree-shaking, optimized production builds |
| Strong TypeScript integration | Full type safety from API contract to rendered component |

> **Exceptions:** Data science / ML workloads use Python. Infrastructure tooling may use Go. Both have separate standards documents.

---

## Core Engineering Principles

### 1. TypeScript Everywhere — No `any`
The TypeScript compiler is a tool for correctness. Disabling it with `any`, `@ts-ignore`, or `as unknown as X` defeats its purpose. Use proper type narrowing, generics, and discriminated unions.

```typescript
// ✅ Good — proper type narrowing
function processResult(result: User | null): string {
  if (result === null) return "not found";
  return result.name;
}

// ❌ Bad — opts out of type safety
function processResult(result: any): string {
  return result.name;
}
```

### 2. Async/Await, Never Raw Callbacks
All asynchronous code uses `async`/`await`. Raw callbacks and `.then()/.catch()` chains are only acceptable when wrapping a legacy callback-only API.

```typescript
// ✅ Good
const user = await userService.get(userId);

// ❌ Bad
userService.get(userId).then(user => {
  // ...
}).catch(err => {
  // ...
});
```

### 3. Fail Fast — Validate at the Boundary
All external input (HTTP requests, environment variables, database results) is validated with **Zod** at the entry point. Validated data is typed — unvalidated data is `unknown`.

```typescript
// ✅ Good — parse at the boundary; the rest of the code is type-safe
const body = CreateUserSchema.parse(req.body);
await userService.create(body);

// ❌ Bad — using unvalidated data directly
await userService.create(req.body as CreateUserDto);
```

### 4. Dependency Injection, Not Singletons
Services and repositories are created once and injected as constructor arguments. Module-level singleton instances make testing difficult and introduce hidden coupling.

```typescript
// ✅ Good — explicit injection
class UserService {
  constructor(
    private readonly db: UserRepository,
    private readonly logger: Logger,
  ) {}
}

// ❌ Bad — global singleton
import { db } from "../db"; // Hidden dependency
```

### 5. Immutability by Default
Prefer `const` over `let`. Use `readonly` on interface properties and class fields. Use spread operators or `structuredClone` instead of mutation.

### 6. No Implicit Side Effects in Modules
Module top-level code must not produce side effects (network calls, DB connections, file reads) when the module is imported. Move side effects into explicitly called `initialize()` or `connect()` functions.

### 7. Pure Functions and Testability
Business logic lives in pure functions or classes with injected dependencies. Functions that mix I/O with logic are hard to test. Separate them.

---

## When to Deviate

Standards are defaults, not laws. A deviation is acceptable when:

1. The standard creates a provable performance or reliability problem.
2. A third-party integration requires a specific approach.
3. An experiment needs to validate an assumption before promoting to a standard.

**Process for deviations:**
1. Open a GitHub issue in this repository with the label `deviation-request`.
2. Describe the context, reason, and proposed alternative.
3. Get sign-off from the Platform Engineering team lead.
4. Document the deviation in the relevant service/app README.

---

## How This Document Is Maintained

- All changes go through a pull request with at least one Platform Engineering review.
- Standards are reviewed quarterly and updated as ecosystem best practices evolve.
- Each section owner is listed in [CODEOWNERS](../.github/CODEOWNERS) (if present).

---

_[← README](../README.md) · [Project Structure →](project-structure.md)_

