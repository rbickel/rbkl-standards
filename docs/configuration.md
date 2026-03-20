# Configuration Management

← [Back to README](../README.md)

## Approved Library

RBKL uses **`viper`** for all configuration management.

```bash
go get github.com/spf13/viper
```

Viper provides a unified interface over environment variables, config files, and remote configuration stores (Azure App Configuration).

---

## Configuration Sources (Priority Order)

Viper reads configuration from these sources, in descending priority:

1. **Environment variables** — Highest priority. Used in all deployed environments.
2. **`.env` file** — For local development only. Never committed to git (add to `.gitignore`).
3. **`config.yaml` file** — Non-sensitive default values. Safe to commit.
4. **Default values** — Set in code via `viper.SetDefault()`.

---

## Config Struct Pattern

Define a strongly-typed config struct. Viper unmarshals into it at startup.

```go
package config

import (
    "fmt"
    "log/slog"
    "strings"
    "time"

    "github.com/spf13/viper"
)

type Config struct {
    // Server
    Port int    `mapstructure:"PORT"`
    Env  string `mapstructure:"ENV"` // local | dev | staging | production

    // Database
    DatabaseURL             string        `mapstructure:"DATABASE_URL"`
    DatabaseMaxConns        int32         `mapstructure:"DATABASE_MAX_CONNS"`
    DatabaseMaxConnLifetime time.Duration `mapstructure:"DATABASE_MAX_CONN_LIFETIME"`

    // Auth
    AzureTenantID string `mapstructure:"AZURE_TENANT_ID"`
    AzureClientID string `mapstructure:"AZURE_CLIENT_ID"`

    // CORS
    AllowedOrigins []string `mapstructure:"ALLOWED_ORIGINS"`

    // Logging
    LogLevel slog.Level `mapstructure:"LOG_LEVEL"`
}

func Load() (*Config, error) {
    v := viper.New()

    // ── Defaults ──────────────────────────────────────────────────────────────
    v.SetDefault("PORT", 8080)
    v.SetDefault("ENV", "local")
    v.SetDefault("LOG_LEVEL", "INFO")
    v.SetDefault("DATABASE_MAX_CONNS", 10)
    v.SetDefault("DATABASE_MAX_CONN_LIFETIME", "1h")
    v.SetDefault("ALLOWED_ORIGINS", []string{"http://localhost:3000"})

    // ── Config file (optional) ────────────────────────────────────────────────
    v.SetConfigName("config")
    v.SetConfigType("yaml")
    v.AddConfigPath(".")
    v.AddConfigPath("./config")
    if err := v.ReadInConfig(); err != nil {
        if _, ok := err.(viper.ConfigFileNotFoundError); !ok {
            return nil, fmt.Errorf("read config file: %w", err)
        }
    }

    // ── .env file (local dev only) ────────────────────────────────────────────
    v.SetConfigFile(".env")
    _ = v.MergeInConfig() // Ignore error if .env doesn't exist

    // ── Environment variables ─────────────────────────────────────────────────
    v.AutomaticEnv()
    v.SetEnvKeyReplacer(strings.NewReplacer(".", "_"))

    // ── Unmarshal into struct ─────────────────────────────────────────────────
    var cfg Config
    if err := v.Unmarshal(&cfg); err != nil {
        return nil, fmt.Errorf("unmarshal config: %w", err)
    }

    if err := cfg.validate(); err != nil {
        return nil, fmt.Errorf("invalid config: %w", err)
    }

    return &cfg, nil
}
```

---

## Config Validation

Validate all required fields at startup. The application must fail fast with a clear error message rather than crashing later with a cryptic nil pointer dereference.

```go
func (c *Config) validate() error {
    var errs []string

    if c.DatabaseURL == "" {
        errs = append(errs, "DATABASE_URL is required")
    }
    if c.AzureTenantID == "" {
        errs = append(errs, "AZURE_TENANT_ID is required")
    }
    if c.AzureClientID == "" {
        errs = append(errs, "AZURE_CLIENT_ID is required")
    }
    if c.Port < 1 || c.Port > 65535 {
        errs = append(errs, "PORT must be between 1 and 65535")
    }
    validEnvs := map[string]bool{"local": true, "dev": true, "staging": true, "production": true}
    if !validEnvs[c.Env] {
        errs = append(errs, fmt.Sprintf("ENV must be one of: local, dev, staging, production (got %q)", c.Env))
    }

    if len(errs) > 0 {
        return fmt.Errorf("configuration errors:\n  - %s", strings.Join(errs, "\n  - "))
    }
    return nil
}
```

