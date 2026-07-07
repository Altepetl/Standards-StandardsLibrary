---
title: Rust - Development Toolchain
status: draft
version: 0.0.1
created: 2026-07-06
updated: 2026-07-06
---

# Rust — Development Toolchain

Covers the standard day-to-day development tools: the toolchain manager, the compiler and runtime, the language server, the formatter, the linter, and the supporting cargo subcommands. Rust has a notably **unified** toolchain story compared to most languages: one installer (`rustup`), one build tool (`cargo`), and a small set of components that ship with the official distribution.

## 1. rustup — the toolchain manager

[`rustup`](https://rustup.rs/) is the official toolchain manager and the recommended way to install Rust. It handles:

- **Channels:** `stable`, `beta`, `nightly`. Stable ships every 6 weeks; nightly ships nightly (and is where unstable features live, gated behind `#![feature(...)]`).
- **Per-toolchain components and targets:** e.g., `rustup component add rust-src`, `rustup target add wasm32-unknown-unknown`.
- **Per-project toolchain:** a `rust-toolchain.toml` (or legacy `rust-toolchain`) file at the project root pins the toolchain for everyone who enters the directory — `rustup` reads it automatically and switches.

```toml
# rust-toolchain.toml — pin the toolchain for a project
[toolchain]
channel = "1.78.0"            # exact stable release, or "stable", "nightly"
components = ["rustfmt", "clippy", "rust-src", "rust-analyzer"]
targets = ["wasm32-unknown-unknown", "x86_64-unknown-linux-gnu"]
profile = "default"           # "default" or "minimal"
```

> Putting a `rust-toolchain.toml` in the repo is the recommended way to ensure reproducible builds across developers and CI. Pin a *specific* stable version (e.g., `1.78.0`) for reproducibility, or `stable` if you want to track the latest.

Common commands:

```bash
rustup default stable
rustup toolchain install nightly
rustup component add rust-analyzer clippy rust-src
rustup target add wasm32-unknown-unknown
rustup update                       # update all installed toolchains
rustup show                         # what is installed / active
```

## 2. The compiler, runtime, and `cargo`

- **`rustc`** is the compiler. You almost never invoke it directly — `cargo` does.
- **Runtime:** Rust compiles to native code; there is no VM and no GC. A Rust program is a standalone binary. (For WebAssembly, the runtime is the WASM host.)
- **`cargo`** is the build tool, package manager, test runner, doc builder, and publishing tool. It is the single entry point for nearly everything:

```bash
cargo new my_app && cd my_app     # scaffold a binary (or --lib for a library)
cargo build                       # debug build → target/debug/
cargo build --release             # optimized build → target/release/
cargo run                         # build and run
cargo test                        # run unit + integration tests
cargo check                       # fast type-check without codegen (the inner loop)
cargo bench                       # run benchmarks (nightly-only built-in; otherwise criterion)
cargo doc --open                  # build rustdoc and open in a browser
cargo fmt                         # format with rustfmt
cargo clippy                      # run the linter
cargo add serde                   # add a dependency (rewrites Cargo.toml)
cargo update                      # refresh Cargo.lock within SemVer bounds
cargo publish                     # publish to crates.io
cargo tree                        # show the dependency graph
```

## 3. Components

`rustup` ships Rust as a set of **components**:

| Component | Purpose |
|---|---|
| `rustc` | The compiler. |
| `cargo` | The build tool / package manager. |
| `rust-std` | The standard library for each target. |
| `rustfmt` | The official formatter (covered in Section 13). |
| `clippy` | The official linter (~700 lints; covered in Section 13). |
| `rust-docs` | The offline Rust documentation (also at https://doc.rust-lang.org). |
| `rust-src` | The source of the standard library — needed by rust-analyzer for inline std source and by `cargo build -Z` build-std workflows. |
| `rust-analyzer` | The LSP server (see below). |
| `miri` | An interpreter for unsafe-code UB detection (nightly-only). |
| `rls` | **Deprecated** — the predecessor LSP; do not use. |

Install components with `rustup component add <name>`.

## 4. rust-analyzer — the language server

[`rust-analyzer`](https://rust-analyzer.github.io/) is the official LSP implementation (it replaced `rls`). Features: code completion, go-to-definition, inline type hints, rename refactoring, inlay hints, expand macro, run/debug test from gutter.

- Install: `rustup component add rust-analyzer` (now bundled in the default profile).
- It is the engine behind Rust support in VS Code, Neovim, Emacs, Vim, Sublime, Kate, and the JetBrains plugin.
- Recommended VS Code extension: the official **`rust-lang.rust-analyzer`** extension (not the legacy `rust` extension).

### Sample user settings (VS Code `settings.json`)

```jsonc
{
  // Use the rust-analyzer LSP (not the legacy rust mode)
  "rust-analyzer.linkedProjects": ["./Cargo.toml"],

  // Recommended rust-analyzer checks
  "rust-analyzer.check.command": "clippy",            // run clippy on save
  "rust-analyzer.check.features": "all",
  "rust-analyzer.cargo.features": "all",
  "rust-analyzer.cargo.noDefaultFeatures": false,

  // Inlay hints (types, parameter names, lifetimes)
  "rust-analyzer.inlayHints.bindingModeHints.enable": true,
  "rust-analyzer.inlayHints.chainingHints.enable": true,
  "rust-analyzer.inlayHints.closureReturnTypeHints.enable": "always",
  "rust-analyzer.inlayHints.lifetimeElisionHints.enable": "always",

  // Format on save with rustfmt
  "[rust]": {
    "editor.defaultFormatter": "rust-lang.rust-analyzer",
    "editor.formatOnSave": true
  },

  // Auto-run cargo check on save
  "rust-analyzer.checkOnSave": true
}
```

A `rust-project.json` is rarely needed; rust-analyzer auto-discovers the workspace from `Cargo.toml`.

## 5. IDE experience

| Editor / IDE | Status |
|---|---|
| **VS Code** | First-class via the `rust-lang.rust-analyzer` extension. The most common Rust editor. |
| **RustRover** (JetBrains) | Standalone JetBrains IDE for Rust, free for non-commercial use (2024+). Uses its own analysis engine with rust-analyzer-style features; the IntelliJ Rust plugin still powers CLion/IntelliJ. |
| **Neovim** | First-class via `nvim-lspconfig` + `rust-analyzer`; `crates.nvim` adds Cargo.toml version hints; `rustaceanvim` is a popular bundle. |
| **Emacs** | `lsp-mode` or `eglot` + rust-analyzer; `rustic` package. |
| **Helix** | Built-in LSP, rust-analyzer works out of the box. |

The **official recommendation** of the Rust project is rust-analyzer + any editor that speaks LSP.

## 6. cargo-watch — auto-rebuild

[`cargo-watch`](https://crates.io/crates/cargo-watch) watches the source tree and re-runs a cargo command on change. The canonical inner loop:

```bash
cargo install cargo-watch
cargo watch -x check -x test      # re-run `cargo check` then `cargo test` on every save
cargo watch -x "run --bin api"    # restart the server on change
```

Alternatives: [`bacon`](https://crates.io/crates/bacon) (TUI-focused, clippy and nextest aware), [`watchexec`](https://crates.io/crates/watchexec) (generic).

## 7. cargo-edit — `cargo add` / `cargo upgrade`

[`cargo-edit`](https://crates.io/crates/cargo-edit) historically provided `cargo add`, `cargo rm`, `cargo upgrade`, `cargo set-version`. **As of Rust 1.62, `cargo add` and `cargo rm` ship with cargo itself** — `cargo-edit` is now primarily used for `cargo upgrade` (bump versions in `Cargo.toml`) and `cargo set-version`.

```bash
cargo add serde --features derive     # adds the dependency + feature
cargo add tokio --features full
cargo rm old-crate
cargo upgrade                         # bump all deps to latest within SemVer (cargo-edit)
```

## 8. cargo-expand — view macro output

[`cargo-expand`](https://crates.io/crates/cargo-expand) prints the result of macro expansion (requires nightly `rustfmt`):

```bash
cargo +nightly expand
cargo +nightly expand ::path::to::fn
```

Essential for debugging `#[derive]`, `#[async_trait]`, and large declarative macros.

## 9. cargo-metadata — introspection

`cargo metadata` emits a JSON description of the workspace (crates, dependencies, features, versions). It is the canonical machine-readable workspace description, consumed by tooling such as `cargo-deny`, `cargo-about` (SBOM), and IDEs.

```bash
cargo metadata --format-version=1 --no-deps
```

## 10. Debugging

- **LLDB** is the debugger on macOS/Linux; **GDB** also works via the `rust-gdb` / `rust-lldb` wrappers that pretty-print Rust types.
- **VS Code:** the `vadimcn.vscode-lldb` (CodeLLDB) extension is the recommended debugger; rust-analyzer can wire up launch configurations.
- **DAP:** rust-analyzer and CodeLLDB both implement the Debug Adapter Protocol.
- **`dbg!` macro:** built into std, prints `‹expr› = ‹value›` to stderr and returns the value — the idiomatic quick-and-dirty debug helper:

```rust
let x = 42;
dbg!(x);              // prints [src/main.rs:2:5] x = 42
let y = dbg!(x * 2);  // prints AND assigns
```

## 11. Recommended setup summary

For a new contributor to a Rust project, the conventional setup is:

```bash
# 1. Install rustup (which installs stable + cargo + rustfmt + clippy)
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh

# 2. In the project:
rustup show                  # reads rust-toolchain.toml, installs what's missing
rustup component add rust-analyzer rust-src

# 3. Install a few widely-used cargo subcommands
cargo install cargo-watch cargo-edit cargo-expand

# 4. Open in VS Code with the rust-lang.rust-analyzer extension
code .
```

## Completion checklist

- [x] Toolchain manager (`rustup`), channels, components, per-project pinning documented.
- [x] Compiler, runtime, and `cargo` documented.
- [x] Sample `rust-toolchain.toml` provided.
- [x] rust-analyzer (LSP) and sample IDE settings documented.
- [x] Editor/IDE landscape documented.
- [x] cargo-watch, cargo-edit, cargo-expand, cargo-metadata documented.
- [x] Debugging documented.
- [x] Recommended setup summarised.

### References

- The rustup book — https://rust-lang.github.io/rustup/
- The Cargo Book — https://doc.rust-lang.org/cargo/
- rust-analyzer Manual — https://rust-analyzer.github.io/manual.html
- The rustdoc book — https://doc.rust-lang.org/rustdoc/
- Configuring a Rust environment — https://doc.rust-lang.org/book/ch01-01-installation.html
