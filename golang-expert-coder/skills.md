# Agent Skills & Knowledge Base: Go Backend Engineering

You possess expert-level proficiency in the following domains. Use this knowledge to make authoritative recommendations and write idiomatic, production-grade code.

---

## 1. Go Language & Standard Library

### 1.1 Core Language
- **Generics (Go 1.18+):** Use generics sparingly and only for type-safe, reusable data structures and functions (e.g., `Set[T]`, `Map[K, V]`, generic `Filter/Map/Reduce`). Do NOT use generics to replace interface-based polymorphism where interfaces are more natural.
- **Type parameters constraints:** Use `cmp.Ordered`, `cmp.Comparable` from the `golang.org/x/exp/constraints` or `cmp` packages.
- **Defer:** Defer is for cleanup (close files, unlock mutexes). It is NOT for control flow. The arguments are evaluated immediately.
- **Slices vs Arrays:** Slices (`[]T`) are the default. Arrays (`[N]T`) are rarely used directly; they're the underlying storage of slices.
- **Maps:** Read-only maps are safe for concurrent reads. Writes require synchronization. `len(m)` on a nil map returns 0; reading from a nil map returns the zero value; writing panics.
- **Pointers vs Values:** Use pointer receivers for large structs, mutable state, or when consistency with other methods demands it. Use value receivers for small, immutable types.
- **Type assertions:** Use the comma-ok form: `v, ok := x.(*Type)`. Never do a bare assertion in non-trivial code.

### 1.2 Concurrency Primitives
- `sync.Mutex` / `sync.RWMutex` for protecting shared state.
- `sync.WaitGroup` for waiting for a group of goroutines (prefer `errgroup` for parallel work with error propagation).
- `sync.Once` for one-time initialization (e.g., singleton init).
- `sync.Pool` for reducing GC pressure on frequently allocated objects.
- `context.Context` for cancellation, deadlines, and request-scoped values.
- `atomic` package for lock-free counters and flags (`atomic.Int64`, `atomic.Pointer[T]`).
- Channels: use for signaling and ownership transfer, NOT as a default communication mechanism. A channel with a buffer size of 1 must be justified.

### 1.3 Standard Library Mastery
- `net/http` — full server with `http.ServeMux` (Go 1.22+ supports method-based routing), middleware patterns, request context.
- `encoding/json` — struct tags (`json:"name,omitempty"`), `json.RawMessage`, custom `MarshalJSON`/`UnmarshalJSON`.
- `database/sql` and `pgx` for PostgreSQL. `sqlx` for query scanning. `sqlc` for type-safe generated code.
- `testing` — table-driven tests, subtests, helpers, benchmarks, examples, fuzz tests (`FuzzXxx`).
- `flag` or `cobra` for CLI parsing.
- `log/slog` for structured logging (Go 1.21+).
- `os/signal` and `context.WithCancel` for graceful shutdown.
- `io` and `bufio` for streaming I/O.
- `time` — always use `time.Time` and `time.Duration`. Never use `int` for seconds.
- `errors`, `fmt.Errorf` with `%w`, `errors.Is`, `errors.As`, `errors.Join` (Go 1.20+).

---

## 2. Web Frameworks & Routing

You are authorized to recommend and use the following proven frameworks:
- **Standard library `net/http` + `http.ServeMux`** (preferred for new Go 1.22+ services).
- **`github.com/go-chi/chi/v5`** — lightweight, idiomatic, composable router.
- **`github.com/gin-gonic/gin`** — only when the team has prior Gin experience.
- **`github.com/labstack/echo/v4`** — feature-rich alternative.
- **`google.golang.org/grpc`** — for service-to-service RPC.

**Anti-patterns to reject:**
- `gorilla/mux` is in maintenance mode. Avoid for new projects.
- Custom HTTP routers when the standard library or chi suffices.
- Heavyweight "batteries-included" frameworks that fight Go's simplicity.

---

## 3. Databases & Persistence

### 3.1 SQL Databases
- **`pgx` (PostgreSQL)** is the preferred driver — high performance, native types, `pgxpool` for pooling.
- **`database/sql` + `lib/pq` or `mysql` driver** for cross-database projects.
- **`sqlc`** for generating type-safe Go from raw SQL — strongly recommended.
- **`sqlx`** for ergonomic struct scanning when sqlc is not used.
- **`ent` or `gorm`** — accept only if the team has strong ORM experience; ORMs are not idiomatic Go and can hide performance issues. Prefer sqlc.

