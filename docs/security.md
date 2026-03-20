# Security

← [Back to README](../README.md)

## Principles

Security is built into every layer. RBKL adopts a **defense-in-depth** strategy: no single control is relied upon exclusively, and every service is treated as potentially compromised.

---

## Input Validation

All external input is validated with **Zod** (backend) or **Fastify JSON Schema** (HTTP layer) before use.

### Backend (Fastify + TypeBox)

```typescript
import { Type, Static } from "@sinclair/typebox";

const CreateUserBody = Type.Object({
  email: Type.String({ format: "email", maxLength: 254 }),
  name: Type.String({ minLength: 2, maxLength: 100 }),
  role: Type.Union([
    Type.Literal("admin"),
    Type.Literal("editor"),
    Type.Literal("viewer"),
  ]),
});

// Fastify validates this schema BEFORE the handler runs
server.post("/", { schema: { body: CreateUserBody } }, handler);
```

### Frontend (React + Zod)

Validate form data with `react-hook-form` + Zod resolver:

```tsx
import { useForm } from "react-hook-form";
import { zodResolver } from "@hookform/resolvers/zod";
import { z } from "zod";

const CreateUserSchema = z.object({
  email: z.string().email("Must be a valid email"),
  name: z.string().min(2, "At least 2 characters").max(100),
});

type FormData = z.infer<typeof CreateUserSchema>;

function CreateUserForm() {
  const { register, handleSubmit, formState: { errors } } = useForm<FormData>({
    resolver: zodResolver(CreateUserSchema),
  });
  // ...
}
```

---

## React-Specific Security

### XSS Prevention

React escapes values by default in JSX. **Never bypass this protection**:

```tsx
// ✅ Safe — React escapes this
<div>{userSuppliedContent}</div>

// ❌ Dangerous — bypasses React's escaping
<div dangerouslySetInnerHTML={{ __html: userSuppliedContent }} />
```

If rendering HTML from a trusted source is absolutely required, sanitize it with **DOMPurify**:

```tsx
import DOMPurify from "dompurify";

<div dangerouslySetInnerHTML={{ __html: DOMPurify.sanitize(trustedHtml) }} />
```

### Content Security Policy (CSP)

Set a strict CSP header on all HTML responses from the backend serving the React app:

```typescript
server.addHook("onSend", async (request, reply, payload) => {
  if (reply.getHeader("content-type")?.toString().includes("text/html")) {
    reply.header(
      "Content-Security-Policy",
      [
        "default-src 'self'",
        "script-src 'self'",        // No 'unsafe-inline', no CDN scripts
        "style-src 'self' 'unsafe-inline'",  // Inline styles needed for CSS-in-JS
        "img-src 'self' data: https://cdn.rbkl.com",
        "connect-src 'self' https://api.rbkl.com https://login.microsoftonline.com",
        "frame-ancestors 'none'",
      ].join("; "),
    );
  }
});
```

### Secure Cookie Handling

```typescript
// ✅ Secure cookie flags (if using cookies for session state)
reply.setCookie("session", value, {
  httpOnly: true,    // Not accessible via JavaScript
  secure: true,      // HTTPS only
  sameSite: "strict",
  maxAge: 3600,
  path: "/",
});
```

---

## HTTP Security Headers

Apply these headers to all responses via a Fastify plugin:

```typescript
import fp from "fastify-plugin";
import helmet from "@fastify/helmet";

// @fastify/helmet sets all recommended security headers
await server.register(helmet, {
  contentSecurityPolicy: false, // We set CSP manually (see above)
  crossOriginEmbedderPolicy: false, // Only needed for SharedArrayBuffer
});

// Additional manual headers
server.addHook("onSend", async (request, reply) => {
  reply.header("X-Content-Type-Options", "nosniff");
  reply.header("X-Frame-Options", "DENY");
  reply.header("Referrer-Policy", "strict-origin-when-cross-origin");
  reply.header("Strict-Transport-Security", "max-age=31536000; includeSubDomains");
  reply.header("Permissions-Policy", "camera=(), microphone=(), geolocation=()");
});
```

Install `@fastify/helmet`:

