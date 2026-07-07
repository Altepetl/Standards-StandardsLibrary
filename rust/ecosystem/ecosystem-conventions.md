---
title: Rust - Ecosystem Conventions
status: draft
version: 0.0.1
created: 2026-07-06
updated: 2026-07-06
---

# Rust — Ecosystem Conventions

Covers the conventions specific to the Rust ecosystem that do not fit into the language-rule sections. Rust's ecosystem is young, fast-moving, and centered on a single toolchain (`cargo` + `rustup`) and a single registry (`crates.io`). Most "conventions" here are de facto community practice rather than standards imposed by a governing body.

> **Catalog, not mandate.** Where the ecosystem has several recognized alternatives (e.g., several "logging" crates), they are documented side by side. Selection happens in StandardBuilder.

## 1. Package registry and metadata

### crates.io is the package registry

- **Registry:** [crates.io](https://crates.io) is the canonical, official package registry for Rust. It is operated by the Rust Foundation. There is no fragmented multi-registry landscape comparable to npm's public/private split; crates.io is the default.
- **Mirrors / private registries:** Cargo supports alternate registries (e.g., Cloudsmith, AWS CodeArtifact, JW Platform, a self-hosted `cargo-registry`), but they are rare and opt-in via `.cargo/config.toml`'s `[registries]` table. Most projects never need one.
- **Documentation hosting:** [docs.rs](https://docs.rs) automatically builds and hosts rustdoc for every published crate; it is the canonical place to read crate documentation. crates.io links to it.
- **Discovery auxiliary sites:** [lib.rs](https://lib.rs) (curated) and [crates.io](https://crates.io) are the two commonly used discovery sites; `cargo search` queries crates.io.

### `Cargo.toml` is the manifest

The manifest is both the build script and the package metadata file. Key conventions:

```toml
[package]
name = "my_crate"          # crates.io naming: lowercase, digits, - or _ (regex [a-z0-9_-]+)
version = "0.4.2"           # SemVer 2.0; crates.io rejects anything non-SemVer
edition = "2021"           # pin a Rust edition (2015 / 2018 / 2021 / 2024)
rust-version = "1.74"      # optional MSRV (Minimum Supported Rust Version)
authors = ["Jane Doe <jane@example.com>"]
license = "MIT OR Apache-2.0"   # SPDX expression; the de facto Rust default
description = "Short one-line summary."
repository = "https://github.com/org/my_crate"
homepage = "https://my_crate.example.com"     # optional, distinct from repository
documentation = "https://docs.rs/my_crate"    # optional; defaults to docs.rs
readme = "README.md"
keywords = ["parser", "async"]   # max 5, used for crates.io discovery
categories = ["parsing", "asynchronous"]  # must come from the fixed crates.io taxonomy
exclude = ["tests/fixtures/", ".github/"]  # files stripped from the published .crate

[dependencies]
serde = { version = "1.0", features = ["derive"] }
```

### Cargo.lock

- **Applications and binaries:** commit `Cargo.lock`. It pins the *full* dependency graph (including transitive deps) for reproducible builds.
- **Libraries:** conventionally do **not** commit `Cargo.lock`; consumers resolve their own graph. This is the long-standing convention ([Cargo FAQ](https://doc.rust-lang.org/cargo/faq.html#why-do-binaries-have-cargolock-in-version-control-but-not-libraries)) and the default behavior of `cargo new --lib`.
- `Cargo.lock` is generated/updated by `cargo build`, `cargo update`, and `cargo fetch`.

### `rust-toolchain.toml`

A pinned toolchain file (covered in Section 19 — Development Toolchain) lets a project pin a specific compiler release and components so every contributor and CI runner uses the same toolchain.

## 2. SemVer culture and the 0.x churn

Rust follows **Semantic Versioning 2.0** strictly, and `cargo` enforces the SemVer rules when resolving versions (`^` is the default operator: `^1.2.3` means `>=1.2.3, <2.0.0`).

However, the Rust community has a distinctive **0.x convention** that surprises newcomers:

- **`0.x.y` is the pre-1.0 zone.** In SemVer proper, anything `0.x` may break on any `0.z` bump. The Rust community takes this seriously: bumping `0.1.4 → 0.2.0` is *expected* to be a breaking change, and downstream crates that depend on `=0.1.4` will not auto-upgrade.
- `cargo`'s default `^` operator treats `0.2.3` as `>=0.2.3, <0.3.0` (only the **first non-zero component** matters). This means: a `0.x` crate *can* ship breaking changes on minor bumps, and consumers must opt into them.
- `1.0.0` is treated as a stability commitment; many widely-used crates (serde, tokio, clap, anyhow) reached 1.0 and then hold backward compatibility indefinitely.
- The Rust API Guidelines contain an entire chapter on [SemVer compatibility](https://rust-lang.github.io/api-guidelines/naming.html#c-naming) and the cargo team maintains the [SemVer Compatibility](https://doc.rust-lang.org/cargo/reference/semver.html) reference enumerating which changes are breaking.

> **Consequence:** when consuming `0.x` crates, pin them precisely in CI-critical paths and review changelogs on every minor bump. When authoring a `0.x` crate, document in the README that breaking changes are expected until 1.0.

## 3. The "small composable crates" philosophy

Rust's compilation model — each crate is a separate compilation unit, and generics/monomorphization reward small boundaries — has produced an ecosystem of **small, focused, composable crates** rather than monolithic frameworks.

- The Go-style "batteries-included standard library" is explicitly *not* the Rust philosophy. The Rust standard library is deliberately small (`std` covers core types, I/O, threads, sync, and not much more): networking, HTTP, JSON, async runtime, regex, random numbers, and logging are all in the ecosystem.
- The reference on this is the RFC discussion around the [Rust platform](https://rust-lang.github.io/rfcs/1242-rust-platform-spec.html) and the [libs team's "Rust platform" notion](https://std-dev-guide.rust-lang.org/introduction.html): crates maintained under `rust-lang/` that are de facto standards even though they live outside `std`.

### De facto "Rust platform" crates (rust-lang/* org)

These crates are maintained under the `rust-lang` GitHub organization and are treated as quasi-standard, even though they are not part of `std`:

| Crate | Domain |
|---|---|
| [`log`](https://crates.io/crates/log) | The logging facade every other crate writes against |
| [`regex`](https://crates.io/crates/regex) | Regular expressions |
| [`rand`](https://crates.io/crates/rand) + `rand_core` | Random number generation |
| [`uuid`](https://crates.io/crates/uuid) | UUID generation/parsing |
| [`bytes`](https://crates.io/crates/bytes) | Efficient byte buffer manipulation |
| [`url`](https://crates.io/crates/url) | URL parsing (WHATWG) |
| [`mime`](https://crates.io/crates/mime) | MIME types |
| [`time`](https://crates.io/crates/time) | Date and time (the newer "platform" effort) |

(See also the [Rust library team's "platform" working list](https://std-dev-guide.rust-lang.org/introduction.html).)

### "There's a crate for that"

The community idiom is "use a crate", and the [Are We Web Yet?](https://www.arewewebyet.org/), [Are We Game Yet?](https://arewegameyet.com/), [Are We GUI Yet?](https://areweguiyet.com/), [Are We IDE Yet?](https://areweideyet.com/) and [blessed.rs](https://blessed.rs/) curated indexes help newcomers navigate. Treat these as community recommendations, not endorsements.

## 4. The RustSec advisory database

- [RustSec Advisory Database](https://github.com/rustsec/advisory-db) is the community-maintained database of security vulnerabilities for crates published on crates.io.
- [`cargo-audit`](https://crates.io/crates/cargo-audit) audits `Cargo.lock` against the database; it is the de facto security-scanning tool and is run in CI by most serious projects (cross-reference Section 24 — CI/CD).
- [`cargo-deny`](https://crates.io/crates/cargo-deny) is a more comprehensive alternative that also checks licenses, banned crates, and duplicate versions.

## 5. Workspaces vs. single crate

Cargo **workspaces** (the `[workspace]` table in a root `Cargo.toml`) let a single repository hold multiple crates that share a `Cargo.lock`, a `target/` directory, and a dependency set. They are the standard structure for:

- Multi-binary services (one crate per binary + shared library crates).
- Library families (e.g., `tokio`, `serde`, the `icu` meta-crate).
- Monorepos.

```toml
# Root Cargo.toml of a workspace
[workspace]
resolver = "2"
members = ["crates/*"]

[workspace.package]
version = "0.5.0"
edition = "2021"
license = "MIT OR Apache-2.0"
rust-version = "1.74"

[workspace.dependencies]
serde = { version = "1.0", features = ["derive"] }
tokio = { version = "1", features = ["full"] }
```

> Each member crate then declares `serde.workspace = true` to inherit the workspace dependency. This is the modern idiom for keeping versions consistent across a workspace.

**When to use a single crate:** libraries and CLIs with one binary. **When to split:** when a piece of code is independently reusable, has its own release cadence, or needs to be publishable separately.

## 6. README / CHANGELOG / LICENSE conventions

### README

- The `README.md` at the crate root is mandatory for published crates (it is uploaded in the `.crate` tarball and rendered on crates.io and docs.rs).
- Community-expected sections: a one-line summary, a feature list, a quick example, installation (`cargo add`), and a link to docs.rs for the full API.
- The [Rust API Guidelines' "Documentation" chapter](https://rust-lang.github.io/api-guidelines/documentation.html) is the closest thing to a README standard.

### CHANGELOG

- [Keep a Changelog](https://keepachangelog.com/) format is the dominant convention; `CHANGELOG.md` at the repo root.
- Many crates automate this with [`cargo-release`](https://crates.io/crates/cargo-release) + Conventional Commits, or with [release-plz](https://github.com/release-plz/release-plz) (workspace-aware changelog generator).
- crates.io renders a "Recent updates" view from published versions; a human-readable `CHANGELOG.md` is still expected.

### LICENSE

- The community default license is **dual `MIT OR Apache-2.0`** — this is what `cargo new` writes by default and what the vast majority of the ecosystem uses. It is the closest thing to a license convention.
- Alternatives seen in practice: `MIT` alone, `Apache-2.0` alone, `MPL-2.0`, `BSD-3-Clause`, `GPL-3.0`, `AGPL-3.0`, `Unlicense OR MIT` (for ultra-permissive).
- The license must be declared as an SPDX expression in `Cargo.toml`'s `license` field; a `LICENSE-MIT` and `LICENSE-APACHE` file pair is the conventional layout for dual-licensed crates.

## 7. Categories and keywords on crates.io

- **`keywords`** (max 5, free-form, lowercase, alphanumeric + `-`/`_`): help search.
- **`categories`** (max 5): must come from the [fixed crates.io category taxonomy](https://crates.io/category_slugs) (e.g., `asynchronous`, `command-line-interface`, `database`, `web-programming::http-server`). The registry rejects unknown slugs.
- **`badges`:** the `[badges]` table in `Cargo.toml` controls the badges shown on the crates.io page (CI status, codecov, maintenance, etc.).

## 8. Editions and `cargo fix --edition`

- **Editions** (2015, 2018, 2021, 2024) are Rust's backward-compatibility mechanism: each edition can introduce small breaking changes *only when the crate opts in via `edition = "2024"` in `Cargo.toml`*. Older editions keep compiling forever; this is the [core contract of Rust's stability promise](https://doc.rust-lang.org/edition-guide/).
- Migration is largely automated:
  ```bash
  # 1. Bump the edition in Cargo.toml: edition = "2024"
  # 2. Apply the automatic fixes (lints that guide edition migration)
  cargo fix --edition
  # 3. Build and test
  cargo test
  ```
- Editions are a per-crate setting; a workspace can have members on different editions, and a crate on edition 2024 can depend on a crate on edition 2018 without issue.

## 9. Crate naming conventions

- crates.io naming is enforced by regex `[a-z0-9_-]+` (lowercase only). Both `kebab-case` (`my-crate`) and `snake_case` (`my_crate`) are accepted.
- **Convention:** `kebab-case` (`my-crate`) is the dominant modern preference, especially for multi-word names. `snake_case` is acceptable and was more common historically; both are widely seen.
- **Crate name vs. identifier:** the crate name uses `-`, but in Rust source it is referenced as a `snake_case` identifier (`use my_crate::...`). Cargo translates automatically.
- **Reserved prefixes:** `std-*`, `core-*`, `alloc-*`, and the names of standard library modules are reserved; do not squat them.

## 10. Module system and public API conventions

Rust's module system is explicit. The way you organize code internally often differs from how you want consumers to see it.

- **`pub use` re-exports:** It is highly idiomatic to have a deep internal module hierarchy (e.g., `src/client/http.rs`, `src/client/websocket.rs`) but expose a flat public API. This is done by re-exporting internal items in `lib.rs` or a parent module: `pub use client::http::Client;`. This keeps internal file structure flexible without breaking the public contract.
- **The `prelude` pattern:** Libraries that require consumers to import many traits or types often provide a `prelude` module. Users can then write `use my_crate::prelude::*;` to get everything needed for common usage. This should be used sparingly, primarily for traits (which must be in scope to use their methods) rather than structs. (Example: `std::io::prelude::*`).

## 11. Cross-language interop (FFI and WebAssembly)

Rust is frequently used as a drop-in replacement for C/C++ or compiled to WebAssembly. The ecosystem has strong conventions for both:

### C FFI (Foreign Function Interface)
- **`-sys` crates:** When binding to a C library (e.g., OpenSSL), the convention is to create a low-level, unsafe crate named `openssl-sys` that only contains the raw FFI declarations. A higher-level, safe wrapper crate named `openssl` is then published that depends on the `-sys` crate.
- **Tooling:** [`bindgen`](https://crates.io/crates/bindgen) is the standard for generating Rust FFI bindings from C headers. [`cbindgen`](https://crates.io/crates/cbindgen) is the standard for generating C headers from Rust code.
- **Conventions:** Use `#[no_mangle]` and `extern "C"` to export Rust functions to C. Use types from `std::ffi` (like `CString`, `CStr`, `c_char`, `c_void`) and `libc` to ensure ABI compatibility.

### WebAssembly (Wasm)
- **`wasm-bindgen`:** The [`wasm-bindgen`](https://crates.io/crates/wasm-bindgen) crate is the ecosystem standard for facilitating high-level interactions between Wasm modules and JavaScript.
- **Tooling:** [`wasm-pack`](https://rustwasm.github.io/wasm-pack/) is the standard CLI tool for building, testing, and publishing Rust-generated WebAssembly to npm.
- **Web APIs:** The `js-sys` and `web-sys` crates provide raw bindings to JavaScript standard built-ins and Web APIs, respectively.

## 12. The specification process: RFCs

- Rust evolves through **RFCs** (Request For Comments), hosted at [rust-lang/rfcs](https://github.com/rust-lang/rfcs). This is the equivalent of Python's PEPs or Java's JEPs.
- Major changes (language features, edition changes, large library additions) require a merged RFC; smaller changes go through the relevant team's PR review.
- Teams (compiler, libs, lang, cargo, infra, etc.) are the governance bodies; see [Rust Governance](https://www.rust-lang.org/governance).
- The [API Guidelines](https://rust-lang.github.io/api-guidelines/) are a community-recognized set of conventions for *designing* public Rust APIs (not a language spec).

## 13. Canonical community resources

| Resource | Link |
|---|---|
| Official forum (users) | https://users.rust-lang.org |
| Official forum (internals) | https://internals.rust-lang.org |
| Discord | https://discord.gg/rust-lang |
| Reddit | https://www.reddit.com/r/rust |
| Main conference | RustConf (and Rust Nation, RustLab, EuroRust, etc.) |
| This Week in Rust | https://this-week-in-rust.org (weekly newsletter) |
| Stack Overflow | `rust` tag |

## Completion checklist

- [x] Package registry (`crates.io`) and `Cargo.toml` metadata conventions documented.
- [x] SemVer / 0.x churn convention documented.
- [x] "Small composable crates" philosophy and the rust-lang/* "platform" crates documented.
- [x] RustSec / `cargo-audit` documented.
- [x] Workspace vs. single crate documented.
- [x] README / CHANGELOG / LICENSE conventions documented.
- [x] Categories and keywords documented.
- [x] Editions and `cargo fix --edition` documented.
- [x] Crate naming conventions documented.
- [x] Module system and public API conventions documented (pub use, prelude).
- [x] Cross-language interop documented (WASM, FFI).
- [x] RFC process and community resources documented.

### References

- The Cargo Book — https://doc.rust-lang.org/cargo/
- crates.io policy docs — https://crates.io/policies
- The Rust SemVer Compatibility reference — https://doc.rust-lang.org/cargo/reference/semver.html
- Rust API Guidelines — https://rust-lang.github.io/api-guidelines/
- RustSec Advisory Database — https://github.com/rustsec/advisory-db
- Rust Edition Guide — https://doc.rust-lang.org/edition-guide/
