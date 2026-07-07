---
title: Rust - Automation Tools (Linters and Formatters)
status: draft
version: 0.0.1
created: 2026-07-06
updated: 2026-07-06
---

# Rust — Automation Tools (Linters and Formatters)

> Rust has two **officially blessed** quality tools: **rustfmt** (the formatter) and **clippy** (the linter), both maintained under `rust-lang` and shipped via `rustup` components. This is unusual strength — most languages have competing de facto tools; Rust has canonical ones. Where alternatives exist (dylint for custom lints, third-party hooks), they are documented side by side. Boundary: this section covers quality *enforcement* tooling; producing the build artifact is in `rust/build/`.

## 1. rustfmt — the standard formatter

`rustfmt` is the official Rust formatter (rust-lang/rustfmt). `cargo fmt` is the entry point. It enforces a single, opinionated style so Rust code looks identical across projects — analogous to `gofmt`.

```bash
cargo fmt            # format in place
cargo fmt -- --check # CI mode: exits non-zero if formatting would change
```

### `rustfmt.toml` configuration
Most projects use rustfmt defaults; `rustfmt.toml` overrides specific options. Recognized options include `max_width` (default 100), `hard_tabs`, `tab_spaces`, `edition`, `use_field_init_shorthand`, `imports_granularity` (nightly only), `group_imports` (nightly only).

✅ Minimal, mostly-default config:
```toml
# rustfmt.toml
edition = "2024"
max_width = 100
use_field_init_shorthand = true
```

❌ Disabling rustfmt entirely or hand-formatting — Rust culture treats `cargo fmt` as near-mandatory.

### Stability note
Some useful rustfmt options (import sorting/grouping) are **only available on nightly** rustfmt, gated behind `#![feature(...)]`. Two recognized approaches:
1. Use stable rustfmt with defaults (most projects).
2. Use a nightly rustfmt component pinned in CI for finer control. Document both.

## 2. clippy — the standard linter

`clippy` (rust-lang/rust-clippy) is Rust's official linter, distributed as a `rustup` component. Run via `cargo clippy`.

```bash
cargo clippy                              # warn on lints
cargo clippy --all-targets -- -D warnings # CI: deny all warnings
```

### Clippy lint categories

Clippy lints are grouped into categories, each with a default level:

| Group | Default level | What it catches |
|---|---|---|
| **`correctness`** | deny | Likely bugs / incorrect code. |
| **`suspicious`** | warn | Probably-bad, but not certain. |
| **`style`** | warn | Non-idiomatic style. |
| **`complexity`** | warn | Overly complex code that can be simplified. |
| **`perf`** | warn | Code that can run faster. |
| **`pedantic`** | allow | Extra-strict, opinionated checks. |
| **`restriction`** | allow | Forbids specific code patterns (very opinionated; opt-in per project). |
| **`nursery`** | allow | Experimental / still-buggy lints. Official advice: **do not enable the whole group**, cherry-pick. |

The composite group `clippy::all` enables correctness + suspicious + style + complexity + perf (the warn/deny defaults).

### Lint levels

A lint can be set to one of four levels:
| Level | Meaning |
|---|---|
| `allow` | Suppress the lint entirely. |
| `warn` | Emit a warning; compile continues. |
| `deny` | Treat as an error; compilation fails. |
| `forbid` | Like `deny`, but **cannot be overridden** by inner attributes (stronger than `deny`). |

Difference between `deny` and `forbid`: a downstream `#[allow(...)]` can relax a `deny`, but cannot relax a `forbid`.

### Configuring lint levels

Three recognized places, documented side by side:

#### (a) Crate-level via attributes in `lib.rs` / `main.rs`
```rust
#![warn(clippy::pedantic)]
#![warn(clippy::nursery)]
#![deny(clippy::correctness)]
#![allow(clippy::module_name_repetitions)]
```

#### (b) Cargo's `[lints]` table (Cargo 1.74+, idiomatic for workspace-wide config)
```toml
# Cargo.toml
[lints.clippy]
pedantic = "warn"
module_name_repetitions = "allow"
```
Workspaces can hoist this under `[workspace.lints]` and inherit with `[lints] workspace = true`.

✅ Prefer the `[lints]` table for new projects — it is version-controllable and applies across targets.

❌ Scattering `#[allow(clippy::...)]` across the codebase without justification.

#### (c) Per-item attributes
```rust
#[allow(clippy::too_many_arguments)]
fn handler(req: Request) -> Response { /* ... */ }
```

### `clippy.toml`
Some clippy lints are configurable via a `clippy.toml` (e.g., `cognitive-complexity-threshold`, `too-many-arguments-threshold`, `enum-variant-name-threshold`).

