---
title: Rust - Project Architecture and Directory Layout
status: draft
version: 0.0.1
created: 2026-07-06
updated: 2026-07-07
---

# Rust — Project Architecture and Directory Layout

Rust's project model is **Cargo**, and Cargo prescribes a specific directory layout. The authoritative reference is the **Cargo Book — Package Layout** (`doc.rust-lang.org/cargo/guide/project-layout.html`). This section catalogs that layout, the module-file conventions (`mod.rs` vs `foo.rs` vs `foo/`), and the **workspace** model for multi-crate repositories.

## Core concepts

- **Package** — a directory with a `Cargo.toml` that you can build, test, and publish. Contains one or more **crates**.
- **Crate** — a single compilation unit: either a *library* (root `src/lib.rs`) or a *binary* (root `src/main.rs`).
- **Module** — a named namespace *inside* a crate; declared with `mod`, organized as files or directories.
- **Workspace** — a set of packages that share `Cargo.lock` and a `target/` directory, declared in a top-level `Cargo.toml` with a `[workspace]` section.

## The canonical Cargo layout

Per the Cargo Book, a single-crate project looks like this:

```
my_crate/
├── Cargo.toml          # manifest: name, version, deps, edition
├── Cargo.lock          # lockfile (committed for binaries, optional for libs)
├── README.md
├── LICENSE-MIT
├── LICENSE-APACHE      # dual licensing is the community norm
├── .gitignore
├── rust-toolchain.toml # optional: pin the toolchain
├── rustfmt.toml        # optional: formatter config
├── clippy.toml         # optional: clippy config
├── build.rs            # optional: build script (e.g. codegen)
├── src/
│   ├── main.rs         # binary crate root
│   ├── lib.rs          # library crate root (a package can have BOTH)
│   ├── bin/
│   │   ├── tool_a.rs   # additional binaries (cargo build builds each)
│   │   └── tool_b/
│   │       └── main.rs # binary spanning multiple files: bin/<name>/main.rs
│   └── module_name/    # see "module organization" below
│       ├── mod.rs      # or module_name.rs + module_name/ — pick one
│       └── child.rs
├── tests/
│   ├── integration.rs  # integration tests (link against the library crate)
│   └── common/
│       └── mod.rs      # shared helpers for integration tests
├── benches/
│   └── my_bench.rs     # benchmarks (require #[bench] or criterion)
├── examples/
│   └── simple.rs       # examples built and runnable via `cargo run --example`
└── .cargo/
    └── config.toml     # optional: per-project cargo config
```

### What goes where

| File / dir | Purpose | Built by |
|---|---|---|
| `Cargo.toml` | Manifest: metadata, dependencies, features, edition. | — |
| `Cargo.lock` | Exact dependency versions. | `cargo build` |
| `src/lib.rs` | Library crate root; declares the public API. | `cargo build` (lib) |
| `src/main.rs` | Binary crate root; thin wrapper over the library. | `cargo build` (bin) |
| `src/bin/*.rs` | Additional binaries. | `cargo build --bins` |
| `src/bin/<name>/main.rs` | A multi-file binary. | `cargo build --bin <name>` |
| `tests/*.rs` | Integration tests; access the crate as an external user. | `cargo test --test <name>` |
| `benches/*.rs` | Benchmarks (historically `#[bench]`; today `criterion`). | `cargo bench` |
| `examples/*.rs` | Runnable examples; shown in `cargo doc`. | `cargo run --example <name>` |
| `build.rs` | Build script (codegen, linking C, embedding assets). | runs before `cargo build` |
| `.cargo/config.toml` | Project-local cargo settings (linker, target, alias). | — |
| `rust-toolchain.toml` | Pin a channel / MSRV for the project. | — |

### Binary + library together

A common and recommended pattern is for a single package to ship **both** `src/main.rs` and `src/lib.rs`. The binary stays thin and delegates to the library:

```rust
// src/main.rs
use my_crate::run;

fn main() -> anyhow::Result<()> {
    run()
}
```

```rust
// src/lib.rs
pub fn run() -> anyhow::Result<()> { /* ... */ Ok(()) }
```

This lets integration tests (in `tests/`) and benchmarks (in `benches/`) exercise the logic through the library's public API.

## Module organization: `foo.rs` vs `foo/mod.rs`

Rust 2018+ offers **two equivalent** ways to declare a module that spans multiple files. Pick one and be consistent within a crate.

**Style A — file + directory (2018+ preferred):**

```
src/
├── lib.rs
└── parser/
    └── mod.rs            # ← wait, this is style B; see below
```

The two real styles are:

**Style A — `foo.rs` plus a `foo/` directory** (the 2018+ convention):

