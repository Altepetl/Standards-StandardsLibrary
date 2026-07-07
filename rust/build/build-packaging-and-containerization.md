---
title: Rust - Build, Packaging, and Containerization
status: draft
version: 0.0.1
created: 2026-07-06
updated: 2026-07-06
---

# Rust — Build, Packaging, and Containerization

> Cargo is the **single entry point** for building, testing, running, and packaging Rust. This section covers producing the artifact; managing which dependencies are used is in `rust/dependencies/dependency-management.md`. Canonical source: The Cargo Book — https://doc.rust-lang.org/cargo/.

## 1. Cargo as the single build entry point

| Command | Purpose |
|---|---|
| `cargo build` | Compile (debug profile). |
| `cargo build --release` | Compile with optimizations. |
| `cargo run` | Build and run a binary target. |
| `cargo test` | Build and run tests. |
| `cargo check` | Type-check without codegen — fast feedback loop. |
| `cargo bench` | Run benchmarks (built-in harness or `#[bench]`, usually replaced by `criterion`). |
| `cargo doc --open` | Build and open rustdoc. |
| `cargo publish` | Publish a crate to crates.io. |
| `cargo install` | Install a binary crate from crates.io or a git/path source. |

✅ One entry point (`cargo`) for the whole lifecycle — this is a major Rust strength.

❌ Hand-rolling `rustc` invocations or Makefiles that wrap cargo opaquely — hides the standard interface from contributors.

## 2. Profiles — `[profile.dev]` / `[profile.release]`

Cargo profiles control compiler flags. Defaults:

| Setting | `dev` (default) | `release` |
|---|---|---|
| `opt-level` | `0` | `3` |
| `debug` | `true` (full) | `false` (none) |
| `lto` | `false` | `false` |
| `codegen-units` | `256` | `16` |
| `panic` | `unwind` | `unwind` |
| `incremental` | `true` | `false` |
| `strip` | `false` | `false` |

### Tuning for release binaries

Common release optimizations (all optional, trade-off: longer compile time):

```toml
[profile.release]
opt-level = 3
lto = "fat"           # or "thin" for a faster, slightly less optimal LTO
codegen-units = 1     # better optimization, slower compile
panic = "abort"       # smaller binary, no unwinding
strip = "symbols"     # or true / "debuginfo"
```

✅ For a size-critical CLI/embedded target:
```toml
[profile.release]
opt-level = "z"     # optimize for size
lto = true
codegen-units = 1
panic = "abort"
strip = true
```

❌ Setting `panic = "abort"` then relying on `std::panic::catch_unwind` to recover — it won't work in an abort binary.

### Trade-offs documented side by side

| Flag | Effect | Cost |
|---|---|---|
| `lto = "fat"` | Whole-program optimization, smaller/faster binary | Long link time (minutes for big crates). |
| `lto = "thin"` | Module-grouped LTO; most of the benefit | Much faster than fat. |
| `codegen-units = 1` | Best optimization | Slowest compile. |
| `panic = "abort"` | Smaller binary, no unwind tables | No unwinding, no `catch_unwind`, aborts on panic. |
| `strip = true` | Smaller binary | No symbol names in crash reports (keep a separate `.pdb`/`*.dwarf` for debugging). |

## 3. Cross-compilation — the `cross` tool and target triples

Rust cross-compiles via **target triples** (e.g., `x86_64-unknown-linux-gnu`, `aarch64-apple-darwin`, `x86_64-pc-windows-msvc`, `wasm32-unknown-unknown`).

```bash
rustup target add x86_64-unknown-linux-musl
cargo build --target x86_64-unknown-linux-musl --release
```

### `cross` (rust-embedded/cross)
**cross** is the recognized helper that bundles cross-compilation toolchains in Docker images, removing the pain of configuring linkers and system libraries:

```bash
cargo install cross --git https://github.com/cross-rs/cross
cross build --target arm-unknown-linux-gnueabihf --release
```

| Approach | Trade-off |
|---|---|
| Native `cargo build --target` | No extra tool; requires you to install the right linker/sysroot (e.g., `musl-tools`, mingw). |
| `cross` | Docker-based, zero-config for common targets; needs Docker. |
| `cargo-zigbuild` | Uses Zig's cross toolchain as the linker; popular for producing musl/macOS/Windows binaries from Linux. |

## 4. `cargo publish` to crates.io

```bash
cargo login <token>            # one-time; stores in ~/.cargo/credentials.toml
cargo publish --dry-run        # validate packaging
cargo publish                  # uploads to crates.io
```

