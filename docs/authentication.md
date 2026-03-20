# Authentication & Authorization

← [Back to README](../README.md)

## Identity Architecture

RBKL uses **Azure Active Directory (Entra ID)** as the single identity provider. All authentication flows use the **OpenID Connect (OIDC)** protocol. Services validate **JWT access tokens** signed by Entra ID.

```
Browser/Client → Azure AD (OIDC) → JWT access token → RBKL API
                                                           ↓
                                                   Validate JWT (jose)
                                                   Validate claims (iss, aud, exp)
                                                   Extract roles → enforce RBAC
```

---

## Required Libraries

| Library | Purpose | Version |
|---|---|---|
| `jose` | JWT parsing, JWKS fetching, OIDC discovery | v5 |
| `@fastify/jwt` | Fastify JWT plugin (wraps jose) | latest |

```bash
pnpm add jose @fastify/jwt
```

---

## JWT Validation

### Standard Claims Validation

Every service **must** validate these claims on every authenticated request:

| Claim | Validation |
|---|---|
| `iss` (issuer) | Must equal `https://login.microsoftonline.com/{tenantID}/v2.0` |
| `aud` (audience) | Must equal the service's own Application (client) ID |
| `exp` (expiry) | Must be in the future |
| `nbf` (not before) | Must be in the past |
| `tid` (tenant ID) | Must match the expected Azure AD tenant |

### Auth Plugin Implementation

```typescript
// apps/api/src/plugins/auth.ts
import fp from "fastify-plugin";
import { FastifyPluginAsync, FastifyRequest, FastifyReply } from "fastify";
import { createRemoteJWKSet, jwtVerify, JWTPayload } from "jose";
import type { Config } from "../config/index.js";

export interface JwtClaims extends JWTPayload {
  tid: string;            // Tenant ID
  oid: string;            // Object ID (user's unique ID in the tenant)
  roles?: string[];       // App roles assigned to the user
  name?: string;
  preferred_username?: string;
}

declare module "fastify" {
  interface FastifyInstance {
    authenticate: (request: FastifyRequest, reply: FastifyReply) => Promise<void>;
  }
  interface FastifyRequest {
    jwtClaims: JwtClaims;
  }
}

interface AuthPluginOptions {
  config: Config;
}

const authPlugin: FastifyPluginAsync<AuthPluginOptions> = fp(
  async (server, { config }) => {
    const JWKS = createRemoteJWKSet(
      new URL(
        `https://login.microsoftonline.com/${config.AZURE_TENANT_ID}/discovery/v2.0/keys`,
      ),
      { cooldownDuration: 300_000 }, // Refresh keys at most every 5 minutes
    );

    const expectedIssuer = `https://login.microsoftonline.com/${config.AZURE_TENANT_ID}/v2.0`;

    server.decorate(
      "authenticate",
      async (request: FastifyRequest, reply: FastifyReply) => {
        const authHeader = request.headers.authorization;
        if (!authHeader?.startsWith("Bearer ")) {
          return reply
            .status(401)
            .send({ title: "Unauthorized", status: 401 });
        }

        const token = authHeader.slice(7);
        try {
          const { payload } = await jwtVerify<JwtClaims>(token, JWKS, {
            issuer: expectedIssuer,
            audience: config.AZURE_CLIENT_ID,
            clockTolerance: 30, // seconds
          });

          if (payload.tid !== config.AZURE_TENANT_ID) {
            throw new Error("Invalid tenant ID");
          }

          request.jwtClaims = payload;
        } catch (err) {
          // Log at info — this is a client error
          request.log.info({ err }, "Authentication failed");
          return reply
            .status(401)
            .send({ title: "Unauthorized", status: 401 });
        }
      },
    );
  },
);

export { authPlugin };
```

---

## Role-Based Access Control (RBAC)

### Role Definition

Roles are defined in Azure AD as **App Roles** on the service's App Registration. They appear in the JWT `roles` claim.

**Naming convention:** `{Service}.{Resource}.{Action}` in `PascalCase`

```
UserService.Users.Read
UserService.Users.Write
UserService.Users.Admin
```

### RBAC Helper

```typescript
// apps/api/src/plugins/auth.ts (extend the plugin)
export function requireRole(...roles: string[]) {
  return async (request: FastifyRequest, reply: FastifyReply): Promise<void> => {
    const userRoles = request.jwtClaims?.roles ?? [];
    const hasRole = roles.some((role) => userRoles.includes(role));

    if (!hasRole) {
      return reply.status(403).send({ title: "Forbidden", status: 403 });
    }
  };
}
```

### Usage in Routes

```typescript
// apps/api/src/routes/users.ts
server.get("/", {
  onRequest: [server.authenticate],
}, listUsersHandler);