```
src/
├── lib.rs            # contains: `mod parser;`
├── parser.rs         # parser module body lives here
└── parser/
    ├── lexer.rs      # `mod lexer;` declared inside parser.rs
    └── ast.rs
```

**Style B — `foo/mod.rs`** (the 2015 convention, still supported):

```
src/
├── lib.rs            # contains: `mod parser;`
└── parser/
    ├── mod.rs        # parser module body lives here
    ├── lexer.rs
    └── ast.rs
```

Both compile identically. The community leans toward **Style A** (`foo.rs` + `foo/`) because it avoids the double-meaning of `mod.rs` and reduces path ambiguity in large trees. Some projects keep `mod.rs` for historical reasons. Clippy's `clippy::mod_module_files` / `clippy::self_named_module_files` lints let a project enforce one or the other.

### Module visibility

```rust
// src/lib.rs
pub mod api;          // public submodule declared in src/api.rs
mod internal;         // private submodule declared in src/internal.rs
```

Submodules are looked up relative to the parent module's file. `pub` items form the crate's public API; non-`pub` items are crate-private (the default visibility is private, not crate).

## Workspaces

A **workspace** groups multiple packages that share a `Cargo.lock` and `target/` directory. The Cargo Book recommends workspaces for any repository with more than one crate. The root `Cargo.toml` declares the workspace and *does not* define a package itself:

```toml
# /Cargo.toml — workspace root
[workspace]
resolver = "2"
members = [
    "crates/core",
    "crates/cli",
    "crates/server",
]

[workspace.package]
edition = "2024"
license = "MIT OR Apache-2.0"
version = "0.1.0"

[workspace.dependencies]
serde = { version = "1", features = ["derive"] }
anyhow = "1"
```

Member packages inherit shared settings:

```toml
# crates/core/Cargo.toml
[package]
name = "myproj-core"
edition.workspace = true
license.workspace = true
version.workspace = true

[dependencies]
serde.workspace = true
```

### Why workspaces

- One `Cargo.lock` keeps all crates on consistent dependency versions.
- One `target/` avoids rebuilding shared dependencies per crate.
- `[workspace.dependencies]` centralizes version pins; members reference them with `serde.workspace = true`.
- `resolver = "2"` (the Rust 2021+ default) fixes feature unification across the workspace — always set this on new workspaces.

### Example workspace tree

```
my_project/
├── Cargo.toml                 # [workspace] root
├── Cargo.lock
├── README.md
├── crates/
│   ├── core/                  # library crate
│   │   ├── Cargo.toml
│   │   └── src/
│   │       └── lib.rs
│   ├── cli/                   # binary crate depending on core
│   │   ├── Cargo.toml
│   │   └── src/
│   │       └── main.rs
│   └── server/                # another binary crate
│       ├── Cargo.toml
│       └── src/
│           └── main.rs
└── xtask/                     # convention: private automation binary
    ├── Cargo.toml
    └── src/
        └── main.rs
```

The `xtask/` pattern (a private workspace member invoked via `cargo run -p xtask -- <task>`) is a recognized community convention for project automation that avoids `make`/`just` dependencies.

## Where tests live

| Test kind | Location | What it can access |
|---|---|---|
| **Unit tests** | `src/foo.rs` under `#[cfg(test)] mod tests` | Private items of the module. |
| **Integration tests** | `tests/*.rs` | Only the crate's **public** API (treated as an external user). |
| **Doctests** | `///` doc comments | The crate's public API, compiled and run by `cargo test --doc`. |

```rust
// src/parser.rs
pub fn parse(input: &str) -> Ast { /* ... */ }

#[cfg(test)]
mod tests {
    use super::parse;            // unit test can see private siblings

    #[test]
    fn parses_simple() {
        assert!(parse("1+2").is_ok());
    }
}
```

```rust
// tests/end_to_end.rs
use my_crate::parse;             // integration test sees only public API

#[test]
fn end_to_end() { /* ... */ }
```

Shared helpers for integration tests live in `tests/common/mod.rs` and are pulled in via `mod common;` from each test file — the Cargo Book documents this pattern.

## `build.rs` — build scripts

A `build.rs` at the package root runs before compilation. Legitimate uses (per the Cargo Book):

- Code generation (Protobuf, SQLx, bindgen for C headers).
- Embedding assets at compile time (`include_str!`-like behavior).
- Setting custom `cfg` flags or linking native libraries.

Build scripts that do non-hermetic work (network, time-dependent logic) hurt reproducibility and caching; avoid them unless necessary.

## Standard Ignore Rules

Cargo automatically creates a basic `.gitignore` when initializing a repository, but a robust standard includes rules for build artifacts, IDEs, and environments.