## 3. ✅ / ❌ examples

✅ Deny correctness, warn on pedantic (a common config):
```rust
#![deny(clippy::correctness)]
#![warn(clippy::pedantic)]
```

✅ Granular allow on a justified case, with a comment:
```rust
// We intentionally hold many config fields.
#[allow(clippy::too_many_arguments)]
fn new_service(a: A, b: B, c: C) -> Service { /* ... */ }
```

❌ Globally allowing correctness lints to silence a real bug:
```rust
#![allow(clippy::correctness)]   // hides bugs project-wide
```

❌ Using `forbid` then trying to relax it locally — won't compile:
```rust
#![forbid(clippy::unwrap_used)]
mod m {
    #[allow(clippy::unwrap_used)]   // ERROR: forbid cannot be relaxed
    fn f() { let _ = Option::Some(1).unwrap(); }
}
```

## 4. `cargo fmt --check` and `cargo clippy -D warnings` in CI

The near-universal CI gate:
```yaml
- run: cargo fmt -- --check
- run: cargo clippy --all-targets -- --deny warnings
- run: cargo test
```

✅ Fail the pipeline on any formatting or lint warning.

❌ Treating clippy warnings as informational and letting them accumulate — defeats the gate.

## 5. Custom lints — `dylint`

**dylint** (rust-lang/dylint... community-supported under rust-lang) lets you write and load **custom clippy-style lints** as dynamic libraries. Used for project- or organization-specific rules (e.g., banning a deprecated internal API).

```bash
cargo install dylint
dylint --path ./my-lints
```

Recognized alternative: write a clippy contribution upstream. dylint is the path for private/team lints.

## 6. Pre-commit hooks — alternatives side by side

| Tool | Role |
|---|---|
| **cargo-husky** | Tiny Rust crate that installs a git hook running `cargo test`/`cargo fmt` on commit. |
| **pre-commit (Python)** | The cross-language `pre-commit` framework with Rust hooks (rustfmt, clippy). |
| **lefthook** / **husky (Node)** | Generic git-hook managers. |
| **Just / Make targets called by hooks** | Many Rust projects use a `justfile` and call `just fmt`/`just lint` from a hook. |

| Approach | Trade-off |
|---|---|
| `cargo-husky` | Pure-Rust, zero extra runtime; minimal. |
| `pre-commit` framework | Polyglot repos, shared hooks, but requires Python. |
| No hooks, rely on CI | Simplest, but allows local commits with unformatted code. |

✅ Run rustfmt + clippy as a pre-commit hook **and** in CI (defense in depth).

❌ Only relying on hooks without a CI gate — contributors who skip hooks break the build.

## 7. Toolchain pinning — `rust-toolchain.toml`

To make local and CI builds reproducible, pin the toolchain with a `rust-toolchain.toml` file at the repo root:

```toml
[toolchain]
channel = "1.85.0"           # or "stable", "nightly-2025-03-01"
components = ["rustfmt", "clippy", "rust-src"]
targets = ["wasm32-unknown-unknown"]
profile = "minimal"
```

`rustup` reads this and installs the exact toolchain automatically. Committing this file makes `cargo`/`rustc`/`rustfmt`/`clippy` versions deterministic across all contributors and CI.

✅ Commit `rust-toolchain.toml` for reproducible builds and CI.

❌ Letting each contributor use a different `rustup default` — "works on my machine" failures.

## 8. SAST / security scanning (cross-ref)

For security-oriented static analysis, see `rust/security/` (planned). Tools include **cargo-audit** (RustSec advisories) and **cargo-deny** (licenses, bans, advisories). These complement clippy, which is correctness/style, not security-focused.

## 9. Code quality platforms

Rust integrates with the usual SaaS quality platforms (SonarQube via the rust plugin, Code Climate, Codacy). Clippy + rustfmt + cargo-audit are the local baseline; the SaaS layer is optional and project-dependent.

## Sources

| Topic | Source |
|---|---|
| rustfmt | https://github.com/rust-lang/rustfmt |
| rustfmt config | https://rust-lang.github.io/rustfmt/ |
| clippy | https://github.com/rust-lang/rust-clippy |
| clippy lints | https://doc.rust-lang.org/clippy/lints.html |
| Cargo `[lints]` | https://doc.rust-lang.org/cargo/reference/manifest.html#the-lints-section |
| clippy.toml | https://doc.rust-lang.org/clippy/configuration.html |
| dylint | https://github.com/awslabs/dylint |
| rust-toolchain.toml | https://rust-lang.github.io/rustup/overrides.html#the-toolchain-file |
