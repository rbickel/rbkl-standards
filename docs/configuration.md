# Configuration Management

← [Back to README](../README.md)

## Approach

RBKL uses **environment variables** as the configuration source, validated with **Zod** at startup. `dotenv` loads variables from `.env` files in non-production environments.

```bash
pnpm add zod dotenv
```

---

## Configuration Sources (Priority Order)

1. **Environment variables** — Highest priority. Used in all deployed environments.
2. **`.env.local`** — Developer-specific overrides. Never committed to git.
3. **`.env`** — Shared non-sensitive defaults for local development. Safe to commit.
4. **`.env.test`** — Test-specific values loaded by Vitest. Committed to git (no secrets).

---

## Config Schema with Zod

Define a Zod schema that parses and validates `process.env` at application startup. The application **must fail fast** with a clear error if any required variable is missing or invalid.

```typescript
// apps/api/src/config/index.ts
import { z } from "zod";
import { config as loadDotenv } from "dotenv";

// Load .env files in non-production environments
if (process.env.NODE_ENV !== "production") {
  loadDotenv({ path: ".env.local", override: true });
  loadDotenv({ path: ".env" });
}

const ConfigSchema = z.object({
  // Server
  NODE_ENV: z.enum(["local", "development", "test", "staging", "production"]).default("local"),
  PORT: z.coerce.number().int().min(1).max(65535).default(3000),
  LOG_LEVEL: z.enum(["trace", "debug", "info", "warn", "error", "fatal"]).default("info"),
  SERVICE_NAME: z.string().min(1),

  // CORS
  ALLOWED_ORIGINS: z
    .string()
    .transform((val) => val.split(",").map((s) => s.trim()))
    .default("http://localhost:5173"),

  // Azure AD
  AZURE_TENANT_ID: z.string().uuid(),
  AZURE_CLIENT_ID: z.string().uuid(),

  // Database
  DATABASE_URL: z.string().url(),

  // Optional features
  FEATURE_NEW_DASHBOARD: z.coerce.boolean().default(false),
});

export type Config = z.infer<typeof ConfigSchema>;

export function loadConfig(): Config {
  const result = ConfigSchema.safeParse(process.env);

  if (!result.success) {
    const formatted = result.error.format();
    console.error("❌ Invalid configuration:");
    console.error(JSON.stringify(formatted, null, 2));
    process.exit(1);
  }

  return result.data;
}
```

### Key Features

- `z.coerce.number()` / `z.coerce.boolean()` — automatically coerces string env vars to the correct type.
- `.transform()` — converts comma-separated strings to arrays.
- `.default()` — provides fallback values without requiring the variable to be set.
- `safeParse` + descriptive error + `process.exit(1)` — clear failure message at startup.

---

## Environment Variable Naming

| Convention | Example |
|---|---|
| All uppercase, underscore-separated | `DATABASE_URL`, `AZURE_CLIENT_ID` |
| Prefix service name in multi-service environments | `USERS_DATABASE_URL` |
| Booleans: `true` / `false` | `FEATURE_NEW_DASHBOARD=true` |
| Lists: comma-separated | `ALLOWED_ORIGINS=https://app.rbkl.com,https://admin.rbkl.com` |
| Durations: milliseconds (integers) | `REQUEST_TIMEOUT_MS=30000` |

---

## Frontend Configuration (Vite)

Vite exposes environment variables prefixed with `VITE_` to the browser bundle. All others are server-side only.

```typescript
// apps/web/src/lib/config.ts
import { z } from "zod";

const EnvSchema = z.object({
  VITE_API_BASE_URL: z.string().url(),
  VITE_AZURE_TENANT_ID: z.string().uuid(),
  VITE_AZURE_CLIENT_ID: z.string().uuid(),
  VITE_API_CLIENT_ID: z.string().uuid(),
});

// import.meta.env is the Vite equivalent of process.env
const parsed = EnvSchema.safeParse(import.meta.env);
if (!parsed.success) {
  throw new Error(`Invalid frontend config: ${parsed.error.message}`);
}

export const env = parsed.data;
```

**Never prefix a secret with `VITE_`** — anything prefixed `VITE_` is embedded in the JavaScript bundle and visible to all users.

---

## Secrets Management

**Rule: No secrets in config files, `.env` files (other than `.env.local`), or source code.** Secrets are stored in **Azure Key Vault** and injected at runtime.

### Options for Secret Injection

#### Option A: CSI Secret Store Driver (Kubernetes)

Recommended for Kubernetes. Secrets are mounted as environment variables by the CSI driver. No application-level Azure SDK required.

```yaml
# kubernetes/deployment.yaml
env:
  - name: DATABASE_URL
    valueFrom:
      secretKeyRef:
        name: my-service-secrets
        key: database-url
```

#### Option B: Azure SDK (non-Kubernetes)

For Azure App Service or Azure Container Apps:

```typescript
import { DefaultAzureCredential } from "@azure/identity";
import { SecretClient } from "@azure/keyvault-secrets";

async function loadSecrets(vaultUrl: string): Promise<void> {
  const credential = new DefaultAzureCredential();
  const client = new SecretClient(vaultUrl, credential);

  const dbUrl = await client.getSecret("database-url");
  process.env.DATABASE_URL = dbUrl.value!;
}

// Call before loadConfig()
await loadSecrets(process.env.KEY_VAULT_URL!);
```

---

## `.env.example` File

Every repository **must** include a `.env.example` file with all required variables (no real values). This file **is** committed to git.

```dotenv
# Server
NODE_ENV=local
PORT=3000
LOG_LEVEL=debug
SERVICE_NAME=user-service

# CORS
ALLOWED_ORIGINS=http://localhost:5173

# Azure AD
AZURE_TENANT_ID=your-tenant-id-here
AZURE_CLIENT_ID=your-client-id-here

# Database
DATABASE_URL=******localhost:5432/mydb

# Feature flags
FEATURE_NEW_DASHBOARD=false
```

---

## Prohibited Patterns

```typescript
// ❌ Hardcoded secrets
const dbPassword = "super-secret-password";

// ❌ Accessing process.env directly in business logic
const tenantId = process.env.AZURE_TENANT_ID; // Use injected config instead

// ❌ No validation — runtime crash with unhelpful message
const port = parseInt(process.env.PORT!); // What if PORT is "abc"?

// ❌ Committing .env files with real values
git add .env // Should be in .gitignore
```

---

_[← Database](database.md) · [Error Handling →](error-handling.md)_
