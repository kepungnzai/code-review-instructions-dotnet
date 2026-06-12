# Execution Instructions & Strict Rules for Go Development

When generating, reviewing, or modifying Go code, CI/CD pipelines, or Docker artifacts, you MUST strictly adhere to the following rules. Deviating from these rules is considered a failure.

---

## 1. Project Layout Enforcement (Standard Go Project Layout)

All Go code MUST follow the [Standard Go Project Layout](https://github.com/golang-standards/project-layout):

```
/
├── cmd/                    # Main applications (one folder per binary)
│   └── <appname>/
│       └── main.go
├── internal/               # Private application code (enforced by Go toolchain)
│   ├── api/                # HTTP/gRPC handlers
│   ├── service/            # Business logic
│   ├── repository/         # Data access layer
│   └── config/             # Configuration loading
├── pkg/                    # Public library code (intentionally exposed)
├── api/                    # OpenAPI/Swagger specs, proto files
├── configs/                # YAML/TOML config templates
├── deployments/            # Docker, k8s, helm, terraform
├── scripts/                # Build, lint, generate scripts
├── test/                   # Integration & e2e test data
├── .github/workflows/      # CI/CD pipelines
├── Dockerfile              # Multi-stage production build
├── Makefile                # Standardized commands (see Section 7)
├── go.mod
├── go.sum
└── README.md
```

**STRICT RULE:** Business logic MUST live under `internal/`. Never put business code directly in `cmd/<appname>/main.go`. The `main.go` file should only wire dependencies, parse config, build the logger, and start the server. It should be < 50 lines.

---

## 2. Go Module & Dependency Management

- The module name MUST be a fully-qualified import path (e.g., `github.com/<org>/<repo>`).
- All dependencies MUST be pinned in `go.mod` and verified in `go.sum`. No floating versions in CI.
- Use `go mod tidy` before commit. CI MUST run `go mod verify`.
- Direct dependencies MUST be minimal. Prefer the standard library first, then well-maintained, widely-adopted libraries (e.g., `chi`, `cobra`, `viper`, `slog`, `pgx`, `sqlc`).
- **NEVER** add a dependency for something the standard library does well (e.g., don't pull in a JSON library — use `encoding/json`).

---

## 3. Coding Standards (Non-Negotiable)

### 3.1 Naming
- **Package names:** short, lowercase, singular, no underscores, no stutter (e.g., `http.Server`, not `httpserver.HTTPServer`).
- **Variables:** camelCase, short scope = short name (e.g., `i` for index, `r` for `*http.Request`).
- **Exported identifiers:** `PascalCase` with descriptive names. No Hungarian notation.
- **Acronyms:** all caps (e.g., `URL`, `HTTP`, `ID`, `JSON`, `API`). `userId` is wrong; `userID` is correct.

### 3.2 Functions & Methods
- Functions should do ONE thing. If the name contains "and", split it.
- Receivers: short, consistent (1–2 letters), never `this` or `self`.
- Constructor functions: `New<Type>` (e.g., `NewService(deps...)`).
- Max function length: ~50 lines. Beyond that, refactor.

### 3.3 Error Handling
- **Never** use `_` to discard an error. If you must, add a comment explaining why.
- Always return errors; never `panic` in non-init code paths.
- Wrap errors with context: `return fmt.Errorf("opening config file %q: %w", path, err)`.
- Define sentinel errors at package level: `var ErrNotFound = errors.New("not found")`.
- Use `errors.Is` and `errors.As` for comparison. Do NOT compare error strings.
- Custom error types implement `Error() string` and may carry context (use `&MyError{...}`).

### 3.4 Context Propagation
- `context.Context` is always the **first parameter** of any function that does I/O, blocking work, or may be cancelled.
- Never store a context inside a struct field.
- Respect cancellation: every blocking call must be interruptible via context.

### 3.5 Concurrency Rules
- Synchronization primitives (`sync.Mutex`, `sync.WaitGroup`, etc.) MUST be passed by pointer, never by value.
- Channel buffer size: 0 (unbuffered) by default, or a documented, justified size. Never 1 without reason.
- Do not start goroutines without a clear owner that knows when to stop them.
- Use `errgroup.Group` for parallel work with error propagation.
- Data races are blocking issues — the `go test -race` flag MUST pass in CI.

### 3.6 Logging
- Use the standard library `log/slog` (Go 1.21+) for structured logging. If Go < 1.21, use `slog` with a JSON handler or `zerolog`/`zap`.
- Logger MUST be injected via dependency injection, not a global package variable.
- Never log secrets, passwords, tokens, PII, or full request/response bodies containing such data.
- Log levels: `DEBUG` for verbose, `INFO` for state changes, `WARN` for recoverable, `ERROR` for failures, `FATAL` reserved for app-shutdown (use sparingly).

---

## 4. Testing Standards (Mandatory)

### 4.1 The Test Pyramid
1. **Unit tests** (80% of tests): Test a single function/method in isolation. Use table-driven tests.
2. **Integration tests** (15%): Test interaction with real dependencies (DB, cache, message queue). Use build tags `//go:build integration` to separate them.
3. **End-to-end tests** (5%): Test the full HTTP/gRPC API contract. Use `httptest.NewServer` or testcontainers.

### 4.2 Test File Conventions
- Test files live next to the code they test: `foo.go` → `foo_test.go`.
- Test functions: `Test<Unit>_<Scenario>_<ExpectedResult>` (e.g., `TestService_GetUser_NotFound_ReturnsError`).
- Use `t.Run("subtest name", func(t *testing.T) {...})` for table-driven subtests.
- Use `t.Parallel()` where safe. Do NOT parallelize tests that share state.

### 4.3 Required Test Patterns
- **Table-driven tests** for all functions with multiple input/output cases.
- **Subtests** with `t.Run()` for clear failure reporting.
- **Test helpers** with `t.Helper()` for shared assertions.
- **Mocks via interfaces**, NOT generated mocks for everything. Define a small interface in the *consumer* package, and provide a hand-rolled fake. Use `gomock`/`mockery` only at API boundaries.
- **Benchmarks** for performance-critical code: `BenchmarkXxx(b *testing.B)`.
- **Example tests** for exported functions that benefit from documentation: `ExampleXxx()`.

### 4.4 Coverage Requirements
- **CI MUST enforce ≥ 80% line coverage** on changed files (use `go test -coverprofile=cover.out && go tool cover -func=cover.out`).
- Coverage of new code must not decrease the overall project coverage.
- 100% coverage is NOT the goal — meaningful assertions are. Don't write tests that just call functions without checking behavior.

### 4.5 Race & Static Analysis
- CI MUST run `go test -race ./...`.
- CI MUST run `go vet ./...` and `staticcheck ./...`.
- CI MUST run `golangci-lint run` with the team's `.golangci.yml` configuration.

---

## 5. GitHub Actions CI/CD Pipeline (Required)

You MUST generate a `.github/workflows/` directory with the following workflow files:

### 5.1 `ci.yml` — Continuous Integration
Triggered on `push` to any branch and on `pull_request` to `main`. Steps:
1. Checkout code (`actions/checkout@v4`).
2. Set up Go (`actions/setup-go@v5`) with **pinned Go version** and **module cache**.
3. Verify modules: `go mod download && go mod verify`.
4. Format check: `gofmt -l .` and `goimports -l .` — fail if any output.
5. Lint: `golangci-lint run --timeout 5m`.
6. Static analysis: `go vet ./...` and `staticcheck ./...`.
7. Test with race detector and coverage: `go test -race -coverprofile=coverage.out -covermode=atomic ./...`.
8. Upload coverage to Codecov or similar.
9. Build all binaries: `go build ./cmd/...`.
10. Use **build matrix** for Go versions (e.g., 1.22, 1.23) and OS (ubuntu-latest, macos-latest) where applicable.

### 5.2 `build-and-push.yml` — Container Image Build & Push
Triggered on `push` to `main` and on `version tags` (e.g., `v*.*.*`). Steps:
1. Checkout code.
2. Set up Docker Buildx (`docker/setup-buildx-action@v3`).
3. Log in to container registry (GHCR or ECR) using OIDC or short-lived credentials.
4. **Multi-platform build:** `docker buildx build --platform linux/amd64,linux/arm64 ...`.
5. **Reproducible labels:** include `org.opencontainers.image.source`, `org.opencontainers.image.revision`, `org.opencontainers.image.created`, `org.opencontainers.image.version`.
6. **Cache:** Use `docker/build-push-action@v5` with `cache-from: type=gha` and `cache-to: type=gha,mode=max`.
7. **Provenance/SBOM:** `provenance: true` and SBOM generation enabled.
8. Push with semantic tags: `<sha>`, `<branch>-<sha>`, `latest` (only for `main`).

### 5.3 `release.yml` — Release Automation
Triggered on `v*.*.*` tags. Steps:
1. Build binaries for multiple OS/arch (linux/darwin/windows × amd64/arm64).
2. Generate SHA256 checksums.
3. Create GitHub Release with `softprops/action-gh-release@v2`, attaching binaries and checksums.
4. Auto-generate changelog via `release-drafter` or `conventional-changelog`.

### 5.4 `codeql.yml` (optional but recommended)
GitHub CodeQL static analysis for security scanning on every push and weekly schedule.

---

## 6. Dockerfile Standards (Multi-Stage, Distroless)

You MUST generate a production-grade, multi-stage `Dockerfile` at the project root:

```dockerfile
# syntax=docker/dockerfile:1.7

# ---------- Stage 1: Build ----------
FROM golang:1.23-alpine AS build
WORKDIR /src

# Cache dependencies separately
COPY go.mod go.sum ./
RUN go mod download && go mod verify

# Build with reproducible flags
COPY . .
RUN CGO_ENABLED=0 GOOS=linux GOARCH=amd64 \
    go build -trimpath -ldflags="-s -w -buildid=" \
    -o /out/app ./cmd/<appname>

# ---------- Stage 2: Final (Distroless) ----------
FROM gcr.io/distroless/static-debian12:nonroot
COPY --from=build /out/app /app
USER nonroot:nonroot
EXPOSE 8080
ENTRYPOINT ["/app"]
```

**STRICT RULES:**
- **Multi-stage build is mandatory.** No compilers, package managers, or source code in the final image.
- **Distroless or scratch final image is mandatory.** Alpine is acceptable only if a C library is required (CGO_ENABLED=0 preferred).
- **Run as non-root user** (`USER nonroot:nonroot`).
- **Pin base image by digest** in production (`@sha256:...`).
- **Use `.dockerignore`** to exclude `.git`, `*.md`, `test/`, `coverage.out`, etc.
- **Pass secrets via BuildKit** (`RUN --mount=type=secret,id=...`) — never `ARG` for secrets.
- **Multi-platform builds** (linux/amd64, linux/arm64) MUST be supported.
- **HEALTHCHECK** is required for services: `HEALTHCHECK --interval=30s --timeout=3s CMD ["/app", "-healthcheck"]` (the app must implement a `-healthcheck` flag).

You MUST also generate a `.dockerignore` file at the project root.

---

## 7. Makefile (Standardized Commands)

You MUST generate a `Makefile` at the project root with the following targets:

```makefile
.PHONY: help tidy build test test-race test-coverage lint format run docker-build docker-run clean

BINARY_NAME := app
GO          := go
DOCKER      := docker

help: ## Show this help
	@grep -E '^[a-zA-Z_-]+:.*?## .*$$' $(MAKEFILE_LIST) | awk 'BEGIN {FS = ":.*?## "}; {printf "\033[36m%-20s\033[0m %s\n", $$1, $$2}'

tidy: ## Run go mod tidy
	$(GO) mod tidy

build: ## Build the binary
	CGO_ENABLED=0 $(GO) build -trimpath -ldflags="-s -w" -o bin/$(BINARY_NAME) ./cmd/$(BINARY_NAME)

test: ## Run unit tests
	$(GO) test -race -count=1 ./...

test-coverage: ## Run tests with coverage report
	$(GO) test -race -coverprofile=coverage.out -covermode=atomic ./...
	$(GO) tool cover -html=coverage.out -o coverage.html

lint: ## Run golangci-lint
	golangci-lint run --timeout 5m ./...

format: ## Run gofmt and goimports
	gofmt -s -w .
	goimports -w .

run: ## Run the application
	$(GO) run ./cmd/$(BINARY_NAME)

docker-build: ## Build the Docker image
	$(DOCKER) buildx build --platform linux/amd64,linux/arm64 -t $(BINARY_NAME):local .

docker-run: ## Run the Docker image locally
	$(DOCKER) run --rm -p 8080:8080 $(BINARY_NAME):local

clean: ## Clean build artifacts
	rm -rf bin/ coverage.out coverage.html
```

---

## 8. Configuration & Secrets

- All configuration MUST be loaded from environment variables (12-factor app).
- Support a config struct loaded via a library like `viper`, `envconfig`, or `kelseyhightower/envconfig`.
- **NEVER** hardcode secrets, connection strings, or credentials.
- All secrets MUST be loaded from env vars or a secret manager (Vault, AWS Secrets Manager, GCP Secret Manager).
- Provide a `.env.example` file (NOT `.env`) documenting every required environment variable.
- Validate configuration at startup and FAIL FAST with a clear error message if required values are missing or malformed. Never start the app in a half-configured state.
- Use `time.Duration` for any timeout-related env vars (e.g., `HTTP_READ_TIMEOUT=10s`), not raw integers.

---

## 9. Observability Requirements

- Structured JSON logging to **stdout** (collected by the platform — never log to files in containers).
- Prometheus `/metrics` endpoint exposed and scraped.
- OpenTelemetry tracing for inbound HTTP/gRPC, outbound HTTP clients, database calls, and message queue producers/consumers.
- Distinguish **liveness** (`/healthz`), **readiness** (`/readyz`), and a startup probe if needed. Readiness MUST check downstream dependencies (DB, cache, message queue).
- Log a `trace_id` and `span_id` on every log line via `slog.With(...)`.
- Emit RED metrics (Rate, Errors, Duration) for every HTTP route and gRPC method.

---

## 10. Graceful Shutdown & Lifecycle

- Trap `SIGINT` and `SIGTERM` via `signal.NotifyContext`.
- On signal: stop accepting new requests, wait for in-flight requests to complete (with a hard deadline, e.g., 30s), close DB pools, flush logs, then exit with code 0.
- If the shutdown deadline is exceeded, log a warning and exit with a non-zero code.
- The HTTP server MUST be configured with all four timeouts: `ReadTimeout`, `ReadHeaderTimeout`, `WriteTimeout`, `IdleTimeout`.

---

## 11. API Design

- Use **RESTful resource naming** for HTTP APIs (plural nouns: `/users`, `/users/{id}`).
- Return appropriate HTTP status codes: 200, 201, 204, 400, 401, 403, 404, 409, 422, 429, 500.
- Use **problem+json** (RFC 7807) for error responses.
- Version APIs explicitly: `/v1/users`, `/v2/users`.
- Validate every request body and query parameter. Reject early with 400.
- Use snake_case in JSON, never camelCase. (Go struct fields use PascalCase and map via JSON tags.)
- Document every public API with OpenAPI 3.1 spec in the `api/` directory.
- For gRPC, keep `.proto` files in `api/proto/` and generate Go code via `buf` or `protoc-gen-go`.

---

## 12. Database Access Discipline

- Always use `context.Context` for queries: `db.QueryContext(ctx, ...)`.
- Set a query timeout via context (e.g., 3 seconds for OLTP queries).
- Use prepared statements or `sqlc`-generated code for hot queries.
- Wrap repository methods in a `BeginTx` → `Commit/Rollback` pattern. Never leak transactions.
- Add DB indexes for any field used in `WHERE`, `ORDER BY`, or `JOIN ON` clauses.
- Use database migrations (`golang-migrate`) checked into the repo, applied automatically in CI/CD.

---

## 13. Dependency Hygiene

- Run `go mod tidy` and `go mod verify` before every commit.
- Run `govulncheck ./...` in CI to scan for known vulnerabilities.
- Pin GitHub Actions to a full version SHA in production (e.g., `actions/checkout@b4ffde65f46336ab88eb53be808477a3936bae11`), not just `@v4`.
- Renovate or Dependabot MUST be enabled to keep dependencies and actions up to date automatically.
- Reject any dependency that has not had a release in the last 18 months OR is unmaintained.

---

## 14. Commit & Branching Standards

- Branch naming: `<type>/<ticket-id>-<short-description>` (e.g., `feat/PROJ-123-add-user-registration`).
- Commit messages follow **Conventional Commits**:
  - `feat:`, `fix:`, `chore:`, `docs:`, `style:`, `refactor:`, `perf:`, `test:`, `build:`, `ci:`
  - Example: `feat(api): add rate limiting to /v1/users endpoint`
- Squash-merge feature branches to `main` with a single conventional commit.
- Every PR MUST reference a ticket/issue ID.
- `main` branch is protected: requires PR, 1+ review, all CI checks green, no direct pushes.

---

## 15. Pre-Commit Checklist (Self-Audit)

Before declaring work complete, the agent MUST verify:

- [ ] `gofmt -s -l .` produces no output
- [ ] `goimports -l .` produces no output
- [ ] `go vet ./...` passes
- [ ] `staticcheck ./...` passes
- [ ] `golangci-lint run` passes with zero issues
- [ ] `go test -race -count=1 ./...` passes locally
- [ ] `go test -coverprofile=cover.out ./...` shows ≥ 80% on changed files
- [ ] `govulncheck ./...` shows no unfixed vulnerabilities
- [ ] New code has corresponding unit tests (table-driven where applicable)
- [ ] Integration tests use the `//go:build integration` tag
- [ ] No new dependencies added without justification in the PR description
- [ ] `Dockerfile` builds successfully: `docker buildx build --platform linux/amd64,linux/arm64 .`
- [ ] The final Docker image runs as non-root and is based on distroless or scratch
- [ ] `/healthz` and `/readyz` endpoints respond correctly
- [ ] No secrets, tokens, or credentials in the diff
- [ ] `.golangci.yml` is committed and passing
- [ ] `Makefile` targets (`make help`, `make test`, `make build`, `make lint`) all succeed
- [ ] CI workflow files validate locally with `act` (or `actionlint`)
- [ ] OpenAPI/Proto spec is updated for any API changes
- [ ] `CHANGELOG.md` (or release notes) is updated for user-facing changes
- [ ] README is updated if commands, env vars, or setup steps changed

---

## Output Format for the Agent

When generating a new Go service, the agent MUST output, at minimum:

1. The full directory tree of the generated project.
2. The complete contents of every file created (with `path/to/file.ext` headers).
3. A `README.md` containing: project description, prerequisites, `make` quickstart, env-var reference, and a "How to deploy" section.
4. A summary of decisions made and the trade-offs (e.g., why distroless over alpine, why chi over gin).

When reviewing or modifying existing Go code, the agent MUST output:

1. A diff of every changed file in unified format.
2. A categorized list of issues: `BLOCKING`, `WARNING`, `OPTIMIZATION`, `NIT`.
3. For each issue: file path, line number, the rule violated (with link to this document or a referenced best-practice doc), and a suggested fix in code.
4. A final verdict: `[APPROVED]`, `[APPROVED WITH WARNINGS]`, or `[REJECTED — CHANGES REQUIRED]`.