crates.io rules:
- Package name and version must be **unique and never reused**. Once `1.2.3` is published, it cannot be republished (use `cargo yank` for soft removal).
- The published artifact is the contents of the package (everything under the package root unless excluded in `Cargo.toml` `include`/`exclude`).
- `Cargo.lock` is **not** included in a published library crate's package.

✅ Set `publish = false` in `Cargo.toml` for crates you never intend to publish (internal crates):
```toml
[package]
publish = false
```

❌ Publishing a crate with secrets or a `/target/` directory inside — use `include`/`exclude` and review `cargo package --list`.

## 5. Binary distribution

| Tool / mechanism | Role |
|---|---|
| **cargo-dist** (axodotdev/cargo-dist) | Generates CI that builds and uploads platform-specific binaries to GitHub Releases on tag; modern, declarative. |
| **cargo-binstall** | Lets users install a prebuilt binary instead of compiling from source (`cargo binstall <crate>`), falling back to `cargo install` if no binary exists. |
| **GitHub Releases** | The conventional place to attach cross-compiled binaries and checksums. |
| **Homebrew / AUR / Snap / Flatpak / winget** | OS-specific distribution channels for popular Rust CLIs. |
| **crates.io** | The source-distribution default (`cargo install <crate>` compiles from source). |

✅ For a CLI shipped to non-Rust users, use **cargo-dist** + **cargo-binstall** so users get a prebuilt binary via `cargo binstall` or a GitHub Release asset.

❌ Expecting end users to `cargo install` a 5-minute-compile CLI with no prebuilt binary.

## 6. Docker best practices for Rust

Rust binaries are statically or dynamically linkable; the dominant pattern is the **multi-stage build**: compile in a full toolchain image, copy the resulting binary into a minimal runtime image.

### Base image alternatives (documented side by side)

| Runtime image | Trade-off |
|---|---|
| `debian:bookworm-slim` (glibc) | Smallest glibc image; runs dynamically-linked `gnu` binaries if you install the right libs. Familiar. |
| `alpine` (musl) | Tiny base; requires a musl-linked binary (`*-musl` target). DNS quirks with musl in some setups. |
| `distroless` (gcr.io/distroless/cc-debian12) | No shell, no package manager — smallest secure glibc runtime. |
| `scratch` | Truly empty; requires a fully static binary (`*-musl` + static, or static-`musl`). Most secure, smallest image. |

### ✅ Multi-stage Dockerfile (canonical Rust pattern)

```dockerfile
# ---- Build stage ----
FROM rust:1.85-bookworm AS build
WORKDIR /app

# Cache dependencies: copy manifests first, build deps, then copy source.
COPY Cargo.toml Cargo.lock ./
COPY src/ ./src/

# Build a static-ish release binary. Use the gnu target by default;
# switch to musl (rustup target add ...-musl) for a static binary.
RUN cargo build --release --locked

# ---- Runtime stage ----
FROM debian:bookworm-slim AS runtime
# Install only the runtime libs the binary needs (often none for static binaries).
RUN apt-get update && apt-get install -y --no-install-recommends \
    ca-certificates libssl3 \
    && rm -rf /var/lib/apt/lists/*

# Non-root user
RUN useradd --create-home --uid 10001 appuser
WORKDIR /home/appuser

COPY --from=build /app/target/release/myapp /usr/local/bin/myapp
USER appuser

# Drop all Linux capabilities via the runtime; expose nothing extra.
EXPOSE 8080
ENTRYPOINT ["/usr/local/bin/myapp"]
```

### ✅ Minimal static binary on `scratch`

```dockerfile
FROM rust:1.85-bookworm AS build
RUN rustup target add x86_64-unknown-linux-musl
WORKDIR /app
COPY . .
RUN cargo build --release --target x86_64-unknown-linux-musl --locked

FROM scratch
COPY --from=build /app/target/x86_64-unknown-linux-musl/release/myapp /myapp
# scratch has no TLS root store; copy CA certs if you make TLS calls.
COPY --from=build /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/
USER 10001:10001
ENTRYPOINT ["/myapp"]
```

### musl vs glibc

| Libc | When | Notes |
|---|---|---|
| **glibc** (`*-gnu`) | Default Linux target; dynamic linking; full ecosystem compat. | Cannot be fully static (NSS, dlopen); runtime image needs the lib. |
| **musl** (`*-musl`) | Static binaries, tiny containers, Alpine. | Some crates (`openssl`) need `vendored` feature or `musl-tools`; DNS resolver differs. |

✅ For TLS in a static binary, use **rustls** (pure-Rust) instead of OpenSSL to avoid the OpenSSL/musl pain.

❌ Shipping the full `rust:1.85` build image (1GB+) as the runtime image — defeats container size benefits.

