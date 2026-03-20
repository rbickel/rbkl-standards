# API Design

← [Back to README](../README.md)

## Principles

RBKL APIs are:

1. **RESTful** — Resources are nouns; HTTP methods are verbs.
2. **Versioned from day one** — URL path versioning (`/v1/`).
3. **OpenAPI-first** — The spec is generated from route schemas using `@fastify/swagger`.
4. **Consistent** — Same conventions across all services.

---

## URL Structure

```
https://api.rbkl.com/{service}/{version}/{resource}/{id}/{sub-resource}
```

| Segment | Convention | Example |
|---|---|---|
| `service` | Kebab-case service name | `user-service` |
| `version` | `v{integer}` | `v1` |
| `resource` | Lowercase plural noun | `users` |
| `id` | Resource identifier | `usr_01HX...` |
| `sub-resource` | Lowercase plural noun | `addresses` |

Examples:
```
GET    /v1/users
GET    /v1/users/usr_01HX
POST   /v1/users
PUT    /v1/users/usr_01HX
DELETE /v1/users/usr_01HX
GET    /v1/users/usr_01HX/addresses
```

---

## HTTP Method Semantics

| Method | Usage | Idempotent | Body |
|---|---|---|---|
| `GET` | Retrieve resource(s) | ✅ | ❌ |
| `POST` | Create a new resource | ❌ | ✅ |
| `PUT` | Replace a resource entirely | ✅ | ✅ |
| `PATCH` | Partial update | ❌ | ✅ |
| `DELETE` | Remove a resource | ✅ | ❌ |

**Rules:**
- `GET` requests must never modify state.
- `DELETE` returns `204 No Content` on success.
- `PUT` must be idempotent.
- Prefer `PUT` over `PATCH` unless partial updates are explicitly required.

---

## HTTP Status Codes

| Code | Meaning | When |
|---|---|---|
| `200 OK` | Success | `GET`, `PUT`, `PATCH` with response body |
| `201 Created` | Resource created | `POST` that creates a resource |
| `204 No Content` | Success, no body | `DELETE`, `PUT` with no response |
| `400 Bad Request` | Malformed syntax or payload | JSON parse error |
| `401 Unauthorized` | Missing or invalid authentication | No/invalid JWT |
| `403 Forbidden` | Authenticated but not authorized | Insufficient role |
| `404 Not Found` | Resource does not exist | ID not found |
| `409 Conflict` | State conflict | Duplicate email, concurrent write |
| `422 Unprocessable Entity` | Semantic validation failure | Invalid email format |
| `429 Too Many Requests` | Rate limit exceeded | — |
| `500 Internal Server Error` | Unexpected server error | Unhandled error |
| `503 Service Unavailable` | Dependency unavailable | Readiness check failed |

---

## Request/Response Format

### Content Type

All request and response bodies use `application/json`. Error responses use `application/problem+json` (see [Error Handling](error-handling.md)).

### Response Envelope

RBKL does **not** wrap single-resource responses in an envelope:

```json
// ✅ Correct — no envelope for single resource
{
  "id": "usr_01HX",
  "email": "alice@rbkl.com",
  "name": "Alice"
}

// ❌ Wrong — unnecessary wrapper
{
  "data": { "id": "usr_01HX" },
  "success": true
}
```

### Collection Responses

Collections use a standard envelope for pagination metadata:

```json
{
  "items": [
    { "id": "usr_01HX", "email": "alice@rbkl.com" },
    { "id": "usr_02HX", "email": "bob@rbkl.com" }
  ],
  "pagination": {
    "total": 42,
    "limit": 20,
    "nextCursor": "eyJpZCI6InVzcl8wMkhYIn0="
  }
}
```

---

## Pagination

Use **cursor-based pagination** for all list endpoints. Offset pagination is only acceptable for admin interfaces with datasets < 10,000 records.

### Request Parameters

| Parameter | Type | Default | Max | Description |
|---|---|---|---|---|
| `limit` | number | 20 | 100 | Items per page |
| `cursor` | string | — | — | Opaque cursor from previous response |

### TypeBox Schema

```typescript
import { Type, Static } from "@sinclair/typebox";

const ListQuerySchema = Type.Object({
  limit: Type.Optional(Type.Integer({ minimum: 1, maximum: 100, default: 20 })),
  cursor: Type.Optional(Type.String()),
});

type ListQuery = Static<typeof ListQuerySchema>;

// Cursor is base64-encoded JSON: { "id": "last-seen-id" }
function encodeCursor(id: string): string {
  return Buffer.from(JSON.stringify({ id })).toString("base64url");
}

function decodeCursor(cursor: string): { id: string } {
  return JSON.parse(Buffer.from(cursor, "base64url").toString());
}
```

---

## Filtering & Sorting

### Filtering

Use query parameters with the field name as the key:

```
GET /v1/users?status=active&role=admin
GET /v1/orders?createdAfter=2026-01-01T00:00:00Z&createdBefore=2026-03-01T00:00:00Z
```

### Sorting

```
GET /v1/users?sort=createdAt&order=desc
GET /v1/users?sort=-createdAt     # Shorthand: - prefix for descending
```

Only expose sorting on indexed columns. Document supported sort fields in the OpenAPI spec.

---

## API Versioning

RBKL uses **URL path versioning**. Increment the version integer on breaking changes.

### What Constitutes a Breaking Change?

- Removing or renaming a field in a response.
- Changing a field's type.
- Removing an endpoint.
- Changing required/optional status of a parameter.
- Changing authentication requirements.

### What Is NOT a Breaking Change?

- Adding new optional request parameters.
- Adding new fields to a response.
- Adding new endpoints.

### Version Lifecycle

| Status | Description |
|---|---|
| `current` | Latest stable version — all new clients use this |
| `deprecated` | Supported for 12 months after successor is released |
| `sunset` | Returns `410 Gone` |

Include these headers on deprecated endpoints:

```
Deprecation: true
Sunset: Sat, 20 Mar 2027 00:00:00 GMT
Link: <https://api.rbkl.com/v2/users>; rel="successor-version"
```

---

## OpenAPI 3.x with `@fastify/swagger`

Every service generates an OpenAPI 3.x spec automatically from route schemas.

### Setup

```typescript
import swagger from "@fastify/swagger";
import swaggerUi from "@fastify/swagger-ui";

await server.register(swagger, {
  openapi: {
    info: {
      title: "User Service API",
      version: "1.0.0",
    },
    components: {
      securitySchemes: {
        bearerAuth: {
          type: "http",
          scheme: "bearer",
          bearerFormat: "JWT",
        },
      },
    },
    security: [{ bearerAuth: [] }],
  },
});

await server.register(swaggerUi, {
  routePrefix: "/docs",
  uiConfig: { deepLinking: true },
});
```

### OpenAPI Requirements

- All endpoints documented with request/response schemas.
- All error response codes included.
- `operationId` set on every route.
- Examples provided for all request/response bodies.
- Spec available at `/docs` (dev/staging only — blocked in production).

---

## Idempotency

For non-idempotent operations (`POST`) that must be safe to retry (payment processing, emails), support the `Idempotency-Key` header:

```
POST /v1/payments
Idempotency-Key: 4f0d6a09-3e1c-4d9f-9c85-2a1b3e5f7890
```

The server stores the response keyed by the idempotency key for 24 hours. Subsequent requests with the same key return the cached response.

---

_[← Testing](testing.md) · [Security →](security.md)_
