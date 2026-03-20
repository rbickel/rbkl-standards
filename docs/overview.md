# Overview & Philosophy

← [Back to README](../README.md)

## Mission

The RBKL standards wiki exists to answer one question for every engineer and AI coding agent working in the organization:

> "What is the right way to build this at RBKL?"

A shared baseline of standards reduces the cognitive overhead of every pull request review, keeps our attack surface consistent, and lets coding agents produce code that fits our stack without requiring manual correction.

---

## Primary Language: Go

RBKL standardizes on **Go (1.22+)** for all backend services, CLIs, and infrastructure tooling.

### Why Go?

| Property | Benefit |
|---|---|
| Static typing + compilation | Catches bugs before runtime; clear interfaces |
| Single binary output | Simple deployment — no runtime dependencies |
| First-class concurrency (`goroutines` / `channels`) | Efficient for HTTP services and data pipelines |
| Minimal standard library surface | Small, auditable dependency trees |
| Fast compile times | Developer and CI velocity |
| Strong ecosystem for cloud-native | Kubernetes, Prometheus, OpenTelemetry all written in Go |

> **Exceptions:** Data science / ML workloads use Python. Frontend applications use TypeScript. Both have separate standards documents.

---

## Core Engineering Principles

### 1. Explicit Over Implicit
Prefer code that states what it does clearly. Avoid magic — no `init()` side effects that change global state, no auto-registration patterns that obscure control flow.

```go
// ✅ Good — explicit wiring
router := chi.NewRouter()
router.Use(middleware.Logger)
router.Get("/health", handlers.Health)

// ❌ Bad — implicit registration via init()
func init() {
    globalRouter.Register("/health", handlers.Health)
}
```

### 2. Error Handling is a First-Class Concern
Every function that can fail returns an `error`. Errors are wrapped with context. They are never silently swallowed.

```go
// ✅ Good
user, err := db.GetUser(ctx, id)
if err != nil {
    return fmt.Errorf("getUser id=%s: %w", id, err)
}

// ❌ Bad
user, _ := db.GetUser(ctx, id)
```

### 3. Context Everywhere
The `context.Context` is the first argument of any function that performs I/O, blocking calls, or needs to propagate deadlines/cancellation.

```go
// ✅ Good
func (s *UserService) Get(ctx context.Context, id string) (*User, error)

// ❌ Bad — no context
func (s *UserService) Get(id string) (*User, error)
```

### 4. Small, Focused Packages
Packages should have a single, clear responsibility. Avoid packages named `util`, `common`, or `helpers`. If code belongs nowhere, it probably needs its own package.

### 5. Interfaces at the Boundary
Define interfaces in the package that **uses** them, not the package that **implements** them. Keep interfaces small (1–3 methods).

```go
// ✅ Good — defined in the consumer package
type UserStore interface {
    Get(ctx context.Context, id string) (*User, error)
    Save(ctx context.Context, u *User) error
}
```

### 6. No Global State
Global variables (including `sync.Once`-initialized singletons) make testing difficult and introduce hidden coupling. Pass dependencies explicitly through constructors.

### 7. Deterministic Builds
Pin all dependency versions in `go.sum`. Use Docker image digests in CI. Never use `latest` tags.

---

## When to Deviate

Standards are defaults, not laws. A deviation is acceptable when:

1. The standard creates a provable performance or reliability problem in a specific context.
2. A third-party integration requires a specific approach (e.g., vendor SDK only supports a particular auth method).
3. An experiment needs to validate an assumption before promoting to a standard.

**Process for deviations:**
1. Open a GitHub issue in this repository with the label `deviation-request`.
2. Describe the context, reason, and proposed alternative.
3. Get sign-off from the Platform Engineering team lead.
4. Document the deviation in the relevant service README.

---

## How This Document Is Maintained

- All changes go through a pull request with at least one Platform Engineering review.
- Standards are reviewed quarterly and updated as ecosystem best practices evolve.
- Each section owner is listed in [CODEOWNERS](../.github/CODEOWNERS) (if present).

---

_[← README](../README.md) · [Project Structure →](project-structure.md)_
