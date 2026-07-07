---
title: Rust - Error Handling, Exceptions, and Logging
status: draft
version: 0.0.1
created: 2026-07-06
updated: 2026-07-07
---

# Rust — Error Handling, Exceptions, and Logging

Rust has **no exceptions**. Recoverable failure is modeled with the value types `Result<T, E>` and `Option<T>`, propagated with the `?` operator. Unrecoverable failure (a violated invariant, a logic bug) is a **panic**, which by default unwinds the stack. This section catalogs the conventions, the canonical crates (`thiserror`, `anyhow`), and the logging facade situation (`log` vs. `tracing`).

## `Result<T, E>` and `Option<T>`

- `Option<T>` — the value might be absent (`Some(v)` or `None`).
- `Result<T, E>` — the operation might fail (`Ok(v)` or `Err(e)`).

Both are exhaustive: the compiler forces the caller to deal with the failure case. There is no silent exception bubbling.

```rust
fn find_user(id: u64) -> Option<User> { /* ... */ None }

fn lookup(id: u64) -> Result<User, LookupError> {
    find_user(id).ok_or(LookupError::NotFound(id))
}
```

Convert `Option` to `Result` with `.ok_or(err)` / `.ok_or_else(|| err)`. Convert `Result` to `Option` with `.ok()` (drops the error) — usually only when you genuinely want to discard it.

## The `?` operator

`?` is the idiomatic propagation operator. On `Ok(v)` it yields `v`; on `Err(e)` it `return`s `Err(From::from(e))`, using the `From` conversion so error types can be widened automatically.

✅ **Do:**

```rust
use std::fs;
use std::io;
use std::num::ParseIntError;
use std::path::Path;

fn read_count(path: &Path) -> Result<u64, Box<dyn std::error::Error>> {
    let text = fs::read_to_string(path)?;          // ? on io::Error
    let n: u64 = text.trim().parse()?;             // ? on ParseIntError
    Ok(n)
}
```

❌ **Don't** — manual `match` propagation that `?` replaces:

```rust
fn read_count(path: &Path) -> Result<u64, Box<dyn std::error::Error>> {
    let text = match fs::read_to_string(path) {
        Ok(t) => t,
        Err(e) => return Err(e.into()),
    };
    let n: u64 = match text.trim().parse() {
        Ok(n) => n,
        Err(e) => return Err(e.into()),
    };
    Ok(n)
}
```

`?` also works on `Option`, returning `None` early — useful in `Option`-returning helpers.

## The `Error` trait

Errors implement `std::error::Error` (which requires `Debug + Display`). **As of Rust 1.81** (Sept 2024), the `Error` trait is also stabilized in `core` as `core::error::Error`, so `#![no_std]` crates can use the same trait. `std::error::Error` is now a re-export of `core::error::Error`. On Rust ≥ 1.81 prefer writing `core::error::Error` (or letting `thiserror` handle it) when you want `no_std` compatibility.

## `thiserror` for libraries, `anyhow` for applications

The community's de facto division of labor (well-supported across the Rust docs, the `thiserror`/`anyhow` READMEs, and countless blog posts):

### `thiserror` — typed, structured errors in **library** crates

`thiserror` is a derive macro that generates `Error`, `Display`, and `From` implementations for an enum of error variants. It lets **callers `match` on the variants** and react differently.

✅ **Do** — in a library:

```rust
use thiserror::Error;

#[derive(Debug, Error)]
pub enum ConfigError {
    #[error("could not read config file {0}")]
    Read(std::io::Error),

    #[error("invalid config: {0}")]
    Parse(toml::de::Error),

    #[error("missing required key {key}")]
    Missing { key: String },
}

// Caller can react differently:
fn handle(e: &ConfigError) {
    match e {
        ConfigError::Read(io_err) => log::error!("io failure: {io_err}"),
        ConfigError::Missing { key } => log::error!("missing {key}"),
        _ => log::error!("other: {e}"),
    }
}
```

### `anyhow` — opaque, context-bearing errors in **application** binaries

`anyhow` provides `anyhow::Error` (a type-erased boxed error) and a `Context` trait for attaching context. Use it where you don't need to programmatically differentiate errors — you just want to propagate and report them.

