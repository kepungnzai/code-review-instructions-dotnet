# Agent Skills & Knowledge Base: Rust Systems & Backend Engineering

You possess expert-level proficiency in the following domains. Use this knowledge to make authoritative recommendations and write idiomatic, production-grade Rust.

---

## 1. Rust Language Mastery

### 1.1 Ownership, Borrowing & Lifetimes
- The three borrowing rules: (1) at any time, you may have ONE `&mut T` OR any number of `&T`; (2) references must always be valid; (3) the compiler enforces both.
- **Lifetime elision rules** (Rust 2018+): the compiler elides lifetimes in functions with a single input reference, methods with `&self`/`&mut self`, and trait objects.
- **'static** is the longest possible lifetime. Avoid `T: 'static` bounds unless you genuinely own the data for the process lifetime.
- **Higher-Rank Trait Bounds (HRTBs):** `F: for<'a> Fn(&'a str) -> &'a str` — used for closures that work with any lifetime.
- **GATs (Generic Associated Types):** for lending iterators and zero-copy parsers.
- **NLL (Non-Lexical Lifetimes):** since Rust 2018, borrows end at last use, not at end of scope.
- **Subtyping & variance:** `&'long T` is a subtype of `&'short T` (covariance). `&mut T` is invariant. Understand this when designing APIs that hold mutable references.

### 1.2 Smart Pointers
- `Box<T>` — heap allocation, single owner, `Sized`.
- `Rc<T>` — single-threaded reference counting (`!Send`).
- `Arc<T>` — thread-safe reference counting.
- `Weak<T>` — non-owning reference (prevents cycles). Use for parent/child structures.
- `Cell<T>` — interior mutability for `Copy` types (no borrowing).
- `RefCell<T>` — interior mutability with runtime borrow checking (`!Sync`).
- `Mutex<T>` / `RwLock<T>` — interior mutability with blocking synchronization. Use `parking_lot::Mutex` for speed.
- `OnceCell<T>` / `OnceLock<T>` — one-time initialization.
- `Cow<'a, T>` — clone-on-write; use for "maybe-owned" parameters.
- `Pin<P>` — pinning for self-referential types. Required for `tokio` futures and `async-trait` (pre-1.75).

### 1.3 Traits
- **Standard library traits to know cold:**
  - `Clone`, `Copy` — duplication semantics.
  - `Default` — provide a sensible default.
  - `Debug` — derive on EVERYTHING.
  - `Display` — human-readable format.
  - `From` / `TryFrom` — conversions (preferred over `Into`).
  - `AsRef` / `AsMut` — cheap reference conversion.
  - `Deref` / `DerefMut` — smart pointer dereferencing.
  - `Iterator` / `IntoIterator` — iteration.
  - `Send` / `Sync` — thread safety markers.
  - `Hash` / `Eq` / `PartialEq` / `Ord` / `PartialOrd` — comparison and hashing.
  - `Drop` — destructor (rare; prefer RAII and `Drop` impls).
  - `Error` — error trait (via `thiserror`).
  - `Future` — async computation.
- **Sealed traits** with private `Sealed` marker trait to prevent external implementation.
- **Marker traits** for compile-time validation: `IsValidated`, `IsAuthenticated`, etc.
- **Blanket impls** on owned types: `impl<T: Display> MyTrait for T {}` — think twice about coherence.

### 1.4 Error Handling
- `thiserror` for library error enums (derive `Error`).
- `anyhow` for binary error wrapping (add context with `.context()` and `.with_context()`).
- `eyre` — alternative to `anyhow` with better reporting.
- `error-stack` — structured error chains with `Report<T>`.
- `snafu` — location-aware error context generation.
- **Never** use `Box<dyn std::error::Error + Send + Sync>` in libraries; always use a typed error enum.

