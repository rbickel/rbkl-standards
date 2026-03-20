# Authentication & Authorization

← [Back to README](../README.md)

## Identity Architecture

RBKL uses **Azure Active Directory (Entra ID)** as the single identity provider. All authentication flows use the **OpenID Connect (OIDC)** protocol. Services validate **JWT access tokens** signed by Entra ID.

```
Client → Azure AD (OIDC) → JWT access token → RBKL Service
                                                     ↓
                                             Validate JWT signature
                                             Validate claims (iss, aud, exp)
                                             Extract roles → enforce RBAC
```

---

## Required Libraries

| Library | Purpose | Version |
|---|---|---|
| `golang-jwt/jwt/v5` | JWT parsing and validation | v5 |
| `MicahParks/keyfunc` | JWKS key set fetching and caching | v3 |

```bash
go get github.com/golang-jwt/jwt/v5
go get github.com/MicahParks/keyfunc/v3
```

---

## JWT Validation

### Standard Claims Validation

Every service **must** validate these claims on every authenticated request:

| Claim | Validation |
|---|---|
| `iss` (issuer) | Must equal `https://login.microsoftonline.com/{tenantID}/v2.0` |
| `aud` (audience) | Must equal the service's own Application (client) ID |
| `exp` (expiry) | Must be in the future (automatically checked by `golang-jwt`) |
| `nbf` (not before) | Must be in the past (automatically checked by `golang-jwt`) |
| `tid` (tenant ID) | Must match the expected Azure AD tenant |

### Implementation

```go
package auth

import (
    "context"
    "fmt"
    "net/http"
    "strings"
    "time"

    "github.com/MicahParks/keyfunc/v3"
    "github.com/golang-jwt/jwt/v5"
)

type Claims struct {
    jwt.RegisteredClaims
    TenantID string   `json:"tid"`
    Roles    []string `json:"roles"`
    Name     string   `json:"name"`
    Email    string   `json:"preferred_username"`
    ObjectID string   `json:"oid"`
}

type Validator struct {
    jwks      keyfunc.Keyfunc
    issuer    string
    audience  string
    tenantID  string
}

func NewValidator(ctx context.Context, cfg *Config) (*Validator, error) {
    // Automatically fetches and caches the JWKS from Azure AD.
    // Keys are refreshed in the background before expiry.
    jwks, err := keyfunc.NewDefaultCtx(ctx, []string{
        fmt.Sprintf("https://login.microsoftonline.com/%s/discovery/v2.0/keys", cfg.TenantID),
    })
    if err != nil {
        return nil, fmt.Errorf("failed to fetch JWKS: %w", err)
    }

    return &Validator{
        jwks:     jwks,
        issuer:   fmt.Sprintf("https://login.microsoftonline.com/%s/v2.0", cfg.TenantID),
        audience: cfg.ClientID,
        tenantID: cfg.TenantID,
    }, nil
}

func (v *Validator) ValidateToken(tokenString string) (*Claims, error) {
    var claims Claims

    _, err := jwt.ParseWithClaims(tokenString, &claims,
        v.jwks.Keyfunc,
        jwt.WithIssuer(v.issuer),
        jwt.WithAudience(v.audience),
        jwt.WithExpirationRequired(),
        jwt.WithLeeway(30*time.Second),
    )
    if err != nil {
        return nil, fmt.Errorf("invalid token: %w", err)
    }

    if claims.TenantID != v.tenantID {
        return nil, fmt.Errorf("invalid tenant ID in token")
    }

    return &claims, nil
}
```

### Authentication Middleware

