# CI/CD Pipeline

← [Back to README](../README.md)

## Platform

RBKL uses **GitHub Actions** exclusively for all CI/CD pipelines. No other CI system is approved.

---

## Required Pipeline Stages

Every application **must** have a CI pipeline running these stages **on every pull request** and **push to `main`**:

| Stage | Failure Blocks Merge? | Description |
|---|---|---|
| `typecheck` | ✅ Yes | `tsc --noEmit` — type errors |
| `lint` | ✅ Yes | ESLint + Prettier check |
| `test:unit` | ✅ Yes | Vitest unit tests |
| `test:integration` | ✅ Yes | Integration tests with real Postgres |
| `build` | ✅ Yes | `tsc` / `vite build` |
| `audit` | ✅ Yes | `pnpm audit --audit-level=high` |
| `secrets-scan` | ✅ Yes | `gitleaks` |
| `coverage` | ✅ Yes | Fail if < 80% |
| `docker-build` | ✅ Yes | Verify Docker image builds |
| `e2e` | ⚠️ Advisory on PRs | Playwright (always runs on `main`) |

---

## Standard CI Workflow

```yaml
# .github/workflows/ci.yaml
name: CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

env:
  NODE_VERSION: "20.14.0"
  PNPM_VERSION: "9"

jobs:
  typecheck:
    name: Type Check
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: pnpm/action-setup@v4
        with:
          version: ${{ env.PNPM_VERSION }}

      - uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: "pnpm"

      - run: pnpm install --frozen-lockfile

      - run: pnpm typecheck   # Runs tsc --noEmit across all workspaces via Turborepo

  lint:
    name: Lint & Format
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v4
        with: { version: "${{ env.PNPM_VERSION }}" }
      - uses: actions/setup-node@v4
        with: { node-version: "${{ env.NODE_VERSION }}", cache: "pnpm" }
      - run: pnpm install --frozen-lockfile
      - run: pnpm lint           # ESLint
      - run: pnpm format:check   # Prettier --check

  test:
    name: Unit Tests
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v4
        with: { version: "${{ env.PNPM_VERSION }}" }
      - uses: actions/setup-node@v4
        with: { node-version: "${{ env.NODE_VERSION }}", cache: "pnpm" }
      - run: pnpm install --frozen-lockfile
      - run: pnpm test:coverage
      - uses: actions/upload-artifact@v4
        with:
          name: coverage
          path: "**/coverage/lcov.info"

  integration-test:
    name: Integration Tests
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:15
        env:
          POSTGRES_USER: test
          POSTGRES_PASSWORD: test
          POSTGRES_DB: testdb
        ports:
          - 5432:5432
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v4
        with: { version: "${{ env.PNPM_VERSION }}" }
      - uses: actions/setup-node@v4
        with: { node-version: "${{ env.NODE_VERSION }}", cache: "pnpm" }
      - run: pnpm install --frozen-lockfile
      - name: Run DB migrations
        working-directory: apps/api
        env:
          DATABASE_URL: "postgresql://test:test@localhost:5432/testdb"
        run: pnpm db:migrate
      - name: Run integration tests
        working-directory: apps/api
        env:
          DATABASE_URL: "postgresql://test:test@localhost:5432/testdb"
          NODE_ENV: test
        run: pnpm test:integration

  build:
    name: Build
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v4
        with: { version: "${{ env.PNPM_VERSION }}" }
      - uses: actions/setup-node@v4
        with: { node-version: "${{ env.NODE_VERSION }}", cache: "pnpm" }
      - run: pnpm install --frozen-lockfile
      - run: pnpm build

  audit:
    name: Security Audit
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v4
        with: { version: "${{ env.PNPM_VERSION }}" }
      - run: pnpm audit --audit-level=high

  secrets-scan:
    name: Secrets Scan
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
      - uses: gitleaks/gitleaks-action@v2
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}

  docker-build:
    name: Docker Build
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: docker build -t api:${{ github.sha }} apps/api
      - run: docker build -t web:${{ github.sha }} apps/web
```

---

## Deployment Pipeline