### 1.5 Concurrency Primitives
- **Threads:** `std::thread::spawn` for OS threads, `rayon::ThreadPool` for data-parallel, `tokio::runtime` for async.
- **Sync:** `std::sync::Mutex`, `std::sync::RwLock`, `parking_lot::Mutex` (faster, no poisoning).
- **Async:** `tokio::sync::Mutex`, `tokio::sync::RwLock`, `tokio::sync::mpsc`, `tokio::sync::broadcast`, `tokio::sync::watch`, `tokio::sync::Notify`, `tokio::sync::Semaphore`.
- **Atomics:** `std::sync::atomic::AtomicBool`, `AtomicU64`, `AtomicPtr<T>`, `AtomicUsize`. Always specify ordering: `Ordering::Relaxed`, `Acquire`, `Release`, `AcqRel`, `SeqCst`.
- **Channels (sync):** `std::sync::mpsc`, `crossbeam::channel` (MPMC), `crossbeam::queue::ArrayQueue` (lock-free).
- **Barrier:** `std::sync::Barrier` for synchronization points.

### 1.6 Async Rust
- `async` / `.await` — zero-cost abstraction, but a `Future` is lazy and must be awaited or polled.
- **Runtime:** `tokio` (multi-threaded by default) is the de facto standard. Alternatives: `async-std`, `smol`. NEVER mix runtimes in the same call tree.
- **`Send` bounds:** spawned tasks must be `Send + 'static`. Watch out for non-Send futures (e.g., `Rc<T>`, `RefCell<T>`).
- **Cancellation:** cooperative. Futures get cancelled when dropped. Use `CancellationToken` to broadcast shutdown.
- **`select!`:** `tokio::select!` for racing futures, `futures::select!` for cross-runtime.
- **Streams:** `tokio_stream` + `futures::Stream` for async iteration. Use `try_stream` macro for fallible streams.
- **Pinning:** `tokio::pin!`, `Box::pin`, `Pin::new` (requires `Unpin`).
- **Backpressure:** design every channel to be bounded. Unbounded channels leak memory under load.

### 1.7 Macros
- **Declarative macros** (`macro_rules!`): for simple repetition and DSLs.
- **Procedural macros** (proc-macros): `#[derive]`, `#[attribute]`, function-like macros. Use `syn`, `quote`, `proc-macro2`.
- **Common derive crates:** `serde`, `thiserror`, `anyhow`, `derive_more`, `smart-default`, `getset`, `strum`, `validator`, `garde`, `utoipa`, `ts-rs`.
- **Function-like macros:** `tokio::main!`, `tokio::test!`, `tracing::instrument`, `sqlx::query!`, `axum::debug_handler!`, `ctor!`.

### 1.8 Memory & Performance
- **Stack vs heap:** stack-allocate small, short-lived values. `Box` for large or unbounded.
- **Vec vs LinkedList:** `Vec` is almost always better (cache locality).
- **HashMap vs BTreeMap:** `HashMap` is faster for lookups; `BTreeMap` keeps keys sorted and is used for range queries.
- **String interning:** `string_cache`, `lasso`, or `compact_str` for memory savings.
- **Arena allocation:** `bumpalo` for short-lived ASTs and request-scoped data.
- **SIMD:** use `std::simd` (nightly) or `packed_simd` (stable alternatives: `wide`, `simdeez`).
- **Allocator:** `jemalloc` or `mimalloc` for high-throughput servers (via `tikv-jemallocator` or `mimalloc-rs`).
- **Profiling:** `cargo flamegraph`, `cargo profiler`, `perf`, `heaptrack`, `dhat` for heap profiling, `valgrind --tool=callgrind` for call graphs.

---

## 2. Web Frameworks & Routing

You are authorized to recommend and use the following proven frameworks:
- **`axum`** — preferred. Built on `tower` and `hyper`. Composable middleware, ergonomic extractors, type-safe routing. Tokio-native.
- **`actix-web`** — high-performance, mature, multi-threaded. Slightly more opinionated.
- **`warp`** — composable, filter-based. Declining in popularity.
- **`rocket`** — easy to learn, async since 0.5. Slower than axum/actix.
- **`poem`** — fast, full-featured. Strong Chinese-community adoption.
- **`tonic`** — gRPC for Rust. Built on `hyper` + `prost`.

**Router & middleware ecosystem:**
- `tower` — service abstraction.
- `tower-http` — HTTP middleware: `TraceLayer`, `TimeoutLayer`, `CorsLayer`, `CompressionLayer`, `RequestIdLayer`, `NormalizePathLayer`, `SetResponseHeaderLayer`.
- `axum-extra` — extractors and middleware.
- `axum-prometheus` — metrics middleware.
- `axum-problem` / `axum-errors` — RFC 7807 problem responses.

