# Security

← [Back to README](../README.md)

## Principles

Security is built into every layer. RBKL adopts a **defense-in-depth** strategy: no single control is relied upon exclusively, and every service is treated as potentially compromised.

---

## Input Validation

All external input must be validated before use. Validation happens at the handler layer before data reaches the service layer.

### Approved Validation Library

Use `go-playground/validator` for struct-level validation:

```bash
go get github.com/go-playground/validator/v10
```

```go
package transport

import (
    "github.com/go-playground/validator/v10"
)

var validate = validator.New()

type CreateUserRequest struct {
    Email string `json:"email" validate:"required,email,max=254"`
    Name  string `json:"name"  validate:"required,min=2,max=100"`
    Role  string `json:"role"  validate:"required,oneof=admin editor viewer"`
}

func (h *Handler) CreateUser(w http.ResponseWriter, r *http.Request) {
    var req CreateUserRequest
    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        h.writeError(w, r, fmt.Errorf("%w: invalid JSON", domain.ErrValidation))
        return
    }

    if err := validate.Struct(req); err != nil {
        h.writeError(w, r, toValidationError(err))
        return
    }

    // Safe to use req now
}
```

### Validation Rules

| Input Type | Rules |
|---|---|
| Email | Max 254 chars, RFC 5322 format |
| Free text | Max length, strip leading/trailing whitespace |
| Integers | Range check (min/max) |
| UUIDs | Parse with `uuid.Parse()` — reject malformed |
| Enums | `oneof` validator — reject unknown values |
| URLs | Parse with `url.Parse()`, check scheme is `https` |
| File uploads | Check MIME type, max size, scan content |

---

## SQL Injection Prevention

RBKL uses `sqlc` for all database queries, which generates parameterized queries. Raw SQL interpolation is **never** allowed.

```go
// ✅ Parameterized — safe
pool.QueryRow(ctx, "SELECT id FROM users WHERE email = $1", email)

// ❌ String interpolation — SQL injection vulnerability
pool.QueryRow(ctx, "SELECT id FROM users WHERE email = '"+email+"'")
```

CI enforces a `sqlc vet` check that fails if any query uses unsafe constructs.

---

## Secrets Management

See [Configuration Management](configuration.md) for full details.

Summary:
- **No secrets in source code** — blocked by pre-commit hooks (see below).
- **No secrets in config files** — use Azure Key Vault or environment injection.
- **No secrets in logs** — use the `Redacted` type pattern.
- **No secrets in error messages** — returned to clients.

---

## Dependency Scanning

### `govulncheck`

Run `govulncheck` on every CI pipeline to detect known vulnerabilities in Go dependencies:

```bash
go install golang.org/x/vuln/cmd/govulncheck@latest
govulncheck ./...
```

CI fails if any vulnerability with a fix available is detected.

### Dependabot

Enable Dependabot in `.github/dependabot.yml`:

```yaml
version: 2
updates:
  - package-ecosystem: "gomod"
    directory: "/"
    schedule:
      interval: "weekly"
    open-pull-requests-limit: 10
    labels:
      - "dependencies"
```

---

## Pre-commit Hooks for Secrets Detection

Every repository includes a `.pre-commit-config.yaml` that runs `gitleaks` to prevent secrets from being committed:

```yaml
repos:
  - repo: https://github.com/gitleaks/gitleaks
    rev: v8.18.2
    hooks:
      - id: gitleaks
```

Install hooks on repository clone:
```bash
pip install pre-commit
pre-commit install
```

CI also runs `gitleaks` on every push as a non-bypassable check.

---

## OWASP Top 10 Mitigations

### A01: Broken Access Control
- JWT validation and RBAC middleware on every protected route (see [Authentication](authentication.md)).
- No security decisions based on client-supplied headers (`X-User-ID`, `X-Role`).
- Resources are owned — always filter by the authenticated user's tenant.

