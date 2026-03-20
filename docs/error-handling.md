# Error Handling

← [Back to README](../README.md)

## Philosophy

In Go, errors are values — not exceptions. RBKL treats error handling as a first-class design concern, not an afterthought. Clear, wrapped errors enable fast debugging, accurate alerting, and reliable HTTP responses.

---

## Sentinel Errors

Define sentinel errors for expected failure modes in the `domain` package. These are the only errors that callers should check by identity.

```go
package domain

import "errors"

// User-facing domain errors
var (
    ErrNotFound          = errors.New("not found")
    ErrConflict          = errors.New("conflict")
    ErrUnauthorized      = errors.New("unauthorized")
    ErrForbidden         = errors.New("forbidden")
    ErrValidation        = errors.New("validation error")
    ErrDependencyExists  = errors.New("dependency exists")
)
```

### Checking Errors

Always use `errors.Is` and `errors.As` — never `==` comparison after wrapping.

```go
// ✅ Good — works through wrapping chains
if errors.Is(err, domain.ErrNotFound) {
    return http.StatusNotFound
}

// ❌ Bad — breaks after any wrapping
if err == domain.ErrNotFound {
    return http.StatusNotFound
}
```

---

## Error Wrapping

Wrap errors at every layer boundary to build a call chain. Use `%w` (not `%v`) to preserve unwrappability.

```go
// ✅ Good — wraps with context, preserves unwrappability
user, err := s.repo.GetUser(ctx, id)
if err != nil {
    return nil, fmt.Errorf("UserService.Get id=%s: %w", id, err)
}

// ❌ Bad — loses the original error type
return nil, fmt.Errorf("failed to get user: %v", err)

// ❌ Bad — no context about where it came from
return nil, err
```

### Wrapping Conventions

| Layer | Prefix Format | Example |
|---|---|---|
| Handler | `{Method} {Path}:` | `GET /v1/users:` |
| Service | `{ServiceName}.{Method}:` | `UserService.Create:` |
| Repository | `{repo}.{Method} {key}={value}:` | `userRepo.Get id=abc:` |

---

## Structured Error Types

For errors that carry structured data (validation errors, multi-field errors), define typed errors:

```go
package domain

// ValidationError carries field-level validation failures.
type ValidationError struct {
    Field   string
    Message string
}

func (e *ValidationError) Error() string {
    return fmt.Sprintf("validation failed: %s — %s", e.Field, e.Message)
}

// MultiValidationError aggregates multiple field errors.
type MultiValidationError struct {
    Errors []*ValidationError
}

func (e *MultiValidationError) Error() string {
    msgs := make([]string, len(e.Errors))
    for i, err := range e.Errors {
        msgs[i] = err.Error()
    }
    return strings.Join(msgs, "; ")
}

func (e *MultiValidationError) Unwrap() error {
    return ErrValidation
}
```

---

## HTTP Error Responses (RFC 7807)

All HTTP error responses **must** follow [RFC 7807 Problem Details](https://datatracker.ietf.org/doc/html/rfc7807).

```go
package transport

// Problem is an RFC 7807 problem details object.
type Problem struct {
    Type     string `json:"type,omitempty"`
    Title    string `json:"title"`
    Status   int    `json:"status"`
    Detail   string `json:"detail,omitempty"`
    Instance string `json:"instance,omitempty"`
    // Additional fields for validation errors
    Errors   []FieldError `json:"errors,omitempty"`
}

type FieldError struct {
    Field   string `json:"field"`
    Message string `json:"message"`
}

func ToProblemDetails(r *http.Request, err error) Problem {
    instance := r.URL.Path

    // Validation error — 422
    var multiErr *domain.MultiValidationError
    if errors.As(err, &multiErr) {
        fieldErrors := make([]FieldError, len(multiErr.Errors))
        for i, e := range multiErr.Errors {
            fieldErrors[i] = FieldError{Field: e.Field, Message: e.Message}
        }
        return Problem{
            Title:    "Validation Error",
            Status:   http.StatusUnprocessableEntity,
            Detail:   err.Error(),
            Instance: instance,
            Errors:   fieldErrors,
        }
    }

    // Domain error mapping
    switch {
    case errors.Is(err, domain.ErrNotFound):
        return Problem{Title: "Not Found", Status: http.StatusNotFound, Instance: instance}
    case errors.Is(err, domain.ErrConflict):
        return Problem{Title: "Conflict", Status: http.StatusConflict, Detail: err.Error(), Instance: instance}
    case errors.Is(err, domain.ErrUnauthorized):
        return Problem{Title: "Unauthorized", Status: http.StatusUnauthorized, Instance: instance}
    case errors.Is(err, domain.ErrForbidden):
        return Problem{Title: "Forbidden", Status: http.StatusForbidden, Instance: instance}
    case errors.Is(err, domain.ErrValidation):
        return Problem{Title: "Unprocessable Entity", Status: http.StatusUnprocessableEntity, Detail: err.Error(), Instance: instance}
    default:
        // Internal server error — do NOT expose internal details
        return Problem{Title: "Internal Server Error", Status: http.StatusInternalServerError, Instance: instance}
    }
}
```

### Example Response

```http
HTTP/1.1 422 Unprocessable Entity
Content-Type: application/problem+json

{
  "title": "Validation Error",
  "status": 422,
  "detail": "validation failed: email — must be a valid email address",
  "instance": "/v1/users",
  "errors": [
    {"field": "email", "message": "must be a valid email address"},
    {"field": "name", "message": "must be at least 2 characters"}
  ]
}
```

**Rules:**
- Never expose internal error messages (stack traces, SQL errors) in HTTP responses.
- Always use `application/problem+json` content type for error responses.
- Distinguish between client errors (4xx) and server errors (5xx) accurately.

---

## Logging Rules

| Error Type | Log Level | Include Error? |
|---|---|---|
| Client error (4xx) | `INFO` or `DEBUG` | Yes, at debug |
| Server error (5xx) | `ERROR` | Yes, always |
| Panic (recovered) | `ERROR` | Yes, with stack trace |
| Expected domain errors (not found, conflict) | `DEBUG` | No |

```go
func (h *Handler) writeError(w http.ResponseWriter, r *http.Request, err error) {
    problem := transport.ToProblemDetails(r, err)
    
    if problem.Status >= 500 {
        log.FromContext(r.Context()).ErrorContext(r.Context(), "internal server error",
            "error", err,
            "path", r.URL.Path,
            "method", r.Method,
        )
    } else if problem.Status >= 400 {
        log.FromContext(r.Context()).DebugContext(r.Context(), "client error",
            "error", err,
            "status", problem.Status,
        )
    }

    w.Header().Set("Content-Type", "application/problem+json")
    w.WriteHeader(problem.Status)
    _ = json.NewEncoder(w).Encode(problem)
}
```

---

## Panics

Panics are **never** used for expected error conditions. They are reserved for truly unrecoverable programming errors (e.g., a nil map dereference in `init()` setup code).

The `middleware.Recoverer` middleware catches any panic that escapes a handler, logs the stack trace, and returns a `500` response. This is a safety net — not an error-handling strategy.

```go
// ❌ Never use panic for domain errors
if id == "" {
    panic("id is required") // Wrong!
}

// ✅ Return an error
if id == "" {
    return nil, &domain.ValidationError{Field: "id", Message: "is required"}
}
```

---

_[← Configuration](configuration.md) · [Testing →](testing.md)_