**Anti-patterns to reject:**
- Hand-rolled HTTP servers with raw `hyper` (unless you have a measured reason).
- Hand-rolled JSON parsers (use `serde_json`).
- Hand-rolled connection pools (use `deadpool`, `bb8`, `sqlx::Pool`).
- Using `actix-web`'s older `actix-rt` runtime — it's tied to the framework and limits interop with `tokio`.

---

## 3. Databases & Persistence

### 3.1 SQL Databases
- **`sqlx`** — preferred. Compile-time-checked SQL queries, async, native to `tokio`. Supports PostgreSQL, MySQL, SQLite, MSSQL. Migrations via `sqlx-cli`.
- **`diesel`** — type-safe DSL, sync-first (with `diesel-async` for async). Faster compile times than sqlx in some cases.
- **`sea-orm`** — async ORM on top of `sqlx`. Useful for entity-heavy apps.
- **`tokio-postgres`** — low-level async PostgreSQL client.
- **Connection pools:** `sqlx::Pool` (preferred for sqlx), `deadpool-postgres`, `bb8-postgres`, `r2d2` (sync).

**Connection pool rules:**
- Set `max_connections`, `min_connections`, `acquire_timeout`, `idle_timeout` based on workload.
- Use prepared statements (sqlx does this automatically).
- Always set query timeouts via `tokio::time::timeout`.
- Use `tokio::task::spawn_blocking` for synchronous DB calls from async code.

### 3.2 NoSQL & Caching
- **Redis:** `redis-rs` (sync), `deadpool-redis` (async pool), `bb8-redis`, `fred` (async, high-perf).
- **MongoDB:** `mongodb` crate (async, official driver).
- **RocksDB / Sled:** `rocksdb` (bindings), `sled` (pure-Rust, embedded).
- **In-memory:** `moka` (TTL + size-bounded LRU/LFU cache), `dashmap` (concurrent HashMap).

### 3.3 Migrations
- **`sqlx-cli`** — `sqlx migrate add`, `sqlx migrate run`, `sqlx migrate revert`. SQL or Rust migrations.
- **`refinery`** — SQL or Rust migrations, framework-agnostic.
- **`sea-orm-cli`** — for sea-orm entities + migrations.
- **`barrel`** — type-safe SQL schema builder.

---

## 4. Observability

### 4.1 Logging & Tracing
- **`tracing`** — the de facto standard. Spans, structured events, async-aware.
- **`tracing-subscriber`** — subscribers, formatters, layers, filters (`EnvFilter` for `RUST_LOG`).
- **`tracing-opentelemetry`** — bridge `tracing` spans to OTel spans.
- **`tracing-bunyan-formatter`** — JSON output in bunyan format.
- **`tracing-log`** — bridge `log` to `tracing`.
- **`slog`** — older alternative; prefer `tracing`.

**Best practices:**
- Every request handler has `#[tracing::instrument(skip(self))]`.
- Use fields, not format strings: `info!(user_id = %id, "user fetched")`.
- Use `tracing::field::display` and `tracing::field::debug` for explicit formatting.
- Set `RUST_LOG=info,my_crate=debug,tower_http=info` in production.
- Use `tracing-actix-web` or `tower_http::trace::TraceLayer` for HTTP request logging.

### 4.2 Metrics
- **`metrics`** — abstract metrics facade.
- **`metrics-exporter-prometheus`** — Prometheus exporter.
- **`axum-prometheus`** — axum-specific middleware for RED metrics.
- **`autometrics`** — drop-in macros + Prometheus exporter + dev-time UI.
- **`prometheus`** — direct Prometheus client (lower-level than `metrics`).

**Naming conventions (Prometheus):**
- Counters: `app_<thing>_total` (e.g., `http_requests_total`).
- Histograms: `app_<thing>_duration_seconds` (e.g., `http_request_duration_seconds`).
- Gauges: `app_<thing>_<unit>` (e.g., `db_pool_size`).

### 4.3 Tracing (OpenTelemetry)
- **`opentelemetry`** / **`opentelemetry-otlp`** — OTLP exporter.
- **`opentelemetry-stdout`** — local debugging.
- **`opentelemetry-prometheus`** — Prometheus exporter.
- **`tracing-opentelemetry`** — bridge from `tracing`.
- **Sampling:** `Sampler::ParentBased(TraceIdRatioBased(0.1))` for 10% sampling.
- **Propagation:** `tracing_opentelemetry::OpenTelemetrySpanExt` for W3C `traceparent` injection/extraction.

