# Execution Instructions & Strict Rules for Rust Development

When generating, reviewing, or modifying Rust code, CI/CD pipelines, or Docker artifacts, you MUST strictly adhere to the following rules. Deviating from these rules is considered a failure.

---

## 1. Project Layout Enforcement

All Rust crates MUST follow the standard Cargo workspace layout. For services, use a **Cargo workspace** with one or more member crates:

```
/
├── Cargo.toml                 # Workspace root (members = [...])
├── Cargo.lock                 # ALWAYS committed for binaries
├── rust-toolchain.toml        # Pinned Rust toolchain
├── clippy.toml                # Lint configuration
├── rustfmt.toml               # Format configuration
├── deny.toml                  # cargo-deny config
├── crates/
│   ├── api/                   # HTTP/gRPC handlers (axum, actix-web, tonic)
│   ├── core/                  # Business logic, domain types
│   ├── storage/               # Persistence (sqlx, diesel, redis)
│   └── telemetry/             # Logging, metrics, tracing
├── apps/
│   └── <appname>/
│       ├── Cargo.toml
│       └── src/main.rs        # Thin binary — wires dependencies
├── proto/                     # Protobuf definitions
├── migrations/                # SQLx / refinery migrations
├── deploy/                    # k8s, helm, terraform
├── .github/workflows/         # CI/CD pipelines
├── Dockerfile                 # Multi-stage production build
├── .dockerignore
├── Makefile.toml              # Or Makefile — standardized commands (Section 7)
└── README.md
```

**STRICT RULES:**
- A library crate's `src/lib.rs` exposes the public API; `src/main.rs` (if present) MUST be a thin binary that only wires dependencies, parses config, builds the logger, and starts the server. `main.rs` should be < 50 lines.
- Each crate MUST have a `#![deny(missing_docs)]` (or `#![warn(missing_docs)]`) on the public API.
- Use `pub` sparingly. Default to private; expose only what is consumed externally.
- `Cargo.lock` MUST be committed for binary crates and workspaces that produce binaries. It MUST be `.gitignore`d for pure library crates published to crates.io (Cargo will generate it on publish).

---

## 2. Cargo & Dependency Management