✅ **Do** — in a binary:

```rust
use anyhow::{Context, Result};

fn run() -> Result<()> {
    let path = std::env::var("CONFIG_PATH")
        .context("CONFIG_PATH must be set")?;
    let text = std::fs::read_to_string(&path)
        .with_context(|| format!("reading {path}"))?;
    let cfg: Config = toml::from_str(&text)
        .context("parsing config")?;
    Ok(())
}
```

### When the rule of thumb bends

- A **`#[derive(Error)]`** enum is still the right call in a binary if some call site needs to `match` on it.
- `anyhow::Error` is sometimes acceptable in an *internal* library that only one application consumes. The line is "is this a **public** API contract?" — if yes, prefer a typed error; if no, `anyhow` is fine.

Recognized alternatives worth knowing: **`snafu`** (similar to `thiserror` with embedded context), **`fehler`/`no-panic`** for specific ergonomics, and the experimental `eyre` (an `anyhow` fork with customizable hooks). The standard documents `thiserror` + `anyhow` as the mainstream pair; `snafu` and `eyre` are listed side by side as recognized alternatives.

## Adding context

The pattern of wrapping a low-level error with "what you were doing" is idiomatic:

```rust
use anyhow::Context;

let bytes = std::fs::read(path)
    .with_context(|| format!("loading config from {path}"))?;
```

With `thiserror`, you typically model the context as an enum variant. With `anyhow`, `.context(...)` / `.with_context(|| ...)` does it at the call site.

## panic vs. Result — when each is appropriate