❌ Running as root in the runtime stage — create a non-root `USER`.

## 7. `cargo metadata`, `build.rs`, and `links`

- `cargo metadata` (or the `cargo_metadata` crate) produces structured JSON of the dependency graph; used by tooling.
- `build.rs` (build script) runs before compilation; used to build C libraries, generate code, set `cargo:rustc-link-lib`, query environment. Declared via `build = "build.rs"` in `[package]`.
- `links = "foo"` in `[package]` declares native linkage to a system library `libfoo` and prevents two versions of the wrapper.

✅ Keep `build.rs` minimal and side-effect-free beyond generating code/setting rustc directives; cache its outputs.

❌ A `build.rs` that shells out to the network or takes minutes — kills build portability and reproducibility.

## 8. Environment Configuration Conventions (12-Factor)

Following the **12-Factor App** methodology, Rust applications should read their configuration from the environment at runtime, rather than hardcoding values or relying on complex configuration files baked into the artifact. This ensures that the same artifact can be deployed across multiple environments.

### Runtime vs Build-time Environment Variables

| Approach | Use Case | Mechanism |
|---|---|---|
| **Runtime** | Deployment config (DB URLs, ports, secrets). | `std::env::var("DATABASE_URL")` or the `config` crate. |
| **Build-time** | Embedding version info, compile-time flags. | `env!("CARGO_PKG_VERSION")` or `option_env!("...")`. |

✅ Use the `dotenvy` crate for local development. It automatically loads `.env` files into the environment.
✅ Add `.env` to `.gitignore`. Never commit `.env` files containing secrets or environment-specific configuration to source control.
✅ Create a `.env.example` or `.env.sample` template file and commit it, showing developers what keys are required.
✅ For complex configuration, use a dedicated configuration library like `config` or `figment`, which can seamlessly merge environment variables, `.env` files, and JSON/TOML defaults.

❌ Using the `env!` macro for database connection strings or API keys. This bakes the secret or environment-specific configuration directly into the compiled artifact, violating the principle of "build once, deploy anywhere".

## 9. Artifact Naming and Versioning Conventions

When building Rust artifacts for distribution, naming conventions depend on whether the artifact is a binary, a Rust library, or a C-ABI library.

### Default Cargo Naming

- **Binaries:** Match the `[[bin]]` name or `[package.name]`. Output is `myapp` (Linux/macOS) or `myapp.exe` (Windows).
- **Rust Libraries (`rlib`):** `lib<name>.rlib` (e.g., `libserde.rlib`).
- **C-ABI Libraries:** `lib<name>.so` (Linux), `lib<name>.dylib` (macOS), `<name>.dll` (Windows), or `lib<name>.a` (static).

### Distribution Naming Conventions

When packaging artifacts for GitHub Releases or external distribution (e.g., via `cargo-dist`), the archive should definitively state the program name, version, and target platform.

✅ **Standard distribution artifact format:**
`<name>-v<version>-<target_triple>.<ext>`

Examples:
- `myapp-v1.2.3-x86_64-unknown-linux-gnu.tar.gz`
- `myapp-v1.2.3-x86_64-pc-windows-msvc.zip`
- `myapp-v1.2.3-aarch64-apple-darwin.tar.gz`

✅ **Dynamic libraries versioning:** If you are distributing a `cdylib` (.so), you may need to manually post-process the build to include the `soname` version (e.g., `libmyapp.so.1`) since Cargo does not automatically append version numbers to shared objects natively.

## 10. Reproducible builds & determinism

- `--locked` / `--frozen` to pin dependencies.
- `rust-toolchain.toml` to pin the compiler.
- `SOURCE_DATE_EPOCH` is respected by recent rustc for reproducible embedded timestamps.
- For SBOM and signing, see `rust/cicd/` (planned): `cargo cyclonedx` and `cargo auditable` (embeds the dependency list into the binary for later auditing).

## Sources

| Topic | Source |
|---|---|
| Cargo Book | https://doc.rust-lang.org/cargo/ |
| Profiles | https://doc.rust-lang.org/cargo/reference/profiles.html |
| `cargo publish` | https://doc.rust-lang.org/cargo/commands/cargo-publish.html |
| cross | https://github.com/cross-rs/cross |
| cargo-zigbuild | https://github.com/rust-cross/cargo-zigbuild |
| cargo-dist | https://github.com/axodotdev/cargo-dist |
| cargo-binstall | https://github.com/cargo-bins/cargo-binstall |
| cargo-auditable | https://github.com/rust-secure-code/cargo-auditable |
| build scripts | https://doc.rust-lang.org/cargo/reference/build-scripts.html |