### 4.4 Health Checks
- `/healthz` — liveness. Returns 200 if the process is alive.
- `/readyz` — readiness. Checks DB, cache, message queue, downstream services.
- `/metrics` — Prometheus scrape.
- Use a dedicated `health` crate or roll your own with `axum::Router`.

---

## 5. Testing Ecosystem

### 5.1 Test Helpers & Libraries
- **`rstest`** — fixture-based + parameterized tests. The recommended way to write table-driven tests.
- **`proptest`** — property-based testing with shrinking. The de facto standard.
- **`quickcheck`** — alternative property-based testing.
- **`mockall`** — automatic mock generation for traits. Use ONLY at API boundaries.
- **`wiremock`** — mock HTTP servers.
- **`mockito`** — simpler alternative to `wiremock`.
- **`httpmock`** — async HTTP mocking.
- **`testcontainers`** — spin up real DB/cache/queue containers in tests.
- **`insta`** — snapshot testing.
- **`criterion`** — benchmarking.
- **`divan`** — modern alternative to criterion.
- **`pretty_assertions`** — nicer `assert_eq!` output.
- **`tokio-test`** — `tokio::test_now!`, simulated time, etc.
- **`axum-test`** — ergonomically test axum routers.
- **`fake`** — fake data generation (names, addresses, UUIDs, etc.).

### 5.2 Required Test Discipline
- `cargo test --workspace` must always pass.
- `cargo test --doc` must always pass.
- `cargo test --workspace -- --include-ignored` to catch ignored tests during local development.
- Use `serial_test` for tests that must NOT run in parallel (e.g., port-binding).
- Fuzz tests with `cargo-fuzz` for any code taking untrusted input.
- Miri tests (`cargo +nightly miri test`) for any code using `unsafe`.

---

## 6. Build, Lint, and Static Analysis

### 6.1 Linters and Formatters (run all in CI)
- **`cargo fmt`** — official Rust formatter. Config in `rustfmt.toml`.
- **`cargo clippy`** — official linter with 700+ lints. Use `clippy.toml` and `clippy::pedantic`, `clippy::nursery`, or `clippy::restriction` groups as appropriate. Run with `-- -D warnings` to deny.
- **`cargo check`** — fast type-check without code generation.
- **`cargo udeps`** — detect unused dependencies.
- **`cargo machete`** — alternative to udeps, faster.
- **`cargo deny`** — license, advisory, ban, and source restrictions.
- **`cargo audit`** — known vulnerability scanner (RUSTSEC).
- **`cargo geiger`** — count `unsafe` usage in your dependency tree.
- **`cargo miri`** — undefined behavior detector (for `unsafe` code).

### 6.2 Recommended `clippy.toml` and `clippy::pedantic` Lints
Enable at minimum:
- `clippy::unwrap_used`, `clippy::expect_used`, `clippy::panic_used`, `clippy::todo_used` — deny in lib code.
- `clippy::dbg_macro` — deny.
- `clippy::print_stdout`, `clippy::print_stderr` — deny in lib code.
- `clippy::missing_docs_in_private_items` — warn.
- `clippy::module_name_repetitions`, `clippy::needless_pass_by_value`, `clippy::redundant_clone`, `clippy::large_types_passed_by_value` — warn.
- `clippy::cognitive_complexity`, `clippy::too_many_arguments`, `clippy::too_many_lines` — warn.

### 6.3 Build Tooling & Profile
- **Cargo workspaces** for multi-crate projects.
- **Release profile** tuning in `Cargo.toml`:
  ```toml
  [profile.release]
  opt-level = 3
  lto = "fat"
  codegen-units = 1
  strip = "symbols"
  panic = "abort"  # or "unwind" for graceful error reporting
  ```
- **Reproducible builds:** set `SOURCE_DATE_EPOCH`, `RUSTFLAGS="-C strip=symbols -C link-arg=-Wl,-build-id=none"`.
- **Cargo features:** use default features sparingly. Each feature flag is a `cfg` switch; document every feature in `Cargo.toml`.
- **`cargo-hakari`** for `workspace-hack` crates to unify feature flags across the workspace.
- **`cargo-nextest`** — faster test runner with better output and per-test isolation.

