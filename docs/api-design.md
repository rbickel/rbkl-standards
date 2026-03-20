# API Design

← [Back to README](../README.md)

## Principles

RBKL APIs are:

1. **RESTful** — Resources are nouns; HTTP methods are verbs.
2. **Versioned from day one** — URL path versioning (`/v1/`).
3. **OpenAPI-first** — The spec is written before the implementation.
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
GET  /v1/users
GET  /v1/users/usr_01HX
POST /v1/users
PUT  /v1/users/usr_01HX
DELETE /v1/users/usr_01HX
GET  /v1/users/usr_01HX/addresses
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
- `PUT` must be idempotent — calling it twice produces the same result.
- Avoid `PATCH` unless partial updates are explicitly required; prefer `PUT` with full replacement.

---

## HTTP Status Codes

| Code | Meaning | When |
|---|---|---|
| `200 OK` | Success | `GET`, `PUT`, `PATCH` returns data |
| `201 Created` | Resource created | `POST` that creates a resource |
| `204 No Content` | Success, no body | `DELETE`, `PUT` with no response body |
| `400 Bad Request` | Invalid request syntax or payload | JSON parse error, missing required field |
| `401 Unauthorized` | Missing or invalid authentication | No/invalid JWT |
| `403 Forbidden` | Authenticated but not authorized | Insufficient role |
| `404 Not Found` | Resource does not exist | ID not found |
| `409 Conflict` | State conflict | Duplicate email, concurrent write |
| `422 Unprocessable Entity` | Semantic validation failure | Email format invalid, name too short |
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

// ❌ Wrong — unnecessary envelope
{
  "data": {
    "id": "usr_01HX",
    ...
  },
  "success": true
}
```

### Collection Responses

Collections use a standard envelope to support pagination metadata:

```json
{
  "items": [
    {"id": "usr_01HX", "email": "alice@rbkl.com"},
    {"id": "usr_02HX", "email": "bob@rbkl.com"}
  ],
  "pagination": {
    "total": 42,
    "limit": 20,
    "offset": 0,
    "next_cursor": "eyJpZCI6InVzcl8wMkhYIn0="
  }
}
```

---

## Pagination

Use **cursor-based pagination** for all list endpoints. Offset pagination is only acceptable for admin interfaces with small datasets (< 10,000 records).

### Request Parameters

| Parameter | Type | Default | Max | Description |
|---|---|---|---|---|
| `limit` | int | 20 | 100 | Number of items per page |
| `cursor` | string | — | — | Opaque cursor from previous response |

### Implementation

```go
type ListUsersParams struct {
    Limit  int    `schema:"limit"`
    Cursor string `schema:"cursor"`
}

type PageInfo struct {
    Total      int64  `json:"total"`
    Limit      int    `json:"limit"`
    NextCursor string `json:"next_cursor,omitempty"`
}

// Cursor is a base64-encoded JSON object: {"id": "last-seen-id", "ts": 1234567890}
func encodeCursor(id string, ts time.Time) string {
    data, _ := json.Marshal(map[string]any{"id": id, "ts": ts.Unix()})
    return base64.URLEncoding.EncodeToString(data)
}
```

---

## Filtering & Sorting

### Filtering

Use query parameters with the field name as the key:

```
GET /v1/users?status=active&role=admin
GET /v1/orders?created_after=2026-01-01T00:00:00Z&created_before=2026-03-01T00:00:00Z
```

### Sorting

```
GET /v1/users?sort=created_at&order=desc
GET /v1/users?sort=-created_at          # Shorthand: - prefix for descending
```

Only expose sorting on indexed columns. Document supported sort fields in the OpenAPI spec.

---

## API Versioning

RBKL uses **URL path versioning**. The version number is a single integer incremented on breaking changes.

### What Constitutes a Breaking Change?

- Removing or renaming a field in a response.
- Changing a field's type.
- Removing an endpoint.
- Changing required/optional status of a request parameter.
- Changing authentication requirements.

### What Is NOT a Breaking Change?

- Adding new optional request parameters.
- Adding new fields to a response.
- Adding new endpoints.

### Version Lifecycle

| Status | Description |
|---|---|
| `current` | The latest stable version — all new clients should use it |
| `deprecated` | Supported for 12 months after successor version is released |
| `sunset` | No longer functional — returns `410 Gone` |

Include `Deprecation` and `Sunset` headers on deprecated endpoints:

```
Deprecation: true
Sunset: Sat, 20 Mar 2027 00:00:00 GMT
Link: <https://api.rbkl.com/v2/users>; rel="successor-version"
```

---

## OpenAPI 3.x Specification

Every service **must** include a complete `api/openapi.yaml` specification. The spec is the **source of truth** for the API contract.

### Requirements

- All endpoints documented with request/response schemas.
- All possible error response codes documented.
- `operationId` set on every operation (used for code generation).
- Security schemes defined and applied to all authenticated endpoints.
- Examples provided for all request/response bodies.

### Code Generation

Use `oapi-codegen` to generate server stubs and client code from the spec:

```bash
go install github.com/oapi-codegen/oapi-codegen/v2/cmd/oapi-codegen@latest
oapi-codegen -generate chi-server,types -package api api/openapi.yaml > internal/api/server.gen.go
```

---

## Idempotency

For non-idempotent operations (`POST`) that must be safe to retry (e.g., payment processing, email sending), support the `Idempotency-Key` header.

```
POST /v1/payments
Idempotency-Key: 4f0d6a09-3e1c-4d9f-9c85-2a1b3e5f7890
```

The server stores the response keyed by the idempotency key for 24 hours. Subsequent requests with the same key return the cached response without re-executing the operation.

---

_[← Testing](testing.md) · [Security →](security.md)_