---

## Environment Variable Naming

| Convention | Example |
|---|---|
| All uppercase, underscore-separated | `DATABASE_URL`, `AZURE_CLIENT_ID` |
| Prefix with service name in multi-service environments | `USERS_DATABASE_URL` |
| Duration values use Go duration format | `REQUEST_TIMEOUT=30s` |
| Boolean values: `true` / `false` | `FEATURE_NEW_DASHBOARD=true` |
| List values: comma-separated | `ALLOWED_ORIGINS=https://app.rbkl.com,https://admin.rbkl.com` |

---

## Secrets Management

**Rule: No secrets in config files, environment variable files (`.env`), or source code.** Secrets are stored in **Azure Key Vault** and injected at runtime.

### Options for Secret Injection

#### Option A: CSI Secret Store Driver (Kubernetes)
The recommended approach for Kubernetes deployments. Secrets are mounted as files or environment variables by the CSI driver. No application-level Azure SDK required.

```yaml
# kubernetes/deployment.yaml
env:
  - name: DATABASE_URL
    valueFrom:
      secretKeyRef:
        name: my-service-secrets
        key: database-url
```

#### Option B: Managed Identity + Azure SDK (non-Kubernetes)
For Azure App Service or Azure Functions:

```go
import (
    "github.com/Azure/azure-sdk-for-go/sdk/azidentity"
    "github.com/Azure/azure-sdk-for-go/sdk/security/keyvault/azsecrets"
)

func loadSecret(ctx context.Context, vaultURL, secretName string) (string, error) {
    cred, err := azidentity.NewDefaultAzureCredential(nil)
    if err != nil {
        return "", fmt.Errorf("get credential: %w", err)
    }

    client, err := azsecrets.NewClient(vaultURL, cred, nil)
    if err != nil {
        return "", fmt.Errorf("create secrets client: %w", err)
    }

    resp, err := client.GetSecret(ctx, secretName, "", nil)
    if err != nil {
        return "", fmt.Errorf("get secret %q: %w", secretName, err)
    }

    return *resp.Value, nil
}
```

---

## Feature Flags

For runtime feature toggles, use Azure App Configuration with feature flag support. Do not use environment variables for feature flags that need to be toggled without redeployment.

```go
// Simple boolean feature flag check
func (c *Config) IsEnabled(feature string) bool {
    return viper.GetBool("feature." + feature)
}

// Usage
if cfg.IsEnabled("new_dashboard") {
    handler.ServeNewDashboard(w, r)
} else {
    handler.ServeLegacyDashboard(w, r)
}
```

---

## `.env.example` File

Every repository **must** include a `.env.example` file with all required variables documented (but no real values). This file **is** committed to git.

```dotenv
# Server
PORT=8080
ENV=local

# Database
DATABASE_URL=postgres://user:password@localhost:5432/mydb?sslmode=disable
DATABASE_MAX_CONNS=10

# Azure AD
AZURE_TENANT_ID=your-tenant-id-here
AZURE_CLIENT_ID=your-client-id-here

# CORS
ALLOWED_ORIGINS=http://localhost:3000

# Logging
LOG_LEVEL=DEBUG
```

---

## Prohibited Patterns

```go
// ❌ Hardcoded secrets
const dbPassword = "super-secret-password"

// ❌ Reading secrets from files in production
ioutil.ReadFile("/etc/secrets/db-password")

// ❌ Environment variables directly (bypass viper)
os.Getenv("DATABASE_URL") // Use cfg.DatabaseURL instead

// ❌ Config struct with no validation
func Load() Config {
    return Config{Port: viper.GetInt("PORT")} // No validation!
}
```

---

_[← Database](database.md) · [Error Handling →](error-handling.md)_
