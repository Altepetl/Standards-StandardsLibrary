---
title: Rust - Dependency Management
status: draft
version: 0.0.1
created: 2026-07-06
updated: 2026-07-06
---

# Rust — Dependency Management

> **Catalog, not mandate.** Rust dependency management is overwhelmingly centralized on **Cargo** (the official build system and package manager) and **crates.io** (the official registry). Where the community has recognized alternatives (e.g., vendoring vs. registry, `cargo-edit` vs. built-in), this document presents them side by side with trade-offs rather than picking a winner. Boundary: this section covers the *external library lifecycle* (what to use and how to manage it); producing the build artifact is covered in `rust/build/build-packaging-and-containerization.md`.

## 1. Cargo and `Cargo.toml`

Cargo is the canonical tool, installed with the Rust toolchain via `rustup`. Every Rust package is described by a manifest named `Cargo.toml` (TOML format), and the resolver produces `Cargo.lock` (the lockfile). For a broader look at the package management ecosystem and conventions, see Section 18 (Ecosystem Conventions).

Canonical sources:
- The Cargo Book — https://doc.rust-lang.org/cargo/
- The manifest format — https://doc.rust-lang.org/cargo/reference/manifest.html
- crates.io — https://crates.io

### Dependency sections

| Section | Purpose |
|---|---|
| `[dependencies]` | Runtime dependencies for both libraries and binaries. |
| `[dev-dependencies]` | Used only in tests, examples, and benchmarks; not compiled into the released artifact. |
| `[build-dependencies]` | Used by `build.rs` build scripts; not part of the final artifact. |
| `[target.'cfg(...)'.dependencies]` | Platform/target-specific dependencies. |
| `[dependencies.<name>]` | Per-dependency table form (for features, optional, path/git overrides). |

✅ Correct separation of dependency kinds:
```toml
[package]
name = "myapp"
version = "0.1.0"
edition = "2024"
rust-version = "1.80"   # MSRV declaration (see §9)

[dependencies]
serde = { version = "1.0", features = ["derive"] }
tokio = { version = "1", features = ["full"] }

[dev-dependencies]
proptest = "1.5"
rstest = "0.23"

[build-dependencies]
# only what build.rs actually needs
```

❌ Putting test-only crates in `[dependencies]`:
```toml
[dependencies]
proptest = "1.5"      # ships to your users for no reason; bloats compile time
```

## 2. Version requirements and SemVer

crates.io packages follow **Semantic Versioning**. A *version requirement* in `Cargo.toml` is a constraint, not an exact version. Cargo's default operator is the **caret** (`^`), which allows SemVer-compatible updates.

| Requirement | Meaning | Allows |
|---|---|---|
| `serde = "1.0"` | shorthand for `^1.0` | `>=1.0.0, <2.0.0` |
| `serde = "^1.0"` | caret, explicit | `>=1.0.0, <2.0.0` |
| `serde = "~1.0"` | tilde — patch updates only | `>=1.0.0, <1.1.0` |
| `serde = "=1.0.197"` | exact pin | `1.0.197` only |
| `serde = ">=1.0, <2.0"` | range | any version in the range |
| `serde = "*"` | any version (wildcard) | **avoid** — non-reproducible |

Key Cargo SemVer facts (per the Cargo Book reference):
- `0.x.y` is treated specially: `^0.8.3` allows `>=0.8.3, <0.9.0` (the `0.x` is the "major" for pre-1.0 crates).
- `^0.0.3` allows only `0.0.3` exactly (the `0.0.x` is the major).

✅ Default caret ranges are idiomatic:
```toml
serde = "1.0"
```

❌ Pinning every dependency exactly hurts maintenance and slows ecosystem updates, and `*` is dangerous for reproducibility:
```toml
serde = "=1.0.197"   # overly strict unless security-critical
libc   = "*"          # non-reproducible builds
```

❌ Forgetting the pre-1.0 rule — `mycrate = "^0.3.1"` will **not** auto-upgrade to `0.4.0`, which is treated as a breaking major bump.

## 3. `Cargo.lock` — commit or gitignore? (the canonical rule, and the debate)

This is the most-debated Cargo topic. The guidance has two phases:

### The classic rule (long-standing, still widely cited)
- **Binaries/applications (crates with a `[[bin]]` target):** ✅ commit `Cargo.lock` for reproducible builds.
- **Libraries (crates published to crates.io for others to consume):** the lockfile is *ignored by downstream consumers*; historically the recommendation was to gitignore it.

Source: the original Cargo FAQ and GitHub issue rust-lang/cargo#315.

### The updated 2023 guidance (Cargo team blog post)
In *"Change in Guidance on Committing Lockfiles"* (blog.rust-lang.org, 2023-08-29), the Cargo team **encourages all packages to commit `Cargo.lock`, including libraries**, because:
- It improves reproduc of CI, benchmarks, and bisecting regressions.
- A library's lockfile is never used by its dependents, so committing it is harmless to consumers.

### Practical position — document both, because the community is split