- The workspace root `Cargo.toml` MUST define a `[workspace.dependencies]` table; member crates MUST reference dependencies as `dep = { workspace = true }`. **Never duplicate version strings across crates.**
- All dependencies MUST be pinned to exact versions in `Cargo.lock`. Use `cargo update --precise <version>` when upgrading.
- Use `cargo add` for adding dependencies (writes the correct version).
- Direct dependencies MUST be minimal. Prefer the standard library first, then well-maintained, widely-adopted crates (e.g., `tokio`, `axum`, `serde`, `tracing`, `sqlx`, `anyhow`/`thiserror`).
- **NEVER** add a dependency for something the standard library does well (e.g., don't pull in a JSON library — use `serde_json`; don't pull in a custom error crate if `thiserror` suffices; don't pull in a logging facade if `tracing` is already in your tree).
- Run `cargo update --dry-run` in CI to detect supply-chain drift; pin if a transitive crate would float.
- Use `cargo-deny` in CI to enforce: license allow-list, banned crates, duplicate versions, and advisory database (`RUSTSEC`).

---

## 3. Coding Standards (Non-Negotiable)

### 3.1 Naming
- **Crate names:** `kebab-case` in `Cargo.toml` (`my-crate`); `snake_case` in `use` statements.
- **Types, Traits, Enums:** `UpperCamelCase`.
- **Functions, methods, variables, modules:** `snake_case`.
- **Constants and statics:** `SCREAMING_SNAKE_CASE`.
- **Acronyms in `UpperCamelCase`:** treat as a single word (e.g., `HttpServer`, not `HTTPServer`). In `snake_case`, all lowercase (e.g., `http_server`).
- **Lifetimes:** short, lowercase (`'a`, `'de` for "deserialize"). Use descriptive names only when ambiguous.

### 3.2 Functions & Methods
- Functions should do ONE thing. If the name contains "and", split it.
- Methods that return `Self` with no arguments are constructors; methods that return `Self` with arguments are `new` (or fallible `try_new`/`from_*`).
- Max function length: ~50 lines. Beyond that, refactor.
- Use **builder pattern** for structs with > 3 optional fields. Use **typestate pattern** for state machines.
- Prefer `impl Trait` in return position for ergonomic public APIs; use `Box<dyn Trait>` only when heterogeneous collection is required.

### 3.3 Ownership, Borrowing & Lifetimes
- **Default to borrowing (`&T`, `&mut T`).** Clone only when ownership is genuinely required.
- Functions should take `&str` over `&String`, `&[T]` over `&Vec<T>`, `&Path` over `&PathBuf`. Use `Cow<'_, T>` when ownership is conditionally needed.
- **Never** use `Rc<RefCell<T>>` to escape the borrow checker. If you need shared ownership with interior mutability, use `Arc<Mutex<T>>` (async-aware: `tokio::sync::Mutex`) or restructure.
- `Box<T>` for heap allocation, `Rc<T>` for single-threaded shared ownership, `Arc<T>` for multi-threaded shared ownership.
- **Lifetimes MUST be elided** whenever possible. Named lifetimes are required only when:
  - Multiple input lifetimes and one is `&self`/`&mut self`.
  - Returning a reference that depends on multiple inputs.
  - A struct holds a reference.
- Avoid `static` lifetime in function signatures unless the function genuinely returns a process-wide singleton.

### 3.4 Error Handling
- **Never** use `unwrap()` or `expect()` in production code paths. Use them ONLY in tests, examples, and benchmarks.
- Library code MUST define a single crate-level error enum:
  ```rust
  #[derive(Debug, thiserror::Error)]
  pub enum Error {
      #[error("opening config file {path:?}: {source}")]
      Config { path: PathBuf, source: io::Error },
      #[error(transparent)]
      Database(#[from] sqlx::Error),
      // ...
  }
  pub type Result<T> = std::result::Result<T, Error>;
  ```
- Implement `From<ForeignError>` for ergonomic `?` propagation.
- Implement `std::error::Error + Send + Sync` (required for use across async boundaries).
- Use `anyhow::Result` ONLY in binary crates (`main.rs`) and tests. NEVER in library public APIs.
- Sentinel errors are acceptable only for `#[non_exhaustive]` enums; the canonical pattern is the dedicated error enum.
- NEVER silently swallow errors. If you genuinely need to discard one, log it via `tracing::error!` and document why.

### 3.5 Pattern Matching
- Always use exhaustive `match`. The `#[deny(unreachable_patterns)]` lint MUST be on.
- Use `if let Some(x) = opt` for single-pattern matching.
- Use `let ... else` (Rust 1.65+) for early-return guard clauses.
- NEVER use `_` to silently discard a match arm in non-test code. List each variant explicitly.
- Use `_` only as a deliberate catch-all for the discriminant (e.g., exhaustiveness of an `#[non_exhaustive]` enum).

### 3.6 Traits & Generics
- Define traits in the **consumer** crate, not the producer. (Acceptance: `Display` and `From` are in `std`; the same principle applies to your code.)
- Prefer trait bounds on generic functions over trait objects when monomorphization is acceptable.
- Use `dyn Trait` only when:
  - You need a heterogeneous collection (`Vec<Box<dyn Trait>>`).
  - You cannot afford monomorphization (binary size).
- Marker traits (`Send`, `Sync`, `Unpin`) MUST propagate correctly. If your type contains an `Rc<T>`, it is intentionally `!Send`.
- Object safety: a trait is object-safe only if all methods have `self` as `&Self`/`&mut Self`/`Box<Self>` and there are no generic methods. Use `where Self: Sized` on non-object-safe methods.

### 3.7 Concurrency & Async
- **Default to async with `tokio`** for I/O-bound services. Use `axum` or `tonic` on top.
- **Default to `rayon`** for CPU-bound data-parallel work.
- Spawned tasks MUST be cancellation-safe. Use `tokio::select!` and `tokio::task::JoinHandle` properly.
- All async functions MUST accept a `&mut tokio::time::Duration` or rely on caller-provided `tokio::time::timeout` — never spin in an infinite loop.
- Use `tokio::sync::Mutex` (not `std::sync::Mutex`) inside async code. Holding a `std::sync::MutexGuard` across an `.await` point will block the runtime and is a BUG.
- Channels: `tokio::sync::mpsc` (multi-producer, single-consumer), `tokio::sync::broadcast` (pub/sub), `crossbeam::channel` (sync), `async-channel` (alternative async).
- Data races are compile-time errors in safe Rust. `Send`/`Sync` violations are blocking issues.
- Always propagate cancellation via `tokio_util::sync::CancellationToken` or `tokio::select!` on a shutdown signal.

### 3.8 Logging & Telemetry
- Use the **`tracing`** crate for structured logging and span-based instrumentation. `log` is acceptable for `tracing-subscriber` compatibility.
- Spans: every request MUST enter a `#[instrument]`-decorated handler or wrap a `tracing::info_span!`.
- Log structure: `time`, `level`, `target`, `message`, `span`, plus domain fields.
- Logger MUST be initialized once via `tracing_subscriber::fmt::init()` or `tracing_subscriber::registry().with(...).init()` in `main.rs`.
- Never log secrets, passwords, tokens, PII, or full request/response bodies containing such data. Use `tracing::field::display` and explicit redaction.
- Log levels: `error`, `warn`, `info`, `debug`, `trace`. Default to `info` in production, controlled by `RUST_LOG`.

---

## 4. Testing Standards (Mandatory)

### 4.1 The Test Pyramid
1. **Unit tests** (80% of tests): Test a single function/method in isolation. Co-located in the same module.
2. **Integration tests** (15%): In `tests/` directory of each crate. Test the public API end-to-end.
3. **Property-based tests** (for parsers/serializers/pure functions): Use `proptest` or `quickcheck`.
4. **Doc tests** (free, mandatory): Every public API example in `///` is run as a test.
5. **End-to-end / black-box tests** (5%): Spin up the full service with `testcontainers` and exercise the binary.

### 4.2 Test File Conventions
- Unit tests live in `#[cfg(test)] mod tests` blocks within the same file.
- Integration tests live in `tests/*.rs` (each file is a separate crate).
- Test functions: `fn <unit>_<scenario>_<expected_result>` (e.g., `fn get_user_not_found_returns_err`).
- Test modules MUST have `#[allow(clippy::unwrap_used, clippy::expect_used)]` at the top — `unwrap` and `expect` are acceptable in tests.

### 4.3 Required Test Patterns
- **Table-driven tests** for all functions with multiple input/output cases. Use `rstest` (preferred) or manual:
  ```rust
  use rstest::rstest;
  #[rstest]
  #[case("hello", 5)]
  #[case("", 0)]
  fn len_returns_byte_count(#[case] input: &str, #[case] expected: usize) {
      assert_eq!(input.len(), expected);
  }
  ```
- **Property-based tests** with `proptest!` for parsers, validators, and any pure function:
  ```rust
  use proptest::prelude::*;
  proptest! {
      #[test]
      fn parse_doesnt_panic(s in ".*") {
          let _ = parser::parse(&s);
      }
  }
  ```
- **Doc tests** for every public function with a non-trivial example.
- **Mocking:** Prefer the real implementation with a test double (in-memory store, mock server) over generated mocks. Use `mockall` only at API boundaries (e.g., HTTP clients). DO NOT use `mockall` for every internal trait.
- **Benchmarks** for performance-critical code in `benches/*.rs` using `criterion`.
- **Snapshot tests** for complex output (e.g., error displays, serialized responses) with `insta`.
- **Miri tests** for any code using `unsafe` — `cargo miri test`.

### 4.4 Coverage Requirements
- **CI MUST enforce ≥ 80% line coverage** on changed files (use `cargo-llvm-cov` or `grcov`).
- Coverage of new code must not decrease the overall project coverage.
- 100% coverage is NOT the goal — meaningful assertions are.

### 4.5 Lint, Format, and Static Analysis
- CI MUST run `cargo fmt --all -- --check` — fail if any diff.
- CI MUST run `cargo clippy --workspace --all-targets --all-features -- -D warnings` (deny all warnings).
- CI MUST run `cargo check --workspace --all-targets --all-features`.
- CI MUST run `cargo test --workspace --all-features` with `RUSTFLAGS="-D warnings"` to fail builds on warnings.
- CI MUST run `cargo miri test` for any workspace member using `unsafe`.
- CI MUST run `cargo deny check` (license, advisory, bans, sources).

---

## 5. GitHub Actions CI/CD Pipeline (Required)

You MUST generate a `.github/workflows/` directory with the following workflow files:

### 5.1 `ci.yml` — Continuous Integration
Triggered on `push` to any branch and on `pull_request` to `main`. Steps:
1. Checkout code (`actions/checkout@v4`).
2. Install Rust toolchain via `dtolnay/rust-toolchain@stable` with pinned version from `rust-toolchain.toml`.
3. Cache Cargo registry and target via `Swatinem/rust-cache@v2`.
4. Install additional components: `clippy`, `rustfmt`, `llvm-tools-preview`, `rust-src` (for Miri).
5. Verify formatting: `cargo fmt --all -- --check`.
6. Lint: `cargo clippy --workspace --all-targets --all-features -- -D warnings`.
7. Static checks: `cargo check --workspace --all-targets --all-features`.
8. Build: `cargo build --workspace --all-features --locked`.
9. Test with coverage: `cargo llvm-cov --workspace --all-features --lcov --output-path lcov.info`.
10. Miri (only for crates with `unsafe`): `cargo miri test --workspace`.
11. Security: `cargo deny check`.
12. Build matrix: stable, beta, and MSRV (Minimum Supported Rust Version) for the project.
13. OS matrix: `ubuntu-latest`, `macos-latest` (Windows optional for non-Windows-specific code).

### 5.2 `build-and-push.yml` — Container Image Build & Push
Triggered on `push` to `main` and on version tags (e.g., `v*.*.*`). Steps:
1. Checkout code.
2. Set up Docker Buildx (`docker/setup-buildx-action@v3`).
3. Install Rust toolchain (pinned) and cache.
4. **Cross-compile inside the builder stage** of the Dockerfile (recommended) OR build natively with the buildx cache.
5. **Reproducible builds:** use `SOURCE_DATE_EPOCH`, `--build-arg`, and strip binaries (`-C strip=symbols` in `RUSTFLAGS`).
6. **Multi-platform build:** `docker buildx build --platform linux/amd64,linux/arm64 ...`.
7. **Reproducible labels:** include `org.opencontainers.image.source`, `org.opencontainers.image.revision`, `org.opencontainers.image.created`, `org.opencontainers.image.version`.
8. **Cache:** Use `docker/build-push-action@v5` with `cache-from: type=gha` and `cache-to: type=gha,mode=max`.
9. **Provenance/SBOM:** `provenance: true` and SBOM generation enabled.
10. Push with semantic tags: `<sha>`, `<branch>-<sha>`, `latest` (only for `main`), and `vX.Y.Z` for tags.

### 5.3 `release.yml` — Release Automation
Triggered on `v*.*.*` tags. Steps:
1. Build binaries for multiple OS/arch (linux/darwin/windows × amd64/arm64) via `cargo build --release --target <triple>` or `cross`.
2. Generate SHA256 checksums.
3. Create GitHub Release with `softprops/action-gh-release@v2`, attaching binaries and checksums.
4. Auto-generate changelog via `git-cliff` (recommended) or `release-drafter`.
5. **Publish to crates.io** (if the crate is a library): `cargo publish --dry-run` first, then `cargo publish`. Tag the commit.

### 5.4 `codeql.yml` (optional but recommended)
GitHub CodeQL static analysis for security scanning on every push and weekly schedule.

### 5.5 `audit.yml` (recommended)
Scheduled daily run of `cargo audit` (or `cargo deny check advisories`) to detect new vulnerabilities in dependencies.

---

## 6. Dockerfile Standards (Multi-Stage, Distroless)

You MUST generate a production-grade, multi-stage `Dockerfile` at the project root:

```dockerfile
# syntax=docker/dockerfile:1.7

# ---------- Stage 1: Chef (cargo-chef cache) ----------
FROM rust:1.83-slim-bookworm AS chef
RUN cargo install cargo-chef --locked
WORKDIR /app

# ---------- Stage 2: Planner (recipe of dependencies) ----------
FROM chef AS planner
COPY . .
RUN cargo chef prepare --recipe-path recipe.json

# ---------- Stage 3: Builder ----------
FROM chef AS builder
COPY --from=planner /app/recipe.json recipe.json
# Cache dependencies separately — layer reuse is the secret sauce
RUN cargo chef cook --release --recipe-path recipe.json
COPY . .
# Reproducible build flags
ENV SOURCE_DATE_EPOCH=1 \
    CARGO_INCREMENTAL=0 \
    RUSTFLAGS="-C strip=symbols -C link-arg=-Wl,-build-id=none"
RUN cargo build --release --locked --bin <appname>

# ---------- Stage 4: Runtime (Distroless) ----------
FROM gcr.io/distroless/cc-debian12:nonroot
COPY --from=builder /app/target/release/<appname> /usr/local/bin/app
USER nonroot:nonroot
EXPOSE 8080
ENTRYPOINT ["/usr/local/bin/app"]
```

**STRICT RULES:**
- **Multi-stage build is mandatory.** No Rust toolchain, source code, or intermediate artifacts in the final image.
- **Use `cargo-chef`** for deterministic, cacheable dependency layers. This gives 10–50× faster rebuilds.
- **Distroless final image is mandatory** (`gcr.io/distroless/cc-debian12` if glibc/musl is required, `static` for pure Rust with `musl`).
- **Cross-compile to `musl`** for the smallest static binaries:
  `RUN rustup target add x86_64-unknown-linux-musl && cargo build --release --target x86_64-unknown-linux-musl`
- **Run as non-root user** (`USER nonroot:nonroot`).
- **Pin base image by digest** (`@sha256:...`) in production.
- **Use `.dockerignore`** to exclude `target/`, `.git/`, `*.md`, `tests/` (or keep tests for the build stage only).
- **Pass secrets via BuildKit** (`RUN --mount=type=secret,id=...`) — never `ARG` for secrets.
- **Multi-platform builds** (linux/amd64, linux/arm64) MUST be supported.
- **HEALTHCHECK** is required for services.
- Set `SOURCE_DATE_EPOCH` for reproducible builds.

You MUST also generate a `.dockerignore` file at the project root excluding at minimum: `target/`, `.git/`, `.github/`, `*.md`, `coverage/`, `Cargo.lock.bak`.

---

## 7. Standardized Commands (Makefile or Makefile.toml)

You MUST generate a `Makefile` (or `Makefile.toml` if using `cargo-make`) at the project root with the following targets:

```makefile
.PHONY: help tidy build build-release test test-coverage test-property lint format check audit deny run docker-build docker-run clean

BINARY_NAME := app
CARGO       := cargo
DOCKER      := docker

help: ## Show this help
	@grep -E '^[a-zA-Z_-]+:.*?## .*$$' $(MAKEFILE_LIST) | awk 'BEGIN {FS = ":.*?## "}; {printf "\033[36m%-20s\033[0m %s\n", $$1, $$2}'

tidy: ## Run cargo tidy
	$(CARGO) tidy

build: ## Debug build
	$(CARGO) build --workspace --all-features

build-release: ## Release build (optimized, stripped)
	RUSTFLAGS="-C strip=symbols" $(CARGO) build --release --workspace --locked

test: ## Run unit and integration tests
	$(CARGO) test --workspace --all-features --locked

test-coverage: ## Run tests with coverage report
	$(CARGO) llvm-cov --workspace --all-features --html --output-dir coverage

test-property: ## Run property-based tests
	$(CARGO) test --workspace --features proptest

lint: ## Run clippy
	$(CARGO) clippy --workspace --all-targets --all-features -- -D warnings

format: ## Run rustfmt
	$(CARGO) fmt --all

format-check: ## Check formatting
	$(CARGO) fmt --all -- --check

check: ## Run cargo check
	$(CARGO) check --workspace --all-targets --all-features --locked

audit: ## Security audit
	$(CARGO) install --locked cargo-audit || true
	$(CARGO) audit

deny: ## License & advisory check
	$(CARGO) install --locked cargo-deny || true
	$(CARGO) deny check

run: ## Run the application
	$(CARGO) run --bin $(BINARY_NAME)

docker-build: ## Build the Docker image
	$(DOCKER) buildx build --platform linux/amd64,linux/arm64 -t $(BINARY_NAME):local .

docker-run: ## Run the Docker image locally
	$(DOCKER) run --rm -p 8080:8080 $(BINARY_NAME):local

clean: ## Clean build artifacts
	$(CARGO) clean
	rm -rf coverage/
```

---

## 8. Configuration & Secrets

- Use the **`config`** crate (`config-rs`) or **`figment`** for layered configuration (defaults, file, env vars, CLI flags).
- All configuration MUST be loaded from environment variables (12-factor app).
- Provide a typed config struct, loaded at startup, validated with `config::Config::try_from`.
- **NEVER** hardcode secrets, connection strings, or credentials.
- All secrets MUST be loaded from env vars or a secret manager (Vault, AWS Secrets Manager, GCP Secret Manager). For local dev, use a `.env` file via `dotenvy` and provide a `.env.example` (NOT `.env`) documenting every required variable.
- Validate configuration at startup and FAIL FAST with a clear error message if required values are missing or malformed. Never start the app in a half-configured state.
- Use `std::time::Duration` for any timeout-related config — never raw integers.

---

## 9. Observability Requirements

- Use the **`tracing`** crate for structured, span-based logging.
- Use **`metrics`** + **`metrics-exporter-prometheus`** for Prometheus metrics with a `/metrics` endpoint.
- Use **`opentelemetry` / `tracing-opentelemetry`** for distributed tracing (OTLP exporter).
- Initialize telemetry ONCE in `main.rs` via a `telemetry::init()` function that returns a guard for graceful flush on shutdown.
- Distinguish **liveness** (`/healthz`), **readiness** (`/readyz`), and a startup probe if needed. Readiness MUST check downstream dependencies (DB, cache, message queue).
- Log a `trace_id` and `span_id` on every log line via `tracing_subscriber` + `tracing_opentelemetry` layer.
- Emit RED metrics (Rate, Errors, Duration) for every HTTP route and gRPC method via `axum-prometheus` or a custom middleware.
- Emit database query metrics via `sqlx::query!` macros' `#[instrument]` or a custom wrapper.

---

## 10. Graceful Shutdown & Lifecycle

- Trap `SIGINT` and `SIGTERM` via `tokio::signal::ctrl_c()` and `tokio::signal::unix::signal(SignalKind::terminate())`.
- Use a `tokio_util::sync::CancellationToken` (or `tokio::sync::watch` channel) to broadcast shutdown to all spawned tasks.
- On signal:
  1. Stop accepting new requests (server enters graceful shutdown).
  2. Wait for in-flight requests to complete (with a hard deadline, e.g., 30 seconds).
  3. Flush logs and metrics.
  4. Close DB pools (`sqlx::Pool::close()`).
  5. Exit with code 0.
- If the shutdown deadline is exceeded, log a warning and exit with a non-zero code.
- The HTTP server MUST be configured with sensible timeouts: `axum::serve(...).with_graceful_shutdown(shutdown_signal())`.

---

## 11. API Design

- Use **RESTful resource naming** for HTTP APIs (plural nouns: `/users`, `/users/{id}`).
- Return appropriate HTTP status codes: 200, 201, 204, 400, 401, 403, 404, 409, 422, 429, 500.
- Return errors as JSON with a consistent `Problem` shape (RFC 7807 inspired) — use the **`axum-problem`** or **`error-stack`** crate.
- Version APIs explicitly: `/v1/users`, `/v2/users`.
- Validate every request body and query parameter via the **`validator`** crate (derive `Validate`) or **`garde`**. Reject early with 400.
- Document every public API with OpenAPI 3.1 spec, generated via **`utoipa`** (axum), `aide` (axum), or `paperclip` (actix).
- For gRPC, keep `.proto` files in `proto/` and generate Rust code via **`tonic-build`** or **`buf`** + `tonic`.

---

## 12. Database Access Discipline

- Use **`sqlx`** for compile-time-checked SQL queries (preferred), or **`diesel`** for type-safe DSL.
- Always use a connection pool (`sqlx::PgPool`, `deadpool`) — never open a connection per request.
- Always use `&mut sqlx::Transaction` for multi-statement transactions. Never leak transactions.
- Set query timeouts via `tokio::time::timeout` or `sqlx::query::fetch` with cancellation.
- Use prepared statements (sqlx does this automatically).
- Migrations: use **`sqlx-cli`** (`sqlx migrate add` / `sqlx migrate run`) or **`refinery`**. Checked into the repo, applied automatically in CI/CD.
- Add DB indexes for any column used in `WHERE`, `ORDER BY`, or `JOIN ON` clauses.

---

## 13. Dependency Hygiene

- Run `cargo update --dry-run` in CI to detect supply-chain drift.
- Run `cargo audit` and `cargo deny check advisories` in CI.
- Pin GitHub Actions to a full version SHA in production, not just a major tag.
- Renovate or Dependabot MUST be enabled to keep dependencies and actions up to date automatically.
- Reject any crate that has not had a release in the last 24 months OR is unmaintained. Use `cargo-deny` to enforce bans.
- Prefer crates with `#![forbid(unsafe_code)]` when possible for added safety.

---

## 14. Commit & Branching Standards

- Branch naming: `<type>/<ticket-id>-<short-description>` (e.g., `feat/PROJ-123-add-user-registration`).
- Commit messages follow **Conventional Commits**:
  - `feat:`, `fix:`, `chore:`, `docs:`, `style:`, `refactor:`, `perf:`, `test:`, `build:`, `ci:`
  - Example: `feat(api): add rate limiting to /v1/users endpoint`
- Squash-merge feature branches to `main` with a single conventional commit.
- Every PR MUST reference a ticket/issue ID.
- `main` branch is protected: requires PR, 1+ review, all CI checks green, no direct pushes.
- Every PR MUST pass `cargo fmt`, `cargo clippy -D warnings`, `cargo test`, and `cargo audit` before merge.

---

## 15. Pre-Commit Checklist (Self-Audit)

Before declaring work complete, the agent MUST verify:

- [ ] `cargo fmt --all -- --check` produces no diff
- [ ] `cargo clippy --workspace --all-targets --all-features -- -D warnings` passes
- [ ] `cargo check --workspace --all-targets --all-features` passes
- [ ] `cargo test --workspace --all-features` passes locally
- [ ] `cargo build --release --locked` produces an optimized binary
- [ ] `cargo llvm-cov` shows ≥ 80% coverage on changed files
- [ ] `cargo audit` shows no unfixed vulnerabilities
- [ ] `cargo deny check` passes (licenses, advisories, bans)
- [ ] No `unwrap()`, `expect()`, `panic!()`, `unreachable!()`, or `todo!()` in production code paths
- [ ] No new `unsafe` blocks; if any were added, they have `// SAFETY:` comments and `cargo miri test` passes
- [ ] No `.clone()` calls added without justification (prefer borrowing)
- [ ] New code has corresponding unit tests (table-driven or property-based where applicable)
- [ ] Doc tests pass (`cargo test --doc`)
- [ ] Integration tests use the standard `tests/` directory
- [ ] No new dependencies added without justification in the PR description
- [ ] `Dockerfile` builds successfully: `docker buildx build --platform linux/amd64,linux/arm64 .`
- [ ] The final Docker image runs as non-root and is based on distroless or scratch
- [ ] `/healthz` and `/readyz` endpoints respond correctly
- [ ] No secrets, tokens, or credentials in the diff
- [ ] `rust-toolchain.toml` is committed
- [ ] `clippy.toml`, `rustfmt.toml`, and `deny.toml` are committed
- [ ] `Makefile` targets (`make help`, `make test`, `make build`, `make lint`, `make audit`, `make deny`) all succeed
- [ ] CI workflow files validate locally with `actionlint` or `zizmor`
- [ ] OpenAPI/Proto spec is updated for any API changes
- [ ] `CHANGELOG.md` (or release notes via `git-cliff`) is updated for user-facing changes
- [ ] README is updated if commands, env vars, or setup steps changed

---

## Output Format for the Agent

When generating a new Rust service, the agent MUST output, at minimum:

1. The full directory tree of the generated workspace.
2. The complete contents of every file created (with `path/to/file.rs` headers).
3. A `README.md` containing: project description, prerequisites, `make` quickstart, env-var reference, and a "How to deploy" section.
4. A summary of decisions made and the trade-offs (e.g., why sqlx over diesel, why axum over actix-web, why distroless over alpine).

When reviewing or modifying existing Rust code, the agent MUST output:

1. A unified diff of every changed file.
2. A categorized list of issues: `BLOCKING`, `WARNING`, `OPTIMIZATION`, `NIT`.
3. For each issue: file path, line number, the rule violated (with link to this document or a referenced best-practice doc), and a suggested fix in code.
4. A final verdict: `[APPROVED]`, `[APPROVED WITH WARNINGS]`, or `[REJECTED — CHANGES REQUIRED]`.