---

## 7. Security

- **`rustls`** — modern TLS implementation (preferred over OpenSSL).
- **`ring`** or **`aws-lc-rs`** — cryptography primitives (use `ring` for new projects).
- **`argon2`**, **`bcrypt`**, **`scrypt`** — password hashing.
- **`jsonwebtoken`** — JWT validation. **Always validate signature, issuer, audience, expiry, and not-before.**
- **`secrecy`** — `Secret<T>` wrapper that prevents accidental logging of sensitive data.
- **OWASP Rust Secure Coding:** input validation, output encoding, parameterized queries (no string concatenation in SQL), CSRF protection for state-changing endpoints.
- **`cargo-geiger`** — count `unsafe` blocks in your crate and dependencies.
- **`cargo-audit`** — daily CI scan for RUSTSEC advisories.
- **Secrets:** never log (use `secrecy::Secret<T>`), never commit, use Vault or cloud secret managers.
- **TLS:** always use `rustls` with TLS 1.2 minimum (TLS 1.3 preferred). Configure cipher suites explicitly.
- **Rate limiting:** use `governor` for in-memory token bucket, or a Redis-backed limiter.
- **CORS:** explicit allow-list via `tower-http::cors::CorsLayer`. NEVER `allow_origin(Any)`.
- **Security headers:** HSTS, X-Content-Type-Options, X-Frame-Options via `tower-http::set_header::SetResponseHeaderLayer`.
- **Input validation:** `validator` (derive `Validate`) or `garde`. Validate at the edge.

---

## 8. Resilience Patterns

- **Circuit breaker:** `failsafe-rs` (port of Failsafe) or custom with `tokio::sync::RwLock<State>`.
- **Retry with backoff:** `backoff` or `retry` crate. Always use **exponential backoff with jitter**. Never retry non-idempotent operations without safeguards.
- **Timeouts:** every outbound call MUST have a timeout via `tokio::time::timeout`. Set both connection and request timeouts on HTTP clients.
- **Bulkhead:** isolate failures using `tokio::sync::Semaphore` to limit concurrent operations.
- **Rate limiting:** `governor` (in-memory) or `redis-cell` (distributed).
- **Graceful shutdown:** on SIGINT/SIGTERM, cancel the `CancellationToken`, wait for tasks (with timeout), flush telemetry, close DB pools, exit 0.
- **Health checks:** `/healthz` (liveness) and `/readyz` (readiness with dependency checks).

---

## 9. Recommended GitHub Actions

You are authorized to use and configure the following proven actions:
- `actions/checkout@v4` — repository checkout.
- `dtolnay/rust-toolchain@stable` — install pinned Rust toolchain.
- `Swatinem/rust-cache@v2` — Cargo registry and target cache.
- `taiki-e/install-action` — install `clippy`, `rustfmt`, `cargo-audit`, `cargo-deny`, `cargo-llvm-cov`, `cargo-machete`, `cargo-geiger`, `cargo-miri`.
- `actions/upload-artifact@v4` — build artifacts.
- `docker/setup-qemu-action@v3`, `docker/setup-buildx-action@v3`, `docker/login-action@v3`, `docker/metadata-action@v5`, `docker/build-push-action@v5` — container build & push.
- `aquasecurity/trivy-action@master` — container & filesystem vulnerability scanning.
- `github/codeql-action@v3` — code scanning.
- `softprops/action-gh-release@v2` — GitHub releases.
- `codecov/codecov-action@v4` — coverage reporting.
- `orhun/git-cliff-action@v1` — changelog generation.
- `EmbarkStudios/cargo-deny-action@v1` — deny checks.
- `woodruffw/zizmor-action@v1` — workflow security analysis.
- `reviewdog/action-rust-clippy@v1` — inline clippy annotations on PRs.

---

## 10. Recommended Docker Base Images

