# Testing

← [Back to README](../README.md)

## Approved Libraries

| Library | Purpose | Version |
|---|---|---|
| **Vitest** | Unit and integration test runner | v1 |
| **React Testing Library** | React component testing | latest |
| **@testing-library/user-event** | Realistic user interaction simulation | latest |
| **Supertest** | HTTP integration testing | latest |
| **Playwright** | End-to-end browser testing | latest |
| **msw** (Mock Service Worker) | API mocking in browser and Node | v2 |

```bash
pnpm add -D vitest @vitest/coverage-v8 supertest @types/supertest
pnpm add -D @testing-library/react @testing-library/user-event @testing-library/jest-dom
pnpm add -D msw playwright
```

---

## Vitest Configuration

```typescript
// vitest.config.ts (root or per-app)
import { defineConfig } from "vitest/config";
import path from "path";

export default defineConfig({
  test: {
    environment: "node",        // Use "jsdom" for React component tests
    globals: false,             // Explicit imports only — no magic globals
    coverage: {
      provider: "v8",
      reporter: ["text", "lcov", "html"],
      thresholds: {
        statements: 80,
        branches: 80,
        functions: 80,
        lines: 80,
      },
      exclude: [
        "node_modules",
        "dist",
        "**/*.d.ts",
        "**/index.ts",          // Re-export files
        "prisma/**",
        "**/*.config.ts",
      ],
    },
  },
  resolve: {
    alias: {
      "@": path.resolve(__dirname, "./src"),
    },
  },
});
```

---

## Test File Structure

Tests live alongside the code they test:

```
apps/api/src/
├── services/
│   ├── user-service.ts
│   └── user-service.test.ts      # Unit tests
├── routes/
│   ├── users.ts
│   └── users.test.ts             # Integration tests (Supertest)
└── repositories/
    ├── user-repository.ts
    └── user-repository.test.ts   # Integration tests (real DB, .env.test)

apps/web/src/
├── features/users/
│   ├── UserCard.tsx
│   ├── UserCard.test.tsx         # Component tests (RTL)
│   └── use-user.test.ts          # Hook tests
```

---

## Coverage Requirements

**Minimum: 80% statement, branch, function, and line coverage** on all `src/` code.

CI enforces the threshold via `vitest run --coverage`. A single well-tested module does not compensate for an untested one.

---

## Unit Testing Services

Use `vi.fn()` and `vi.spyOn()` for mocks. Structure tests with `describe` + `it`.

```typescript
// apps/api/src/services/user-service.test.ts
import { describe, it, expect, vi, beforeEach } from "vitest";
import { UserService } from "./user-service.js";
import { NotFoundError } from "../domain/errors.js";

const mockRepo = {
  findById: vi.fn(),
  create: vi.fn(),
  softDelete: vi.fn(),
};

const mockLogger = {
  info: vi.fn(),
  warn: vi.fn(),
  error: vi.fn(),
  debug: vi.fn(),
};

describe("UserService", () => {
  let service: UserService;

  beforeEach(() => {
    vi.clearAllMocks();
    service = new UserService(mockRepo as any, mockLogger as any);
  });

  describe("get", () => {
    it("returns the user when found", async () => {
      const user = { id: "usr_1", email: "alice@example.com", name: "Alice" };
      mockRepo.findById.mockResolvedValue(user);

      const result = await service.get("usr_1");

      expect(result).toEqual(user);
      expect(mockRepo.findById).toHaveBeenCalledWith("usr_1");
    });

    it("re-throws NotFoundError when user does not exist", async () => {
      mockRepo.findById.mockRejectedValue(new NotFoundError("User not found"));

      await expect(service.get("missing")).rejects.toThrow(NotFoundError);
    });
  });
});
```

---

## HTTP Integration Testing with Supertest

Test Fastify routes end-to-end without a running server (Fastify supports `inject`):