```yaml
# .github/workflows/deploy.yaml
name: Deploy

on:
  push:
    branches: [main]

jobs:
  deploy-dev:
    name: Deploy to Dev
    needs: [typecheck, lint, test, integration-test, build, audit, secrets-scan, docker-build]
    runs-on: ubuntu-latest
    environment: dev
    steps:
      - uses: actions/checkout@v4

      - name: Login to container registry
        uses: azure/docker-login@v2
        with:
          login-server: ${{ vars.REGISTRY_URL }}
          username: ${{ secrets.REGISTRY_USERNAME }}
          password: ${{ secrets.REGISTRY_PASSWORD }}

      - name: Build and push images
        run: |
          docker build -t ${{ vars.REGISTRY_URL }}/api:${{ github.sha }} apps/api
          docker push ${{ vars.REGISTRY_URL }}/api:${{ github.sha }}
          docker build -t ${{ vars.REGISTRY_URL }}/web:${{ github.sha }} apps/web
          docker push ${{ vars.REGISTRY_URL }}/web:${{ github.sha }}

      - name: Run DB migrations
        uses: azure/k8s-set-context@v4
        # ... connect to cluster and run: kubectl exec ... pnpm db:migrate

      - name: Deploy to Kubernetes
        uses: azure/k8s-deploy@v5
        with:
          namespace: dev
          manifests: kubernetes/
          images: |
            ${{ vars.REGISTRY_URL }}/api:${{ github.sha }}
            ${{ vars.REGISTRY_URL }}/web:${{ github.sha }}

  deploy-staging:
    name: Deploy to Staging
    needs: deploy-dev
    runs-on: ubuntu-latest
    environment: staging   # Requires 1 manual approval

  deploy-prod:
    name: Deploy to Production
    needs: deploy-staging
    runs-on: ubuntu-latest
    environment: production  # Requires 2 approvals
```

---

## Deployment Environments

| Environment | Trigger | Approval | DB Migration | Notes |
|---|---|---|---|---|
| `dev` | Push to `main` | Automatic | Automatic | Auto-deployed |
| `staging` | Push to `main` | 1 approver | Automatic | Production-like data |
| `production` | Push to `main` | 2 approvers | Manual gate | Real user data |

---

## Release Process & Semantic Versioning

RBKL uses [Semantic Versioning 2.0](https://semver.org/) and **Conventional Commits** for automated changelog generation.

### Commit Message Convention

```
feat: add user export endpoint
fix: correct pagination cursor encoding
chore: update prisma to v5.5.0
docs: update API design section
refactor: extract JWT validation to separate module
BREAKING CHANGE: rename createUser to registerUser
```

| Prefix | SemVer impact |
|---|---|
| `feat` | MINOR |
| `fix` | PATCH |
| `BREAKING CHANGE:` or `feat!:` | MAJOR |
| `chore`, `docs`, `refactor`, `test` | None |

### Creating a Release

```bash
# Create and push a tag
git tag -a v1.2.3 -m "Release v1.2.3"
git push origin v1.2.3
```

The release workflow automatically:
1. Creates a GitHub Release with an auto-generated changelog.
2. Builds and pushes versioned Docker images.
3. Deploys to production after approval.

---

## Rollback Procedures

### Immediate Rollback (< 5 min)

```bash
kubectl rollout undo deployment/api -n production
kubectl rollout undo deployment/web -n production
```

### Planned Rollback

1. Identify the last stable image tag.
2. Update Kubernetes manifests to the previous tag.
3. If a DB migration was deployed, run the down migration first.
4. Create a PR for the image tag change and fast-track approval.

**Never rollback the application to a version incompatible with the current database schema.** Down migrations must be run before rolling back the application code.

---

## ESLint Configuration

Every repository uses a shared RBKL ESLint config:

```json
// .eslintrc.cjs
module.exports = {
  root: true,
  extends: [
    "eslint:recommended",
    "plugin:@typescript-eslint/strict-type-checked",
    "plugin:@typescript-eslint/stylistic-type-checked",
    "prettier"
  ],
  parser: "@typescript-eslint/parser",
  parserOptions: {
    project: true,
    tsconfigRootDir: __dirname,
  },
  plugins: ["@typescript-eslint"],
  rules: {
    "@typescript-eslint/no-explicit-any": "error",
    "@typescript-eslint/no-floating-promises": "error",
    "@typescript-eslint/await-thenable": "error",
    "no-console": "warn",
  }
};
```

For React apps, also extend `plugin:react-hooks/recommended` and `plugin:jsx-a11y/recommended`.

---

_[← Dependency Management](dependency-management.md) · [Back to README](../README.md)_
