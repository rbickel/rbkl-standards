# CI/CD Pipeline

← [Back to README](../README.md)

## Platform

RBKL uses **GitHub Actions** exclusively for all CI/CD pipelines. No other CI system (Jenkins, CircleCI, Azure DevOps Pipelines, etc.) is approved.

---

## Required Pipeline Stages

Every service **must** have a CI pipeline that runs these stages **on every pull request** and **every push to `main`**:

| Stage | Failure Blocks Merge? | Description |
|---|---|---|
| `lint` | ✅ Yes | `golangci-lint` + `gofmt` check |
| `test` | ✅ Yes | Unit tests with `-race` flag |
| `build` | ✅ Yes | `go build ./...` |
| `vuln-scan` | ✅ Yes | `govulncheck ./...` |
| `secrets-scan` | ✅ Yes | `gitleaks` |
| `integration-test` | ✅ Yes (on `main` only) | Tests with a real Postgres instance |
| `docker-build` | ✅ Yes | Verify Docker image builds |
| `coverage` | ⚠️ Advisory | Report coverage; fail if < 80% |

---

## Standard Workflow File

Every service repository includes `.github/workflows/ci.yaml`:

```yaml
name: CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

env:
  GO_VERSION: "1.22"

jobs:
  lint:
    name: Lint
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-go@v5
        with:
          go-version: ${{ env.GO_VERSION }}
          cache: true

      - name: Check formatting
        run: |
          if [ "$(gofmt -l . | wc -l)" -gt 0 ]; then
            echo "Files not formatted:"
            gofmt -l .
            exit 1
          fi

      - name: golangci-lint
        uses: golangci/golangci-lint-action@v6
        with:
          version: latest

  test:
    name: Test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-go@v5
        with:
          go-version: ${{ env.GO_VERSION }}
          cache: true

      - name: Run unit tests
        run: go test -race -coverprofile=coverage.out ./...

      - name: Check coverage
        run: |
          COVERAGE=$(go tool cover -func=coverage.out | grep total | awk '{print $3}' | tr -d '%')
          echo "Total coverage: ${COVERAGE}%"
          if (( $(echo "$COVERAGE < 80" | bc -l) )); then
            echo "Coverage ${COVERAGE}% is below 80% threshold"
            exit 1
          fi

      - name: Upload coverage
        uses: actions/upload-artifact@v4
        with:
          name: coverage
          path: coverage.out

  build:
    name: Build
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-go@v5
        with:
          go-version: ${{ env.GO_VERSION }}
          cache: true

      - name: Build binary
        run: go build -v ./...

  vuln-scan:
    name: Vulnerability Scan
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-go@v5
        with:
          go-version: ${{ env.GO_VERSION }}
          cache: true

      - name: Run govulncheck
        run: |
          go install golang.org/x/vuln/cmd/govulncheck@latest
          govulncheck ./...

  secrets-scan:
    name: Secrets Scan
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0   # Full history for gitleaks

      - name: Run gitleaks
        uses: gitleaks/gitleaks-action@v2
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}

  integration-test:
    name: Integration Tests
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main' || github.event_name == 'pull_request'
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

      - uses: actions/setup-go@v5
        with:
          go-version: ${{ env.GO_VERSION }}
          cache: true

      - name: Run migrations
        run: |
          go install -tags 'postgres' github.com/golang-migrate/migrate/v4/cmd/migrate@latest
          migrate -path migrations -database "postgres://test:test@localhost:5432/testdb?sslmode=disable" up

      - name: Run integration tests
        env:
          TEST_DATABASE_URL: "postgres://test:test@localhost:5432/testdb?sslmode=disable"
        run: go test -tags=integration -race ./...

  docker-build:
    name: Docker Build
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Build Docker image
        run: docker build -t my-service:${{ github.sha }} .
```

---

## Deployment Pipeline

The deployment pipeline runs **only on pushes to `main`** after all CI checks pass.

