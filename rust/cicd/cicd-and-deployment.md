---
title: Rust - CI/CD and Deployment
status: draft
version: 0.0.1
created: 2026-07-06
updated: 2026-07-06
---

# Rust — CI/CD and Deployment

Covers how Rust code moves from a merged commit to a running system. The Rust community has converged on a clear, mostly standardized CI story: **GitHub Actions is the dominant platform**, with a small set of widely-used actions and a conventional pipeline.

## 1. Dominant CI platform: GitHub Actions

GitHub Actions is the de facto CI platform for Rust crates and binaries. GitLab CI, CircleCI, Jenkins, and Azure Pipelines are all seen in industry, but the open-source ecosystem has standardized on Actions.

### The conventional pipeline

A serious Rust CI workflow runs, in order:

1. `cargo fmt --check` — formatting must be unchanged.
2. `cargo clippy --all-targets --all-features -- -D warnings` — linter with warnings as errors.
3. `cargo test --all-features` — unit + integration tests.
4. `cargo build --release` (and cross builds) — release build verification.
5. `cargo audit` — security advisory scan against RustSec.
6. (optional) coverage via `cargo-tarpaulin` or `cargo-llvm-cov`.

### Canonical actions

| Action | Role |
|---|---|
| [`dtolnay/rust-toolchain`](https://github.com/dtolnay/rust-toolchain) | Installs a specific toolchain + components in one step. The community-standard way to install Rust in CI. |
| [`Swatinem/rust-cache`](https://github.com/Swatinem/rust-cache) | Caches `~/.cargo` and `target/` per-OS-per-lockfile. Essential for acceptable CI times. |
| [`taiki-e/install-action`](https://github.com/taiki-e/install-action) | Installs cargo subcommands (`cargo-nextest`, `cargo-deny`, `cargo-llvm-cov`, etc.). |
| [`rustsec/audit-check`](https://github.com/rustsec/audit-check) | Runs `cargo audit`. |

### Sample workflow

```yaml
# .github/workflows/ci.yml
name: CI

on:
  push:
    branches: [main]
  pull_request:

env:
  CARGO_TERM_COLOR: always
  RUSTFLAGS: -D warnings

jobs:
  lint-test:
    name: fmt + clippy + test (${{ matrix.os }})
    runs-on: ${{ matrix.os }}
    strategy:
      fail-fast: false
      matrix:
        os: [ubuntu-latest, windows-latest, macos-latest]
    steps:
      - uses: actions/checkout@v4
      - uses: dtolnay/rust-toolchain@stable
        with:
          components: rustfmt, clippy
      - uses: Swatinem/rust-cache@v2

      - name: Format check
        run: cargo fmt --all -- --check

      - name: Clippy
        run: cargo clippy --all-targets --all-features -- -D warnings

      - name: Test
        run: cargo test --all-features --workspace

  audit:
    name: security audit
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: rustsec/audit-check@v2.0.0
        with:
          token: ${{ secrets.GITHUB_TOKEN }}

  build-release:
    name: release build
    runs-on: ubuntu-latest
    needs: [lint-test]
    steps:
      - uses: actions/checkout@v4
      - uses: dtolnay/rust-toolchain@stable
      - uses: Swatinem/rust-cache@v2
      - run: cargo build --release --all-features
```

> Use `cargo-nextest` instead of `cargo test` for faster, more parallel test execution on large workspaces — it is the de facto upgrade for serious projects.

## 2. Cross-compilation in CI

Rust cross-compiles well; common targets in CI:

- `x86_64-unknown-linux-gnu`, `aarch64-unknown-linux-gnu`
- `x86_64-pc-windows-msvc`, `x86_64-apple-darwin`, `aarch64-apple-darwin`
- `wasm32-unknown-unknown`, `wasm32-wasi`

For targets that need a C cross-toolchain, [`cross`](https://github.com/cross-rs/cross) wraps the complexity in a Docker image:

```bash
cargo install cross --git https://github.com/cross-rs/cross
cross build --release --target aarch64-unknown-linux-gnu
```

## 3. Deploying binaries — cargo-dist, cargo-binstall, GitHub Releases

The modern Rust release story centers on two complementary tools:

### [`cargo-dist`](https://github.com/axodotdev/cargo-dist) (axodotdev)

Automates the **release engineering**: building binaries for multiple targets, packaging them into archives + shell/powershell installers, generating a machine-readable manifest, and cutting a GitHub Release. Workflow: *plan → host → tag → publish → announce*. Configured via a `[workspace.metadata.dist]` table in `Cargo.toml`.

### [`cargo-binstall`](https://github.com/cargo-bins/cargo-binstall)

The **consumer side**: lets end users install a prebuilt binary with `cargo binstall <crate>` instead of `cargo install` (which compiles from source). Reads a manifest from crates.io or a GitHub Release and downloads the matching archive.

```bash
cargo binstall ripgrep    # gets a prebuilt binary in seconds, not a 5-minute build
```

The conventional pairing: **`cargo-dist` produces GitHub Releases with `binstall`-compatible manifests; users install via `cargo binstall`.** This is the 2025 standard for shipping Rust CLIs and standalone binaries.

### GitHub Releases + cross

For projects not using cargo-dist, the simpler approach is a CI matrix that `cross build --release` for each target, then uploads the binaries to a GitHub Release on tag.

## 4. Library publishing

For libraries, "deployment" is `cargo publish` to crates.io:

```bash
cargo publish --dry-run        # verify
cargo publish                  # actually publish (irreversible for a given version)
```

[`cargo-release`](https://crates.io/crates/cargo-release) automates the version-bump → tag → publish → push workflow; it is the standard tool for serious library maintenance.

## 5. Docker builds

Multi-stage Dockerfiles are the convention — compile with the full toolchain, copy only the static binary into a minimal runtime image:

```dockerfile
# Build stage
FROM rust:1.78 AS build
WORKDIR /app
COPY . .
RUN cargo build --release

# Runtime stage — minimal, non-root
FROM debian:bookworm-slim AS runtime
RUN useradd --create-home nonroot
WORKDIR /app
COPY --from=build /app/target/release/my_app /usr/local/bin/my_app
USER nonroot
ENTRYPOINT ["my_app"]
```

Community-recommended base images:

- **Static Linux binaries:** `rust:alpine` with `RUSTFLAGS='-C target-feature=+crt-static'`, or the `distroless` runtime images.
- **Scratch runtime:** for fully static binaries, `FROM scratch` + `COPY --from=build`.
- Always run as a **non-root user** (`USER nonroot`).
- Avoid `latest`; pin a specific Rust version.

## 6. Feature flags and dark launches

Rust has no built-in feature-flag system. The recognized patterns:

| Approach | Crate / mechanism |
|---|---|
| **Compile-time features** | Cargo `[features]` + `#[cfg(feature = "...")]` — best for things that genuinely change the binary (e.g., choosing a TLS backend). |
| **Runtime config** | the [`config`](https://crates.io/crates/config) crate with env-var / file flags — best for toggling behavior without redeploying. |
| **Managed services** | SDKs for LaunchDarkly, Unleash, Flagsmith, OpenFeature (the [`open-feature`](https://crates.io/crates/open-feature) Rust SDK is the emerging standard abstraction). |

Convention: keep secrets out of feature-flag definitions; prefer env-var-driven flags for early projects, managed services once you need targeting rules.

## 7. 12-factor for Rust services

The [Twelve-Factor App](https://12factor.net/) maps cleanly to Rust:

| Factor | Rust practice |
|---|---|
| I. Codebase | one repo per deployable; workspaces for multi-binary. |
| II. Dependencies | `Cargo.toml` explicit; no system-wide installs. |
| III. Config | env vars via `config` / `std::env`; never bake into binary. |
| IV. Backing services | treat DB/queue/SMTP as attached resources (URLs in env). |
| V. Build, release, run | `cargo build --release` → artifact → ship; same artifact across envs. |
| VI. Processes | stateless processes; state in DB/cache. |
| VII. Port binding | `0.0.0.0:$PORT` from env; the binary is the server (`axum`, `actix-web`). |
| VIII. Concurrency | scale via processes/containers; tokio for in-process concurrency. |
| IX. Disposability | graceful shutdown (`tokio::signal::ctrl_c`); fast startup. |
| X. Dev/prod parity | same Docker image in dev and prod. |
| XI. Logs | write to stdout as structured JSON (`tracing` / `tracing-subscriber`); never a log file. |
| XII. Admin processes | one-off `cargo run --bin …` with the same release binary + env. |

## 8. Observability of the pipeline

- Pipeline health: GitHub Actions itself; integrate `cargo-llvm-cov` for coverage reports (Codecov / Coveralls).
- Deployment markers: emit a structured log line (`{"event":"deploy","version":"…","commit":"…"}`) at startup so production logs can be filtered by release.
- Smoke tests: after deploy, hit `/healthz` (cross-reference Section 7 — Error Handling).

## 9. Release and rollback conventions

- **Artifact promotion:** the same release artifact moves dev → staging → prod; only configuration changes. Never rebuild per environment.
- **Database changes:** coordinated with expand/contract migrations (Section 23); a rollback must never require a schema revert.
- **Roll back by redeploying the previous artifact tag** (the binary is immutable; this is safe). Roll *forward* to fix when feasible.

## Completion checklist

- [x] Dominant CI platform (GitHub Actions) and conventional pipeline documented.
- [x] Sample GitHub Actions workflow included.
- [x] Cross-compilation in CI documented.
- [x] Binary release tooling (cargo-dist, cargo-binstall, GitHub Releases) documented.
- [x] Library publishing (`cargo publish`, `cargo-release`) documented.
- [x] Docker multi-stage build documented.
- [x] Feature-flag patterns documented.
- [x] 12-factor mapping for Rust documented.
- [x] Release and rollback conventions documented.

### References

- dtolnay/rust-toolchain action — https://github.com/dtolnay/rust-toolchain
- Swatinem/rust-cache — https://github.com/Swatinem/rust-cache
- cargo-dist — https://github.com/axodotdev/cargo-dist
- cargo-binstall — https://github.com/cargo-bins/cargo-binstall
- `cross` — https://github.com/cross-rs/cross
- cargo-release — https://github.com/crate-ci/cargo-release
- cargo-nextest — https://nexte.st/
- The Twelve-Factor App — https://12factor.net/