- **`rust:1.83-slim-bookworm`** — for builder stages (with cargo-chef pre-installed).
- **`rust:1.83-bookworm`** — for builder stages when glibc dev tools are needed.
- **`rust:1.83-alpine`** — smaller builder, but musl is needed for static linking.
- **`messense/rust-musl-cross:x86_64-unknown-linux-musl`** — for cross-compilation to musl.
- **`gcr.io/distroless/static-debian12:nonroot`** — preferred for statically linked Rust binaries (musl).
- **`gcr.io/distroless/cc-debian12:nonroot`** — for binaries that need glibc or `libssl` (e.g., `rustls` + `ring` + system CA roots).
- **`cgr.dev/chainguard/static`** — hardened minimal image, alternative to distroless.
- **`alpine:3.20`** — acceptable; always pin to a specific version.
- **`scratch`** — only for the most hardened cases; you must supply CA certificates and timezone data yourself.
- **Always pin by digest (`@sha256:...`) in production Dockerfiles.**

---

## 11. DevOps & Cloud-Native Best Practices

- **12-Factor App methodology** — strict separation of config from code.
- **Health probes** — separate liveness, readiness, and startup probes for Kubernetes.
- **Graceful shutdown** — handle SIGINT/SIGTERM via `tokio::signal`, drain in-flight requests, flush telemetry, close DB pools.
- **Pod Security Standards:** run as non-root (UID 65532 for distroless `nonroot`), read-only root filesystem, drop all capabilities.
- **Resource requests and limits** MUST be set in Kubernetes manifests. Set both `requests` and `limits` to enable HPA.
- **Horizontal Pod Autoscaler (HPA)** configured based on CPU and custom Prometheus metrics.
- **PodDisruptionBudget (PDB)** for availability during rollouts (set `minAvailable` or `maxUnavailable`).
- **Observability:** logs to stdout in JSON format (collected by the platform), metrics scraped by Prometheus, traces via OTLP.
- **GitOps:** infrastructure and app config managed via Git (ArgoCD, Flux).
- **Progressive delivery:** canary, blue/green, or feature flags for new releases.
- **Distributed runtime features** (in `tokio`): enable `tokio-console` for production debugging of task scheduling.

---

## 12. Anti-Patterns to Always Reject

1. **`unwrap()`, `expect()`, `panic!()` in production code paths.** They crash the process. Use `?`, `match`, or `let ... else`.
2. **Using `Rc<RefCell<T>>` to escape the borrow checker.** Restructure the API or use `Arc<Mutex<T>>` (async: `tokio::sync::Mutex`).
3. **Excessive `.clone()`** to bypass the borrow checker. Pass references; clone only when ownership is genuinely required.
4. **Holding `std::sync::MutexGuard` across `.await`.** This blocks the entire async runtime thread. Use `tokio::sync::Mutex` and scope the lock tightly.
5. **Mixing async runtimes** (`tokio` and `async-std` in the same call tree).
6. **String concatenation in SQL queries** (SQL injection risk). Use `sqlx::query!` with bind parameters.
7. **Stringly-typed APIs** when an enum or newtype is the natural fit.
8. **Premature `unsafe`** without a measured need and a `// SAFETY:` comment.
9. **Using `Box<dyn Error + Send + Sync>` in library public APIs.** Always use a typed error enum.
10. **Fighting the borrow checker with `to_string()`, `clone()`, or `into()`.** Step back; the design probably needs a `&` reference or a refactor.
11. **Reflexive trait abstraction** ("trait everything" — define traits in the consumer crate, not the producer).
12. **Storing `tokio::sync::MutexGuard` in a struct field.** The guard is not `Send` and will leak the lock semantics.
13. **Using `unsafe` for performance** without first measuring with `cargo flamegraph` or `cargo profiler`.
14. **Using `std::thread::sleep` in async code.** Use `tokio::time::sleep`. `std::thread::sleep` blocks the entire runtime thread.
15. **Blocking the async runtime** with `std::fs`, `std::net`, or CPU-bound work. Use `tokio::fs`, `tokio::net`, or `tokio::task::spawn_blocking`.
16. **Cargo features that don't compose** — use `#[cfg(feature = "...")]` carefully and test every feature combination.
17. **`tokio::spawn` without cancellation awareness** — tasks that run forever and are never joined leak resources.
18. **Building Docker images without multi-stage builds** — final image must not contain the Rust toolchain.
19. **Shipping Docker images that run as root** — use the `nonroot` user.
20. **Using `unwrap()` in tests "because it's just a test".** Use `.expect("...")` with a message that identifies the failure.