| Approach | When | Rationale |
|---|---|---|
| **Always commit** (post-2023 recommendation) | Any crate, esp. with CI/benchmarks | Reproducible CI and bisection; harmless to library consumers. |
| **Classic split** (still used by many libraries) | Libraries only | Keeps the diff noise-free; downstream consumers regenerate their own lockfile anyway. |

> Note: the default `cargo new --lib` template historically added `Cargo.lock` to `.gitignore`, and there is an open proposal (rust-lang/cargo#11510) to change the template. Check the template version before assuming.

✅ Use `--locked` / `--frozen` in CI to fail fast on lockfile drift:
```bash
cargo build --locked --release
cargo test --frozen   # also forbids network access
```

❌ Silently letting `cargo update` drift the lockfile in CI without a checkpoint.

## 4. Feature flags (`features`, `default-features`)

Cargo features are the standard mechanism for conditional compilation. They are additive and combinable by design.

```toml
[dependencies]
serde = { version = "1.0", default-features = false, features = ["derive", "alloc"] }

# Optional dependency gated behind a feature
[dependencies]
sqlx = { version = "0.8", optional = true }

[features]
default = ["std"]
std = []
postgres = ["dep:sqlx"]   # "dep:" syntax (edition 2021+) to avoid implying a feature
```

Feature unification: if crate A enables `serde/derive` and crate B enables `serde/alloc`, Cargo **unions** them — features are additive. This is why the Cargo Book mandates features be *strictly additive*.

✅ Additive features that only *enable* functionality:
```toml
[features]
json = ["serde"]
tls  = ["rustls"]
```

❌ Mutually exclusive features (Cargo cannot express "X *or* Y but not both" cleanly). Documenting alternatives for this need:
- Use separate types behind `#[cfg(feature = "...")]` (idiomatic but verbose).
- Use the `configurator`-style `cfg_if!` macro.
- For truly exclusive variants, consider splitting into separate crates.

❌ Enabling `default-features = ["std"]` then a no_std consumer can't opt out — always gate `std` behind a non-default or `default` feature you allow to be disabled:
```toml
[features]
default = ["std"]
std = []
```

## 5. Workspaces (`[workspace]`)

For multi-crate repositories, a Cargo workspace shares a single `Cargo.lock`, a single `target/`, and lets dependencies be hoisted to the workspace level (Cargo 1.64+).

```toml
# /Cargo.toml (workspace root)
[workspace]
members = ["crates/*"]
resolver = "2"          # resolver v2 is the default since edition 2021; avoids feature unification surprises

[workspace.dependencies]
serde = "1.0"
tokio = "1"
```

```toml
# /crates/api/Cargo.toml
[dependencies]
serde = { workspace = true }
```

✅ One `Cargo.lock` for the whole monorepo; hoist shared deps with `[workspace.dependencies]`.

❌ Each sub-crate maintaining its own divergent lockfile — defeats the point of a workspace.

## 6. Updating dependencies

| Tool | What it does |
|---|---|
| `cargo update` | Built-in. Updates `Cargo.lock` to the latest versions allowed by the requirements in `Cargo.toml`. Does **not** modify `Cargo.toml`. |
| `cargo add` / `cargo remove` | Built-in since Cargo 1.62. Adds/removes a dependency and edits `Cargo.toml`. |
| `cargo upgrade` (cargo-edit) | The `cargo-edit` crate provides `cargo upgrade`, which **rewrites the version requirements in `Cargo.toml`** to newer numbers. Not part of rustup. |

### Update cadence conventions

While there is no single universally mandated cadence, the community recognizes the following practices:
- **Routine Updates**: Run `cargo update` on a regular cadence (e.g., weekly or bi-weekly via automated tools like Dependabot or Renovate) to keep the lockfile fresh and incorporate non-breaking patch and minor updates.
- **Major Upgrades**: Schedule time periodically (e.g., quarterly) to evaluate and apply major version upgrades (which may contain breaking changes) using `cargo upgrade` or similar tools.
- **Security Patches**: Apply updates immediately when tools like `cargo audit` or `cargo deny` report a vulnerability in the dependency tree.

✅ `cargo update -p serde --precise 1.0.197` to pin a single transitive dependency without touching everything.

❌ Confusing `cargo update` (lockfile only) with `cargo upgrade` (rewrites the manifest).

## 7. Dependency auditing (security & policy)

Rust has a strong, officially-supported auditing ecosystem. These tools are complementary, not exclusive — document them side by side.

| Tool | Maintainer | Purpose |
|---|---|---|
| **cargo-audit** | RustSec | Checks `Cargo.lock` against the **RustSec Advisory Database** for known CVEs and yanked/unmaintained crates (`RUSTSEC-YYYY-NNNN`). |
| **cargo-deny** | EmbarkStudios | Lint licenses, ban specific crates, check advisories, validate sources — all in one `deny.toml`. The most policy-rich option. |
| **cargo-outdated** | kbknapp | Lists dependencies that have newer versions than what your requirements allow. |
| **cargo-bloat** | | Measures how much binary size each dependency contributes. |

Canonical sources: RustSec advisory database (https://rustsec.org), https://github.com/RustSec/advisory-db.

✅ Run advisory checks in CI:
```bash
cargo install cargo-audit
cargo audit            # fails on known advisories
cargo deny check       # advisories + bans + licenses + sources
```

✅ An advisory is a RUSTSEC id like `RUSTSEC-2024-0344`; track it as an issue and `cargo update` to a fixed version.

❌ Ignoring unmaintained-crate warnings — `cargo audit` flags crates whose maintainers have marked them unmaintained (e.g., the old `chrono` < 0.4.20).

## 8. Vendoring, source replacement, and mirrors

### Vendoring (offline / air-gapped / reproducible)
`cargo vendor` downloads all dependencies into a `vendor/` directory and emits the `.cargo/config.toml` snippet to use them as a source replacement. Used for fully offline builds, supply-chain review, and reproducibility.

```bash
cargo vendor vendor > .cargo/config.toml
```

### Source replacement / mirrors (corporate proxying)
`.cargo/config.toml` can replace crates.io with a mirror (e.g., for air-gapped enterprises or Chinese mirrors):

```toml
[source.crates-io]
replace-with = "vendored-sources"

[source.vendored-sources]
directory = "vendor"

# Or a mirror:
[source.mirror]
registry = "https://mirrors.tuna.tsinghua.edu.cn/git/crates.io-index"
```

| Alternative | Use case |
|---|---|
| **Vendoring** | Air-gapped builds, supply-chain auditing, exact reproducibility. Trade-off: large repo, manual updates. |
| **Mirror source replacement** | Corporate proxying, geo latency. Trade-off: trust shifts to the mirror. |
| **Direct crates.io** | The default; simplest. |

## 9. MSRV (`rust-version` / Minimum Supported Rust Version)

The **MSRV** is the oldest Rust version a crate is guaranteed to compile on. Since Cargo 1.56, it is declared in the manifest:

```toml
[package]
rust-version = "1.80"
```

Behavior (and the debate):
- `cargo build` will **refuse** to build with a toolchain older than `rust-version` (since Cargo 1.65+ it errors rather than just warning).
- A separate concern is the **resolver**: Cargo 1.84+ can pick dependency versions compatible with a target MSRV via `resolver.incompatible-rust-versions = "fallback"` (or the `--update-precise`-style pinning). The ecosystem is still settling on whether the resolver should auto-downgrade deps for MSRV.
- Document your MSRV policy in the crate README (e.g., "we support the last N stable releases" or "we pin to a specific MSRV").

✅ Declare `rust-version` and document the policy in the README and in `## MSRV` section.

❌ Using a bleeding-edge API without bumping `rust-version` — your users get a confusing compile error.

## 10. Evaluating a new dependency (before adding it)

Community-recognized checklist:
1. **License** — must be compatible with your project (MIT/Apache-2.0 dual is the Rust norm; check for GPL/AGPL/unclear).
2. **Maintenance** — recent commits, open issues triaged, releases in the last ~year.
3. **Download volume** — crates.io download stats; a crate with millions of downloads is lower risk.
4. **Security history** — search the RustSec advisory DB; check for past `RUSTSEC-*` entries.
5. **Transitive cost** — `cargo tree` to inspect what else it pulls in; compile-time impact matters in Rust.
6. **Crates.io verification** — crates.io now has provenance features (publisher verification via GitHub, etc.).

```bash
cargo tree -i <crate>          # what depends on this?
cargo tree -d                  # duplicate versions (-d) — bloat indicator
cargo build --timings          # see compile-time cost per crate
```

✅ `cargo tree -d` regularly to catch duplicate-version bloat (two copies of `syn` or `tokio` is common and expensive).

❌ Adding a dependency for a 5-line function that the stdlib or an existing dep already covers — Rust culture values a lean dependency tree.

## 11. Transitive dependencies

- Cargo resolves a **single unified version** per package per build graph, favoring the newest requirement that satisfies all (resolver v2 unifies features per-target).
- Use `cargo tree` to inspect; `cargo tree -i <crate>` shows the inverse (who depends on it).
- Duplicate versions are allowed when requirements genuinely conflict, but each duplicate multiplies compile time and binary size.

## Summary table — sources

| Topic | Canonical source |
|---|---|
| Cargo manifest | https://doc.rust-lang.org/cargo/reference/manifest.html |
| SemVer / version requirements | https://doc.rust-lang.org/cargo/reference/specifying-dependencies.html |
| Cargo.lock guidance (2023) | https://blog.rust-lang.org/2023/08/29/committing-lockfiles/ |
| Features | https://doc.rust-lang.org/cargo/reference/features.html |
| Workspaces | https://doc.rust-lang.org/cargo/reference/workspaces.html |
| Source replacement / vendor | https://doc.rust-lang.org/cargo/reference/source-replacement.html |
| MSRV (`rust-version`) | https://doc.rust-lang.org/cargo/reference/manifest.html#the-rust-version-field |
| RustSec advisories | https://rustsec.org / https://github.com/RustSec/advisory-db |
| cargo-deny | https://github.com/EmbarkStudios/cargo-deny |
| cargo-edit (`cargo upgrade`) | https://github.com/killercup/cargo-edit |