**Connection pool rules:**
- Set `MaxOpenConns`, `MaxIdleConns`, `ConnMaxLifetime` based on workload.
- Always use context with query execution: `db.QueryContext(ctx, ...)`.
- Use prepared statements for hot queries.
- Set query timeouts via context.

### 3.2 NoSQL & Caching
- **Redis:** `github.com/redis/go-redis/v9` for caching, pub/sub, distributed locks.
- **MongoDB:** `go.mongodb.org/mongo-driver` — only when document model is a clear fit.
- **BadgerDB / BoltDB:** for embedded key-value stores.

### 3.3 Migrations
- **`golang-migrate/migrate`** — file-based SQL migrations, runs in CI.
- **`pressly/goose`** — alternative with Go-based migrations.

---

## 4. Observability

### 4.1 Logging
- **`log/slog`** (Go 1.21+) with JSON handler for production.
- **`rs/zerolog`** or **`uber-go/zap`** for high-performance JSON logging.
- Log structure: `time`, `level`, `msg`, `service`, `trace_id`, `span_id`, plus domain fields.
- NEVER log at the `Info` level in tight loops. Use sampling.

### 4.2 Metrics
- **`prometheus/client_golang`** — Prometheus metrics with `/metrics` endpoint.
- Use `promauto` for clean registration.
- RED metrics: Rate, Errors, Duration for every endpoint.
- USE metrics: Utilization, Saturation, Errors for resources.
- Business metrics: domain-specific counters and histograms.

### 4.3 Tracing
- **`go.opentelemetry.io/otel`** — OpenTelemetry SDK.
- Instrument: HTTP server, HTTP client, gRPC, database drivers (via `otelpgx`, `otelgorm`).
- Always propagate `traceparent` headers. Use `otelgin`, `otelhttp`, or `otelmiddleware` for automatic instrumentation.
- Sample appropriately: 100% in dev, 1–10% in production, 100% for errors.

### 4.4 Health Checks
- `/healthz` — liveness (process is alive).
- `/readyz` — readiness (dependencies are reachable).
- `/metrics` — Prometheus scrape endpoint.

---

## 5. Testing Ecosystem

### 5.1 Test Helpers & Libraries
- **`github.com/stretchr/testify`** — `assert`, `require`, `suite`, `mock` packages. Use `require` for setup failures, `assert` for verification.
- **`github.com/onsi/ginkgo` / `gomega`** — BDD-style; accept only if the team prefers it.
- **`github.com/golang/mock`** or **`github.com/uber-go/mock`** — generated mocks. Prefer hand-rolled fakes for internal interfaces.
- **`github.com/alicebob/miniredis/v2`** — in-memory Redis for tests.
- **`github.com/ory/dockertest`** or **`testcontainers/testcontainers-go`** — spin up real DB/cache containers for integration tests.
- **`github.com/h2non/gock`** or **`go.uber.org/mock`** — HTTP mocking.
- **`net/http/httptest`** — standard library for testing HTTP handlers.

### 5.2 Required Test Discipline
- `go test -race ./...` must always pass.
- `go test -shuffle=on` to catch hidden test dependencies.
- `go test -count=1` in CI to disable test caching.
- Fuzz tests (`FuzzXxx`) for parsers, validators, and any code taking untrusted input.

---

## 6. Build, Lint, and Static Analysis

### 6.1 Linters (run all in CI)
- **`golangci-lint`** — the standard meta-linter. Config in `.golangci.yml`.
- **`go vet`** — built-in static analysis.
- **`staticcheck`** — advanced analysis.
- **`gofmt -s`** — formatting (run in CI as a check).
- **`goimports`** — import grouping and ordering.

### 6.2 Recommended `.golangci.yml` Linters
Enable at minimum: `errcheck`, `gosimple`, `govet`, `ineffassign`, `staticcheck`, `unused`, `gocritic`, `gofmt`, `goimports`, `misspell`, `unconvert`, `unparam`, `bodyclose`, `contextcheck`, `nilerr`, `noctx`, `predeclared`, `revive`, `wrapcheck`.

### 6.3 Build Tooling
- **Go modules** with `go.mod` and `go.sum` (always committed).
- **`go build -trimpath`** for reproducible builds.
- **Build flags:** `-ldflags="-s -w"` to strip debug info, `-buildid=` to remove build ID.
- **`govulncheck`** for known vulnerability scanning — run in CI.

---

## 7. Security

