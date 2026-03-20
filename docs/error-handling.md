# Error Handling

← [Back to README](../README.md)

## Philosophy

In Node.js/TypeScript, errors are objects — not return values. RBKL uses a **custom error class hierarchy** so that errors carry semantic meaning, are distinguishable by type, and map deterministically to HTTP status codes.

Unhandled promise rejections are treated as fatal errors — never leave a `Promise` floating without a `catch`.

---

## Custom Error Classes

Define a base `AppError` class and specialized subclasses in the `domain` package:

```typescript
// packages/types/src/errors.ts  (or apps/api/src/domain/errors.ts)

export class AppError extends Error {
  constructor(
    message: string,
    public readonly statusCode: number,
    public readonly code: string,
  ) {
    super(message);
    this.name = this.constructor.name;
    // Capture stack trace correctly in V8
    Error.captureStackTrace(this, this.constructor);
  }
}

export class NotFoundError extends AppError {
  constructor(message = "Not found") {
    super(message, 404, "NOT_FOUND");
  }
}

export class ConflictError extends AppError {
  constructor(message = "Conflict") {
    super(message, 409, "CONFLICT");
  }
}

export class UnauthorizedError extends AppError {
  constructor(message = "Unauthorized") {
    super(message, 401, "UNAUTHORIZED");
  }
}

export class ForbiddenError extends AppError {
  constructor(message = "Forbidden") {
    super(message, 403, "FORBIDDEN");
  }
}

export class ValidationError extends AppError {
  constructor(
    message: string,
    public readonly fields?: Array<{ field: string; message: string }>,
  ) {
    super(message, 422, "VALIDATION_ERROR");
  }
}
```

---

## Checking Error Types

Use `instanceof` for custom errors and type guards for unknown errors:

```typescript
// ✅ Good — type-safe check
try {
  const user = await userRepo.findById(id);
} catch (err) {
  if (err instanceof NotFoundError) {
    return reply.status(404).send(toProblem(err, request.url));
  }
  throw err; // Re-throw unexpected errors
}

// ✅ Type guard for unknown errors
function isError(value: unknown): value is Error {
  return value instanceof Error;
}
```

---

## HTTP Error Responses (RFC 7807)

All HTTP error responses **must** follow [RFC 7807 Problem Details](https://datatracker.ietf.org/doc/html/rfc7807).

```typescript
// apps/api/src/domain/problem-details.ts
export interface ProblemDetails {
  type?: string;
  title: string;
  status: number;
  detail?: string;
  instance?: string;
  errors?: Array<{ field: string; message: string }>;
}

export function toProblemDetails(err: unknown, instance: string): ProblemDetails {
  if (err instanceof ValidationError) {
    return {
      title: "Validation Error",
      status: 422,
      detail: err.message,
      instance,
      errors: err.fields,
    };
  }

  if (err instanceof AppError) {
    return {
      title: err.message,
      status: err.statusCode,
      instance,
    };
  }

  // Unknown error — do NOT expose internal details
  return {
    title: "Internal Server Error",
    status: 500,
    instance,
  };
}
```

### Example Response

```http
HTTP/1.1 422 Unprocessable Entity
Content-Type: application/problem+json

{
  "title": "Validation Error",
  "status": 422,
  "detail": "email: Invalid email address; name: Name too short",
  "instance": "/v1/users",
  "errors": [
    { "field": "email", "message": "Invalid email address" },
    { "field": "name", "message": "Name must be at least 2 characters" }
  ]
}
```

---

## Fastify Error Hook

Register a global `setErrorHandler` to format all errors consistently:

```typescript
// apps/api/src/server.ts
server.setErrorHandler((error, request, reply) => {
  // Fastify validation errors (JSON Schema failures)
  if (error.validation) {
    const fields = error.validation.map((v) => ({
      field: v.instancePath.replace(/^\//, ""),
      message: v.message ?? "Invalid value",
    }));
    return reply.status(422).send({
      title: "Validation Error",
      status: 422,
      detail: error.message,
      instance: request.url,
      errors: fields,
    });
  }

  const problem = toProblemDetails(error, request.url);

  if (problem.status >= 500) {
    request.log.error({ err: error }, "Unhandled error");
  } else {
    request.log.info({ err: error, status: problem.status }, "Request error");
  }

  return reply
    .status(problem.status)
    .header("Content-Type", "application/problem+json")
    .send(problem);
});
```

---

## Async/Await Error Boundaries

Never let unhandled promise rejections escape. Fastify automatically catches async errors thrown from route handlers.

```typescript
// ✅ Fastify handles this automatically — thrown error goes to setErrorHandler
server.get("/users/:id", async (request, reply) => {
  const user = await userService.get(request.params.id); // May throw NotFoundError
  return user; // Fastify will catch any uncaught throw
});

// ❌ Missing await — rejection is unhandled
server.get("/users/:id", (request, reply) => {
  userService.get(request.params.id).then((user) => reply.send(user));
  // If userService.get rejects, the rejection is unhandled!
});
```

### Global Unhandled Rejection Handler

In `index.ts`, register a final safety net:

```typescript
process.on("unhandledRejection", (reason, promise) => {
  logger.fatal({ reason, promise }, "Unhandled promise rejection — exiting");
  process.exit(1);
});

process.on("uncaughtException", (err) => {
  logger.fatal({ err }, "Uncaught exception — exiting");
  process.exit(1);
});
```

---

## Logging Rules

| Error Type | Log Level | Notes |
|---|---|---|
| Client error (4xx) | `info` | Include status code and path |
| Server error (5xx) | `error` | Include full error with stack trace |
| Validation error (422) | `info` | Include field errors |
| Expected domain error (not found, conflict) | `info` | No stack trace needed |
| Unhandled rejection / uncaught exception | `fatal` | Include full error, then exit |

---

## React Error Boundaries

In the React frontend, use React Error Boundaries to catch rendering errors and display a fallback UI:

```tsx
// apps/web/src/components/ErrorBoundary.tsx
import { Component, ReactNode, ErrorInfo } from "react";

interface Props {
  children: ReactNode;
  fallback?: ReactNode;
}

interface State {
  hasError: boolean;
  error: Error | null;
}

export class ErrorBoundary extends Component<Props, State> {
  state: State = { hasError: false, error: null };

  static getDerivedStateFromError(error: Error): State {
    return { hasError: true, error };
  }

  componentDidCatch(error: Error, info: ErrorInfo): void {
    // Log to monitoring service (e.g., Application Insights)
    console.error("Component error:", error, info.componentStack);
  }

  render(): ReactNode {
    if (this.state.hasError) {
      return this.props.fallback ?? <p>Something went wrong. Please reload the page.</p>;
    }
    return this.props.children;
  }
}
```

---

_[← Configuration](configuration.md) · [Testing →](testing.md)_