```yaml
# .github/workflows/deploy.yaml
name: Deploy

on:
  push:
    branches: [main]

jobs:
  deploy-dev:
    name: Deploy to Dev
    needs: [lint, test, build, vuln-scan, secrets-scan, integration-test, docker-build]
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

      - name: Build and push image
        run: |
          docker build -t ${{ vars.REGISTRY_URL }}/my-service:${{ github.sha }} .
          docker push ${{ vars.REGISTRY_URL }}/my-service:${{ github.sha }}

      - name: Deploy to dev
        uses: azure/k8s-deploy@v5
        with:
          namespace: dev
          manifests: kubernetes/
          images: ${{ vars.REGISTRY_URL }}/my-service:${{ github.sha }}

  deploy-staging:
    name: Deploy to Staging
    needs: deploy-dev
    runs-on: ubuntu-latest
    environment: staging   # Requires manual approval gate
    steps:
      # ... same pattern as dev

  deploy-prod:
    name: Deploy to Production
    needs: deploy-staging
    runs-on: ubuntu-latest
    environment: production  # Requires 2 approvals
    steps:
      # ... same pattern as staging
```

---

## Deployment Environments

| Environment | Trigger | Approval | Migration | Notes |
|---|---|---|---|---|
| `dev` | Push to `main` | Automatic | Automatic | Mirrors staging config |
| `staging` | Push to `main` | 1 approver | Automatic | Production-like data |
| `production` | Push to `main` | 2 approvers | Manual gate | Real user data |

### GitHub Environments Setup

Configure each environment in GitHub repository settings:

1. **dev:** No protection rules.
2. **staging:** Required reviewers: 1 Platform Engineer.
3. **production:** Required reviewers: 2 (Lead Engineer + Platform Engineer). Deployment branches: `main` only.

---

## Release Process & Semantic Versioning

RBKL uses [Semantic Versioning 2.0](https://semver.org/): `MAJOR.MINOR.PATCH`.

| Version Part | When to Increment |
|---|---|
| `MAJOR` | Breaking API change |
| `MINOR` | New feature, backwards-compatible |
| `PATCH` | Bug fix, security patch, backwards-compatible |

### Creating a Release

```bash
# 1. Create and push a tag
git tag -a v1.2.3 -m "Release v1.2.3: add user export endpoint"
git push origin v1.2.3
```

The release workflow automatically:
1. Creates a GitHub Release with auto-generated changelog from commit messages.
2. Builds and pushes a versioned Docker image.
3. Deploys to production after approval.

### Commit Message Convention

Use [Conventional Commits](https://www.conventionalcommits.org/) to enable automated changelog generation:

```
feat: add user export endpoint
fix: correct pagination cursor encoding
chore: update pgx to v5.5.0
docs: update API design section
refactor: extract JWT validation to separate package
```

| Prefix | SemVer impact |
|---|---|
| `feat` | MINOR |
| `fix` | PATCH |
| `feat!` or `BREAKING CHANGE:` | MAJOR |
| `chore`, `docs`, `refactor`, `test` | None |

---

## Rollback Procedures

### Immediate Rollback (< 5 min)

Kubernetes deployments use rolling updates. Rollback with:

```bash
kubectl rollout undo deployment/my-service -n production
```

### Planned Rollback

1. Identify the last stable image tag in the container registry.
2. Update the Kubernetes manifest to the previous image.
3. Create a PR with the change and fast-track the approval.
4. If a database migration was involved, run the `down` migration first.

**Rule: Never rollback to a version that is incompatible with the current database schema.** Always ensure backward compatibility or run the down migration before rolling back the application.

---

## `golangci-lint` Configuration

Every repository includes a `.golangci.yml` configuration:

```yaml
linters:
  enable:
    - errcheck       # Check for unchecked errors
    - govet          # go vet checks
    - staticcheck    # Static analysis
    - gosimple       # Simplification suggestions
    - ineffassign    # Detect ineffectual assignments
    - unused         # Detect unused code
    - gosec          # Security linter
    - misspell       # Spelling mistakes
    - gofmt          # Format check
    - goimports      # Import order check
    - revive         # General linter
    - noctx          # Detect HTTP calls without context

linters-settings:
  gosec:
    excludes:
      - G104  # Unhandled errors in defer — covered by errcheck

issues:
  exclude-rules:
    - path: "_test.go"
      linters:
        - gosec     # Test files can use insecure patterns (e.g., hardcoded test credentials)
```

---

_[← Dependency Management](dependency-management.md) · [Back to README](../README.md)_