```bash
pnpm add @fastify/helmet
```

---

## Dependency Scanning

### `npm audit`

Run `npm audit` (or `pnpm audit`) on every CI pipeline:

```bash
pnpm audit --audit-level=high
```

CI fails on any `high` or `critical` severity vulnerability with a fix available.

### Dependabot

Enable Dependabot in `.github/dependabot.yml`:

```yaml
version: 2
updates:
  - package-ecosystem: "npm"
    directory: "/"
    schedule:
      interval: "weekly"
    open-pull-requests-limit: 10
    labels:
      - "dependencies"
    groups:
      react:
        patterns: ["react", "react-dom", "@types/react*"]
      azure:
        patterns: ["@azure/*", "@azure-rest/*"]
```

---

## Pre-commit Hooks for Secrets Detection

```yaml
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/gitleaks/gitleaks
    rev: v8.18.2
    hooks:
      - id: gitleaks
```

Also run `gitleaks` in CI as a non-bypassable check (see [CI/CD](ci-cd.md)).

---

## OWASP Top 10 Mitigations

### A01: Broken Access Control
- JWT validation + RBAC middleware on every protected route (see [Authentication](authentication.md)).
- No security decisions based on client-supplied headers.
- Always filter data by the authenticated user's tenant ID.

### A02: Cryptographic Failures
- All communication over HTTPS (TLS 1.2+). HTTP is not permitted.
- Passwords hashed with `bcrypt` (cost ≥ 12) or `argon2id`.
- No MD5 or SHA-1 for security-sensitive hashing.
- Tokens generated with `crypto.randomBytes(32)` — never `Math.random()`.

```typescript
// ✅ Cryptographically secure token
import { randomBytes } from "crypto";
const token = randomBytes(32).toString("hex");

// ❌ Predictable
const token = Math.random().toString(36);
```

### A03: Injection
- All database access through Prisma parameterized queries.
- Never use `prisma.$queryRawUnsafe()` with user-supplied input.
- HTML sanitized with DOMPurify before `dangerouslySetInnerHTML`.
- No `eval()`, `new Function()`, or dynamic `import()` with user input.

### A04: Insecure Design
- Threat model documented for every service in `docs/threat-model.md`.
- Debug endpoints (`/metrics`, profiling) protected by auth or removed in production.

### A05: Security Misconfiguration
- `@fastify/helmet` enabled on all services.
- No default or hard-coded credentials.
- Detailed error messages only returned in non-production environments.
- `NODE_ENV` is always set explicitly — never rely on its absence.

### A06: Vulnerable & Outdated Components
- `pnpm audit` + Dependabot (see above).
- Node.js base images use specific digest pins in Dockerfiles.

### A07: Authentication Failures
- Short-lived JWT tokens (max 1 hour).
- No credentials in URLs (query parameters).
- Tokens stored in memory or `sessionStorage` — **never `localStorage`**.
- PKCE flow required for all public SPA clients.

### A08: Software and Data Integrity Failures
- All NPM packages verified by `pnpm audit` and `package-lock.json`/`pnpm-lock.yaml`.
- Docker images signed in the deployment pipeline.

### A09: Security Logging and Monitoring
- All authentication failures logged at `info` (see [Logging](logging.md)).
- Alerts configured for repeated 401/403 responses.
- Audit log for all write operations on sensitive resources.

### A10: SSRF
- Never make HTTP requests to URLs supplied directly by clients.
- Validate URLs against an allowlist of approved domains.
- Block requests to link-local and private IP ranges in any outbound HTTP client.

---

## Rate Limiting

```typescript
import rateLimit from "@fastify/rate-limit";

await server.register(rateLimit, {
  global: true,
  max: 100,           // 100 requests per window
  timeWindow: 60_000, // 1 minute
  keyGenerator: (request) =>
    request.jwtClaims?.sub ?? request.ip, // Per-user when authenticated, per-IP otherwise
  errorResponseBuilder: () => ({
    title: "Too Many Requests",
    status: 429,
  }),
});
```

Install: `pnpm add @fastify/rate-limit`

---

_[← API Design](api-design.md) · [Dependency Management →](dependency-management.md)_