### A02: Cryptographic Failures
- All communication over TLS 1.2+. No HTTP (only HTTPS/H2).
- Passwords (if ever stored) hashed with `bcrypt` cost ≥ 12 or `argon2id`.
- No MD5 or SHA-1 for security-sensitive operations.
- Secrets always 256-bit or longer, generated with `crypto/rand`.

```go
// ✅ Correct random token generation
import "crypto/rand"
import "encoding/hex"

func generateToken(n int) (string, error) {
    b := make([]byte, n)
    if _, err := rand.Read(b); err != nil {
        return "", err
    }
    return hex.EncodeToString(b), nil
}

// ❌ Insecure — predictable
token := fmt.Sprintf("%d", math_rand.Int63())
```

### A03: Injection
- All SQL via parameterized queries (enforced by `sqlc`).
- HTML output escaped via `html/template` (never `text/template` for HTML).
- Shell command execution forbidden. If required, use explicit argument lists (no shell expansion).

### A04: Insecure Design
- Threat model documented for every service in `docs/threat-model.md`.
- No debug endpoints exposed in production (`/debug/pprof`, `/metrics` behind auth).

### A05: Security Misconfiguration
- No default credentials. All credentials are rotated secrets.
- Detailed error messages only in non-production environments.
- HTTP security headers on all responses (see below).

### A06: Vulnerable & Outdated Components
- `govulncheck` and Dependabot (see above).
- Base Docker images use specific digests, updated monthly.

### A07: Identification and Authentication Failures
- Short-lived JWT tokens (max 1 hour).
- Token refresh via OIDC refresh tokens — not long-lived access tokens.
- No credentials in URLs (query parameters or paths).

### A08: Software and Data Integrity Failures
- All Go module downloads verified against `go.sum`.
- Docker images signed and verified in the deployment pipeline.
- CI artifacts are not modifiable after production.

### A09: Security Logging and Monitoring Failures
- All authentication failures logged (see [Logging](logging.md)).
- Alerting configured for repeated 401/403 errors (potential brute force).
- Audit log for all write operations on sensitive resources.

### A10: Server-Side Request Forgery (SSRF)
- Never make HTTP requests to URLs supplied directly by clients.
- If required, validate against an allowlist of approved domains.
- Block requests to private IP ranges (`10.x`, `172.16.x`, `192.168.x`, `169.254.x`).

---

## HTTP Security Headers

Apply these headers to all HTTP responses via middleware:

```go
func SecurityHeaders(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        w.Header().Set("X-Content-Type-Options", "nosniff")
        w.Header().Set("X-Frame-Options", "DENY")
        w.Header().Set("X-XSS-Protection", "0")            // Disabled — rely on CSP
        w.Header().Set("Referrer-Policy", "strict-origin-when-cross-origin")
        w.Header().Set("Content-Security-Policy", "default-src 'none'")
        w.Header().Set("Strict-Transport-Security", "max-age=31536000; includeSubDomains")
        w.Header().Set("Permissions-Policy", "camera=(), microphone=(), geolocation=()")
        next.ServeHTTP(w, r)
    })
}
```

---

## Rate Limiting

All public-facing APIs must implement rate limiting. Use token bucket algorithm via `golang.org/x/time/rate` or a Redis-backed distributed rate limiter for multi-instance deployments.

```go
import "golang.org/x/time/rate"

// Per-IP rate limiter: 10 requests/second, burst of 20
limiter := rate.NewLimiter(rate.Limit(10), 20)

func RateLimit(limiter *rate.Limiter) func(http.Handler) http.Handler {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            if !limiter.Allow() {
                w.Header().Set("Retry-After", "1")
                http.Error(w, `{"title":"Too Many Requests","status":429}`, http.StatusTooManyRequests)
                return
            }
            next.ServeHTTP(w, r)
        })
    }
}
```

---

_[← API Design](api-design.md) · [Dependency Management →](dependency-management.md)_