| Situation | Use |
|---|---|
| Expected, recoverable failure (file missing, parse error, network timeout) | `Result` |
| Absent value that has a sensible default | `Option` (or `unwrap_or`) |
| A **logic bug / violated invariant** (the program is in a state that *cannot* happen if the code is correct) | `panic!` / `assert!` / `debug_assert!` |
| Index out of bounds, integer overflow in debug | `panic` (already the language's behavior) |

✅ **Do** — panic for genuine invariants:

```rust
fn nth(xs: &[i32], n: usize) -> i32 {
    // Precondition: caller guarantees n < xs.len(); otherwise it's a bug.
    xs[n]                       // panics on out-of-bounds — correct for a logic bug
}

fn process(data: &Data) {
    assert!(data.is_valid(), "invariant: data must be validated upstream");
    /* ... */
}
```

❌ **Don't** — panic for expected, recoverable failure:

```rust
fn load_config(path: &str) -> Config {
    let text = std::fs::read_to_string(path).unwrap();   // ❌ file missing is recoverable
    toml::from_str(&text).unwrap()                       // ❌ parse error is recoverable
}
```

The same function done right returns `Result` (see the `anyhow` example above).

### Panic strategy

By default, Rust panics **unwind** the stack (running destructors). You can switch to `panic = "abort"` in `Cargo.toml` to shrink binary size and abort on panic instead:

```toml
[profile.release]
panic = "abort"
```

This trades away safe cleanup for smaller binaries; common in embedded/WASM, less common in long-running servers.

## `unwrap` and `expect` rules

`unwrap()` panics on `Err`/`None`; `expect("msg")` is the same with a message. Both are appropriate in:

- **Tests** — `assert_eq!(parse("1").unwrap(), 1)` is the norm.
- **Prototyping** — when you'll come back and handle errors properly.
- **Provable invariants** — when a failure would indicate a logic bug *and* you would want to panic.

They are **not** appropriate for fallible I/O, parsing, or anything that depends on untrusted input in production.

Prefer `expect("specific message")` to `unwrap()` in non-test code, because the message appears in the panic and the logs. Clippy: `clippy::unwrap_used`, `clippy::expect_used` can forbid them entirely outside tests.

## Logging — `log` vs. `tracing`

Two recognized ecosystems, documented side by side:

### `log` — the simple facade

A lightweight logging facade with five levels (`error!`, `warn!`, `info!`, `debug!`, `trace!`). You call the `log` macros; a separate *backend* (e.g. `env_logger`, `fern`) actually emits the records. This lets libraries log without picking a backend.

```rust
// In a library:
use log::{info, warn};

pub fn process(input: &str) {
    info!(target: "mylib::process", "processing {len} bytes", len = input.len());
    if input.is_empty() { warn!("empty input"); }
}
```

```rust
// In a binary:
fn main() {
    env_logger::init();          // reads RUST_LOG=info,mylib=debug
    /* ... */
}
```

`env_logger` is the de facto simple backend; configure via the `RUST_LOG` environment variable.

### `tracing` — structured, contextual, async-aware

`tracing` (with `tracing-subscriber`) is the more modern, structured approach. It introduces **spans** (scoped, contextual regions) in addition to events, and is the standard in async/Tokio code because it propagates context across `.await` points.

```rust
use tracing::{info, instrument, span, Level};

#[instrument(skip(repo), fields(user_id = %user.id))]
pub fn fetch_user(user: &UserRef, repo: &Repo) -> User {
    let span = span!(Level::DEBUG, "db_lookup", table = "users");
    let _enter = span.enter();
    info!(user_id = %user.id, "fetching");
    /* ... */ unimplemented!()
}
```

```rust
// Binary side, with tracing-subscriber:
fn main() {
    tracing_subscriber::fmt().with_env_filter(
        tracing_subscriber::EnvFilter::from_default_env(),
    ).init();
    /* ... */
}
```

### Which to choose

| Need | Pick |
|---|---|
| Tiny synchronous library, just want levels | `log` + `env_logger` |
| Async/Tokio app, want spans & structured fields, distributed tracing | `tracing` + `tracing-subscriber` |
| Library that wants to be usable by both | `log` (the facade; `tracing` has a `log` compatibility feature) |

Both are recognized; `tracing` is increasingly the default for new application code, but `log` is far from deprecated.

### Log levels — the community meaning

- `error!` — something failed; a user-visible problem or data loss is possible.
- `warn!` — something unusual but recoverable; worth attention.
- `info!` — high-level lifecycle events ("server started", "config loaded").
- `debug!` — diagnostic detail useful in development.
- `trace!` — very fine-grained, typically off in production.

### Structured logging

Both `log` (with `kv` feature / backends like `slog`) and `tracing` support key-value fields, which downstream collectors can index. Prefer structured fields over interpolated strings when the log will be machine-parsed:

✅ `info!(user_id = %user.id, "fetched user");`
❌ `info!("fetched user {}", user.id);`

## Secrets in logs

Never log secrets, tokens, credentials, or PII at any level. Section 9 covers the `secrecy` and `zeroize` crates; the headline rule for this section: treat log statements as if they will be shipped to an aggregator, because they usually are.

## Input Validation

Validating input at the boundaries (e.g., HTTP handlers, CLI arguments) prevents invalid data from propagating deeper into the system.

- **Newtypes for Invariants:** Use the newtype pattern to enforce validity at compile-time. If a function takes a `ValidEmail`, the caller *must* have validated it.
- **The `validator` crate:** A common choice for declarative validation using derive macros on structs.
- **Early Returns:** Validation failures should be returned immediately as a `Result::Err`, typically mapping to a `400 Bad Request` in web APIs rather than a `500 Internal Server Error`.

```rust
use validator::Validate;

#[derive(Validate)]
pub struct SignupRequest {
    #[validate(email)]
    pub email: String,
    #[validate(length(min = 8))]
    pub password: String,
}

pub fn signup(req: SignupRequest) -> Result<(), AppError> {
    req.validate().map_err(AppError::InvalidInput)?;
    // ...
    Ok(())
}
```

## Resilience and Retries

Fail-fast for permanent errors (e.g., validation failures, unauthorized access), but employ retry mechanisms for transient errors (e.g., network timeouts, temporary database unavailability).

- **Retry Crates:** Use crates like `tokio-retry` or `backoff` to implement exponential backoff and jitter.
- **Idempotency:** Ensure the operation being retried is idempotent to prevent side effects on duplicate executions.
- **Timeouts:** Wrap network calls with `tokio::time::timeout` to prevent hanging requests.

```rust
use backoff::{future::retry, ExponentialBackoff};

async fn fetch_data() -> Result<Data, reqwest::Error> {
    retry(ExponentialBackoff::default(), || async {
        let res = reqwest::get("https://api.example.com").await?;
        res.error_for_status()?.json().await.map_err(backoff::Error::transient)
    })
    .await
}
```

## Distributed Tracing and Correlation IDs

In microservices or distributed systems, passing a Correlation ID ensures that logs from a single request can be traced across multiple services.

- **OpenTelemetry integration:** Use `tracing-opentelemetry` to export spans to systems like Jaeger, Zipkin, or Datadog.
- **Correlation IDs:** Extract the `X-Request-Id` or W3C `traceparent` from incoming requests and inject it into the `tracing` Span.
- **Context Propagation:** Use the `opentelemetry` crate's context propagation to inject trace context into outgoing HTTP requests (e.g., via `reqwest` middleware).

```rust
use tracing::{info_span, Instrument};

async fn handle_request(request_id: &str) {
    let span = info_span!("http_request", request_id = %request_id);
    async move {
        tracing::info!("Processing request");
        // ...
    }
    .instrument(span)
    .await;
}
```

## Metrics Conventions

Beyond logging, track application behavior with metrics (counters, gauges, histograms) to build dashboards and alerts.

- **The `metrics` facade:** Similar to the `log` crate, the `metrics` crate provides a facade (macros like `counter!`, `gauge!`, `histogram!`), allowing the backend (e.g., Prometheus) to be configured in the binary.
- **Prometheus:** The `metrics-exporter-prometheus` crate is commonly used to expose a `/metrics` endpoint.
- **Conventions:**
  - *Counters:* Total requests, error counts.
  - *Histograms:* Request latencies, payload sizes.
  - *Gauges:* Current active connections, memory usage.

```rust
use metrics::{counter, histogram};
use std::time::Instant;

pub async fn process_job() {
    let start = Instant::now();
    counter!("jobs_processed_total").increment(1);
    
    // ... do work ...
    
    histogram!("job_processing_duration_seconds").record(start.elapsed().as_secs_f64());
}
```

## Health Checks

Long-running services (e.g., web servers) must expose endpoints for orchestration tools (like Kubernetes) to monitor their state.

- **Liveness Probes (`/health/live`):** Returns `200 OK` simply if the process is running and hasn't deadlocked.
- **Readiness Probes (`/health/ready`):** Returns `200 OK` only if the service is fully ready to handle traffic (e.g., database connection pool is established, caches are warmed).

Using a framework like `axum` or `actix-web`, these are typically simple `GET` routes returning a JSON status or plain text `OK`.

## Summary checklist

- [ ] Fallible code returns `Result`/`Option`; propagate with `?`.
- [ ] Library error types use `thiserror` (typed, matchable); applications use `anyhow` with `.context()`.
- [ ] `panic` reserved for genuine invariants; never for fallible I/O.
- [ ] `unwrap`/`expect` only in tests, prototypes, or with a proven invariant.
- [ ] A deliberate logging choice (`log` or `tracing`), with a subscriber/backend configured in the binary.
- [ ] Structured fields over interpolated strings; no secrets in logs.
- [ ] Input data is validated at the boundary using types or crates like `validator`.
- [ ] Resilience mechanisms (exponential backoff, timeouts) are applied for transient failures.
- [ ] Metrics (counters, histograms) and tracing (OpenTelemetry, correlation IDs) are used for observability.
- [ ] Services expose readiness and liveness health checks.

## References

- The Rust Book — Error handling — `doc.rust-lang.org/book/ch09-00-error-handling.html`.
- The Rust Blog — Rust 1.81.0 (`core::error::Error`) — `blog.rust-lang.org/2024/09/05/Rust-1.81.0/`.
- `thiserror` — `github.com/dtolnay/thiserror`.
- `anyhow` — `github.com/dtolnay/anyhow`.
- `tracing` — `docs.rs/tracing`.
- `env_logger` — `docs.rs/env_logger`.
- `metrics` — `docs.rs/metrics`.
- `validator` — `docs.rs/validator`.
- `backoff` — `docs.rs/backoff`.
- Rust API Guidelines — Documentation of failure (C-FAILURE) — `rust-lang.github.io/api-guidelines/documentation.html`.