```gitignore
# Generated by Cargo (compiled files and executables)
/target/

# These are backup files generated by rustfmt
**/*.rs.bk

# MSVC Windows builds of rustc generate these, which store debugging information
*.pdb

# Environment variables
.env
.env.*

# IDEs and editors
.idea/
.vscode/
*.swp
```

> **Note on `Cargo.lock`:** Historically, libraries ignored `Cargo.lock` while binaries committed it. Today, the community consensus leans heavily toward committing `Cargo.lock` in *all* projects (including libraries) to ensure reproducible CI runs, while relying on `cargo update` to test against newer dependencies.

## Architectural Patterns and Module Organization Strategies

Rust does not enforce a single architectural pattern. Instead, architecture tends to emerge based on the project's domain.

### Organization Strategies

1. **By Layer (Technical Partitioning):**
   Code is grouped by its technical role (`models/`, `controllers/`, `services/`, `repositories/`).
   * **Pros:** Familiar to developers coming from MVC frameworks. Easy to find all HTTP handlers or database queries.
   * **Cons:** Features are spread across multiple directories. Modifying a single feature requires touching files across the entire layer stack.
2. **By Feature (Domain Partitioning):**
   Code is grouped by business domain or feature (`users/`, `orders/`, `inventory/`). Each module contains its own routes, models, and business logic.
   * **Pros:** High cohesion. Easier to extract a feature into a separate microservice or crate later.
   * **Cons:** Can lead to duplicated technical scaffolding.

### Common Architectural Patterns

1. **Hexagonal Architecture (Ports and Adapters):**
   Highly idiomatic in complex Rust backends. The core business logic is isolated in a central module or crate, defined entirely by standard Rust traits (Ports). External integrations (databases, HTTP frameworks) implement these traits (Adapters) and are injected into the core.
   * *Why in Rust:* Rust's powerful trait system and zero-cost abstractions make this pattern very natural, allowing testing of the core domain without mocking HTTP or database layers.
2. **Model-View-Controller (MVC):**
   Used in traditional web frameworks, though less common in modern Rust where APIs and single-page apps dominate. The model manages data, the view handles rendering (e.g., Askama or Tera templates), and the controller binds them.
3. **Data-Oriented Design (DOD):**
   Common in game development (e.g., with Bevy ECS) and high-performance systems. Focuses on data layout in memory rather than object encapsulation, maximizing CPU cache hits.

## Framework-Specific Layouts

While Cargo defines the base layout, specific ecosystems often introduce their own conventions on top of `src/`.

### Axum / Actix-Web (Web APIs)

A typical REST API in Rust groups routing, state, and business logic. A feature-based layout is highly recommended:

```
src/
├── main.rs            # Entry point: builds the router, starts the server
├── lib.rs             # Re-exports modules for testing
├── config.rs          # Environment and app configuration
├── error.rs           # Global API error type and HTTP status mappings
├── api/               # Feature-based API modules
│   ├── mod.rs         # Combines all routes
│   ├── users/         # User domain
│   │   ├── mod.rs     # User routes, handlers, and models
│   │   └── db.rs      # Database queries for users
│   └── orders/
└── db/                # Shared database utilities and connection pooling
```

### Tauri (Desktop Applications)

Tauri applications fuse a web frontend with a Rust backend. The Rust code typically lives in a `src-tauri` directory managed by the Tauri CLI.

```
my-app/
├── package.json       # Frontend dependencies
├── src/               # Frontend source (React, Vue, Svelte)
└── src-tauri/         # Rust backend
    ├── Cargo.toml
    ├── tauri.conf.json # Tauri configuration
    ├── build.rs
    ├── src/
    │   ├── main.rs    # Tauri setup, command registration
    │   └── commands.rs# Rust functions callable from JS (#[tauri::command])
    └── icons/         # App icons
```

## Summary checklist

- [ ] Source under `src/`; binary root `src/main.rs`, library root `src/lib.rs`.
- [ ] Integration tests in `tests/`, benchmarks in `benches/`, examples in `examples/`.
- [ ] One module-file convention per crate (`foo.rs + foo/` *or* `foo/mod.rs`).
- [ ] Multi-crate repos use a `[workspace]` with `resolver = "2"`.
- [ ] `Cargo.lock` committed for binaries; commonly also committed for workspace roots.
- [ ] Thin `main.rs` delegating to `lib.rs` when shipping both.

## References

- The Cargo Book — Package Layout — `doc.rust-lang.org/cargo/guide/project-layout.html`.
- The Cargo Book — Workspaces — `doc.rust-lang.org/cargo/reference/workspaces.html`.
- The Cargo Book — Build scripts — `doc.rust-lang.org/cargo/reference/build-scripts.html`.
- The Reference — Modules — `doc.rust-lang.org/reference/items/modules.html`.
- `xtask` pattern — `github.com/matklad/cargo-xtask`.