```typescript
// apps/api/src/routes/users.test.ts
import { describe, it, expect, beforeAll, afterAll } from "vitest";
import { buildServer } from "../server.js";
import { loadConfig } from "../config/index.js";

const config = loadConfig();
const server = await buildServer(config);

beforeAll(async () => server.ready());
afterAll(async () => server.close());

describe("GET /v1/users/:id", () => {
  it("returns 401 when no token is provided", async () => {
    const response = await server.inject({
      method: "GET",
      url: "/v1/users/usr_1",
    });

    expect(response.statusCode).toBe(401);
  });

  it("returns 404 when user does not exist", async () => {
    const response = await server.inject({
      method: "GET",
      url: "/v1/users/nonexistent",
      headers: { authorization: `Bearer ${await getTestToken()}` },
    });

    expect(response.statusCode).toBe(404);
    const body = response.json<{ title: string; status: number }>();
    expect(body.status).toBe(404);
  });
});
```

> **Note:** For integration tests against a real database, set `TEST_DATABASE_URL` in `.env.test` and run `prisma migrate deploy` before the test suite.

---

## React Component Testing

Use **React Testing Library** (RTL) with `@testing-library/user-event` for realistic user interaction:

```tsx
// apps/web/src/features/users/UserCard.test.tsx
import { describe, it, expect } from "vitest";
import { render, screen } from "@testing-library/react";
import userEvent from "@testing-library/user-event";
import { UserCard } from "./UserCard.js";

describe("UserCard", () => {
  const user = { id: "usr_1", name: "Alice", email: "alice@example.com" };

  it("displays user name and email", () => {
    render(<UserCard user={user} onDelete={vi.fn()} />);

    expect(screen.getByText("Alice")).toBeInTheDocument();
    expect(screen.getByText("alice@example.com")).toBeInTheDocument();
  });

  it("calls onDelete with user ID when delete button is clicked", async () => {
    const onDelete = vi.fn();
    render(<UserCard user={user} onDelete={onDelete} />);

    await userEvent.click(screen.getByRole("button", { name: /delete/i }));

    expect(onDelete).toHaveBeenCalledWith("usr_1");
  });
});
```

### RTL Best Practices

- Query by **role** first (`getByRole`), then by label text, then by test ID.
- **Never** query by class name or internal implementation details.
- Use `userEvent` (not `fireEvent`) for user interactions — it simulates real browser events.
- Avoid testing implementation details — test what the user sees and can do.

```tsx
// ✅ Good — tests user-visible behavior
expect(screen.getByRole("button", { name: /submit/i })).toBeDisabled();

// ❌ Bad — tests implementation detail
expect(wrapper.find("Button").prop("disabled")).toBe(true);
```

---

## API Mocking with MSW

Use **Mock Service Worker** to intercept API calls in component tests without modifying the fetch implementation:

```typescript
// apps/web/src/tests/setup.ts
import { beforeAll, afterEach, afterAll } from "vitest";
import { setupServer } from "msw/node";
import { http, HttpResponse } from "msw";

export const server = setupServer(
  http.get("/v1/users/:id", ({ params }) => {
    return HttpResponse.json({ id: params.id, name: "Alice", email: "alice@example.com" });
  }),
);

beforeAll(() => server.listen({ onUnhandledRequest: "error" }));
afterEach(() => server.resetHandlers());
afterAll(() => server.close());
```

---

## End-to-End Testing with Playwright

E2E tests live in `apps/web/e2e/` and run against a deployed environment:

```typescript
// apps/web/e2e/login.spec.ts
import { test, expect } from "@playwright/test";

test("user can log in and see dashboard", async ({ page }) => {
  await page.goto("/");
  await page.getByRole("button", { name: /sign in/i }).click();
  // Azure AD login handled via test account credentials
  await page.getByLabel("Email").fill(process.env.TEST_USER_EMAIL!);
  await page.getByLabel("Password").fill(process.env.TEST_USER_PASSWORD!);
  await page.getByRole("button", { name: /next/i }).click();

  await expect(page).toHaveURL("/dashboard");
  await expect(page.getByRole("heading", { name: /dashboard/i })).toBeVisible();
});
```

---

## Prohibited Patterns

```typescript
// ❌ Skipping assertions with empty expects
it("creates user", async () => {
  await userService.create(data);
  // No assertion — test always passes
});

// ❌ Sleep-based timing (flaky tests)
await new Promise((r) => setTimeout(r, 500));

// ❌ Importing from node_modules internals
import { internalHelper } from "some-lib/dist/internal.js";

// ❌ Not cleaning up mocks between tests
vi.fn(); // Without vi.clearAllMocks() in beforeEach
```

---

_[← Error Handling](error-handling.md) · [API Design →](api-design.md)_