```go
package middleware

import (
    "net/http"
    "strings"

    "github.com/rbkl/my-service/internal/auth"
)

type contextKey string

const ClaimsKey contextKey = "claims"

func Authenticate(validator *auth.Validator) func(http.Handler) http.Handler {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            authHeader := r.Header.Get("Authorization")
            if !strings.HasPrefix(authHeader, "Bearer ") {
                http.Error(w, `{"title":"Unauthorized","status":401}`, http.StatusUnauthorized)
                return
            }

            token := strings.TrimPrefix(authHeader, "Bearer ")
            claims, err := validator.ValidateToken(token)
            if err != nil {
                // Log at INFO — this is a client error, not a server error
                log.FromContext(r.Context()).Info("authentication failed", "error", err)
                http.Error(w, `{"title":"Unauthorized","status":401}`, http.StatusUnauthorized)
                return
            }

            ctx := context.WithValue(r.Context(), ClaimsKey, claims)
            next.ServeHTTP(w, r.WithContext(ctx))
        })
    }
}

// ClaimsFromContext retrieves the validated JWT claims from the request context.
func ClaimsFromContext(ctx context.Context) (*auth.Claims, bool) {
    claims, ok := ctx.Value(ClaimsKey).(*auth.Claims)
    return claims, ok
}
```

---

## Role-Based Access Control (RBAC)

### Role Definition

Roles are defined in Azure AD as **App Roles** on the service's App Registration and assigned to users or groups. They appear in the JWT `roles` claim.

**Naming convention:** `{service}.{resource}.{action}` in `PascalCase`

```
UserService.Users.Read
UserService.Users.Write
UserService.Users.Admin
```

### RBAC Middleware

```go
func RequireRole(roles ...string) func(http.Handler) http.Handler {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            claims, ok := ClaimsFromContext(r.Context())
            if !ok {
                http.Error(w, `{"title":"Forbidden","status":403}`, http.StatusForbidden)
                return
            }

            for _, required := range roles {
                for _, assigned := range claims.Roles {
                    if assigned == required {
                        next.ServeHTTP(w, r)
                        return
                    }
                }
            }

            http.Error(w, `{"title":"Forbidden","status":403}`, http.StatusForbidden)
        })
    }
}
```

### Usage in Router

```go
r.Route("/v1/users", func(r chi.Router) {
    r.Use(middleware.Authenticate(validator))
    
    // Any authenticated user can list users
    r.Get("/", h.ListUsers)
    
    // Only users with Write role can create
    r.With(middleware.RequireRole("UserService.Users.Write")).Post("/", h.CreateUser)
    
    // Only admins can delete
    r.With(middleware.RequireRole("UserService.Users.Admin")).Delete("/{id}", h.DeleteUser)
})
```

---

## Service-to-Service Authentication

For internal service-to-service communication, use **Azure Managed Identity** and **workload identity federation** — never shared secrets or static API keys.

### Pattern: Acquire Token with Managed Identity

```go
import "github.com/Azure/azure-sdk-for-go/sdk/azidentity"

func newCredential() (azcore.TokenCredential, error) {
    // In production: uses the pod/VM managed identity automatically
    // Locally: falls back to az CLI credentials
    return azidentity.NewDefaultAzureCredential(nil)
}
```

The consuming service acquires a token for the target service's `clientID` and sends it as a `Bearer` token. The target service validates it the same way as user tokens.

---

## Forbidden Patterns

The following patterns are **never** acceptable:

```go
// ❌ Skip token validation — even in dev/staging environments
if cfg.Env == "dev" { next.ServeHTTP(w, r); return }

// ❌ Static API keys stored in code or config files
apiKey := "sk-hardcoded-secret-abc123"

// ❌ Basic authentication (username + password over HTTP)
r.Header.Set("Authorization", "Basic "+base64Encode(user+":"+pass))

// ❌ Trust user-supplied claims without validation
userID := r.Header.Get("X-User-ID") // Anyone can set this header

// ❌ Disable TLS certificate verification
http.Transport{TLSClientConfig: &tls.Config{InsecureSkipVerify: true}}
```

---

## Local Development

For local development, use a test Azure AD application registration or the [Azure AD Local Emulator](https://learn.microsoft.com/en-us/azure/active-directory/develop/). Never disable auth locally — generate real short-lived tokens.

A helper script `scripts/get-dev-token.sh` should be provided in every service to acquire a development token:

```bash
#!/usr/bin/env bash
# Acquire a dev token using az CLI
az account get-access-token \
  --resource "$CLIENT_ID" \
  --query accessToken \
  --output tsv
```

---

_[← Logging](logging.md) · [Database →](database.md)_