- **`golang.org/x/crypto`** for bcrypt, scrypt, argon2.
- **`github.com/golang-jwt/jwt/v5`** for JWT — validate issuer, audience, expiry, signing algorithm strictly.
- **OWASP Go Secure Coding Practices:** input validation, output encoding, parameterized queries (no string concatenation in SQL).
- **`gosec`** — security-focused linter.
- **Secrets:** never log, never commit, use Vault or cloud secret managers.
- **TLS:** always use TLS 1.2+; configure cipher suites explicitly; pin certificates when required.
- **`net/http` server timeouts:** `ReadTimeout`, `WriteTimeout`, `IdleTimeout`, `ReadHeaderTimeout` — all must be set.
- **Rate limiting:** use `golang.org/x/time/rate` for in-memory, or a Redis-backed limiter for distributed.

---

## 8. Resilience Patterns

- **Circuit breaker:** `github.com/sony/gobreaker` or `github.com/failsafe-go/failsafe-go`.
- **Retry with backoff:** `github.com/cenkalti/backoff/v4` — never retry non-idempotent operations without safeguards.
- **Timeouts:** every outbound call MUST have a timeout (via context or `http.Client.Timeout`).
- **Bulkhead:** isolate failures using `errgroup` with `SetLimit(n)`.
- **Graceful shutdown:** on SIGINT/SIGTERM, stop accepting new requests, drain in-flight, close DB pools, flush logs.

---

## 9. Recommended GitHub Actions

You are authorized to use and configure the following proven actions:
- `actions/checkout@v4` — repository checkout.
- `actions/setup-go@v5` — Go toolchain with module caching.
- `actions/cache@v4` — additional caching.
- `golangci/golangci-lint-action@v6` — linting.
- `actions/upload-artifact@v4` — build artifacts.
- `docker/setup-qemu-action@v3`, `docker/setup-buildx-action@v3`, `docker/login-action@v3`, `docker/metadata-action@v5`, `docker/build-push-action@v5` — container build & push.
- `aquasecurity/trivy-action@master` — container & filesystem vulnerability scanning.
- `github/codeql-action@v3` — code scanning.
- `sonarsource/sonarqube-scan-action@master` — SonarQube analysis (optional).
- `softprops/action-gh-release@v2` — GitHub releases.
- `codecov/codecov-action@v4` — coverage reporting.

---

## 10. Recommended Docker Base Images

- **`gcr.io/distroless/static-debian12:nonroot`** — preferred for pure Go binaries (CGO_ENABLED=0).
- **`gcr.io/distroless/base-debian12:nonroot`** — if cgo or libc is required.
- **`cgr.dev/chainguard/static`** — alternative hardened minimal image.
- **`alpine`** — acceptable when image size matters more than minimalism; always pin to a specific version (e.g., `alpine:3.20`).
- **`scratch`** — only for the most hardened cases; you must supply CA certificates and timezone data yourself.
- **Always pin by digest (`@sha256:...`) in production Dockerfiles.**

---

## 11. DevOps & Cloud-Native Best Practices

- **12-Factor App methodology** — strict separation of config from code.
- **Health probes** — separate liveness, readiness, and startup probes for Kubernetes.
- **Graceful shutdown** — handle SIGTERM, drain in-flight requests, close DB connections.
- **Pod Security Standards:** run as non-root, read-only root filesystem, drop all capabilities.
- **Resource requests and limits** MUST be set in Kubernetes manifests.
- **Horizontal Pod Autoscaler (HPA)** configured based on CPU and custom metrics.
- **PodDisruptionBudget (PDB)** for availability during rollouts.
- **Observability:** logs to stdout in JSON format (collected by the platform), metrics scraped by Prometheus, traces via OTLP.
- **GitOps:** infrastructure and app config managed via Git (ArgoCD, Flux).
- **Progressive delivery:** canary, blue/green, or feature flags for new releases.

---

## 12. Anti-Patterns to Always Reject

1. **Panicking in production code paths** to handle errors.
2. **Global mutable state** (singletons, package-level variables holding config).
3. **Ignoring errors with `_`.**
4. **String concatenation in SQL queries** (SQL injection risk).
5. **Using `init()` for anything more than registration.**
6. **Premature optimization** without profiling (`pprof`, benchmarks).
7. **Adding dependencies for trivial functionality** the standard library provides.
8. **Reflexive interface abstraction** ("interface everything" — Go interfaces are for consumers, not producers).
9. **Mixing business logic with HTTP handlers.**
10. **Using `reflect` in hot paths.**
11. **Storing `context.Context` in a struct field.**
12. **Using `time.Sleep` for synchronization in tests** — use channels, condition variables, or polling helpers.
13. **Skipping `defer` for resource cleanup.**
14. **Building Docker images without multi-stage builds.**
15. **Shipping Docker images that run as root.**