server.post("/", {
  onRequest: [server.authenticate, requireRole("UserService.Users.Write")],
}, createUserHandler);

server.delete("/:id", {
  onRequest: [server.authenticate, requireRole("UserService.Users.Admin")],
}, deleteUserHandler);
```

---

## Frontend Authentication (React)

Use the **MSAL React** library to handle authentication in React applications.

```bash
pnpm add @azure/msal-browser @azure/msal-react
```

### MSAL Configuration

```typescript
// apps/web/src/lib/auth/msal-config.ts
import { Configuration, PublicClientApplication } from "@azure/msal-browser";

const msalConfig: Configuration = {
  auth: {
    clientId: import.meta.env.VITE_AZURE_CLIENT_ID,
    authority: `https://login.microsoftonline.com/${import.meta.env.VITE_AZURE_TENANT_ID}`,
    redirectUri: window.location.origin,
  },
  cache: {
    cacheLocation: "sessionStorage", // Do not use localStorage for tokens
    storeAuthStateInCookie: false,
  },
};

export const msalInstance = new PublicClientApplication(msalConfig);
```

### Auth Provider Setup

```tsx
// apps/web/src/main.tsx
import { MsalProvider } from "@azure/msal-react";
import { msalInstance } from "./lib/auth/msal-config.js";

ReactDOM.createRoot(document.getElementById("root")!).render(
  <MsalProvider instance={msalInstance}>
    <App />
  </MsalProvider>,
);
```

### Protected Routes

```tsx
// apps/web/src/components/ProtectedRoute.tsx
import { useIsAuthenticated, useMsal } from "@azure/msal-react";
import { Navigate } from "react-router-dom";

export function ProtectedRoute({ children }: { children: React.ReactNode }) {
  const isAuthenticated = useIsAuthenticated();
  const { inProgress } = useMsal();

  if (inProgress !== "none") return <LoadingSpinner />;
  if (!isAuthenticated) return <Navigate to="/login" replace />;

  return <>{children}</>;
}
```

### Acquiring Tokens for API Calls

```typescript
// apps/web/src/lib/api/client.ts
import { msalInstance } from "../auth/msal-config.js";

export async function apiFetch(url: string, init?: RequestInit): Promise<Response> {
  const account = msalInstance.getActiveAccount();
  if (!account) throw new Error("No active account");

  const tokenResponse = await msalInstance.acquireTokenSilent({
    account,
    scopes: [`api://${import.meta.env.VITE_API_CLIENT_ID}/.default`],
  });

  return fetch(url, {
    ...init,
    headers: {
      ...init?.headers,
      Authorization: `Bearer ${tokenResponse.accessToken}`,
      "Content-Type": "application/json",
    },
  });
}
```

---

## Service-to-Service Authentication

For internal service-to-service calls, use **Azure Managed Identity** — never static API keys or secrets.

```typescript
import { DefaultAzureCredential } from "@azure/identity";

const credential = new DefaultAzureCredential();

async function getServiceToken(targetClientId: string): Promise<string> {
  const tokenResponse = await credential.getToken(
    `api://${targetClientId}/.default`,
  );
  return tokenResponse.token;
}
```

---

## Forbidden Patterns

```typescript
// ❌ Skip validation in any environment
if (process.env.NODE_ENV === "development") return next(); // Never bypass auth

// ❌ Static API keys in code or config files
const apiKey = "sk-hardcoded-secret-abc123";

// ❌ Trust client-supplied identity headers
const userId = req.headers["x-user-id"]; // Anyone can set this

// ❌ Store tokens in localStorage (XSS risk)
localStorage.setItem("access_token", token);

// ❌ Disable TLS certificate verification
process.env.NODE_TLS_REJECT_UNAUTHORIZED = "0";
```

---

_[← Logging](logging.md) · [Database →](database.md)_
