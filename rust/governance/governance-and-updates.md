---
title: Rust - Governance and Standard Updates
status: draft
version: 0.0.1
created: 2026-07-06
updated: 2026-07-06
---

# Rust — Governance and Standard Updates

> This section documents how **the Rust language itself** evolves and how **a project's internal Rust standard** should track those changes. It also defines a review process for *this very document* so it does not rot. Boundary: this is governance/evolution; CI/CD mechanics are in `rust/cicd/` (planned).

## 1. The Rust Foundation and the teams

Rust is governed by a layered community model, not a single company.

### The Rust Foundation
- Founded **2021-02** as a non-profit to own the trademark, trademarks, infrastructure (crates.io, CI), and provide legal/financial backing.
- It does **not** set language direction; the technical teams do. https://foundation.rust-lang.org.

### The teams (the technical governance)
Rust is organized into **teams**, each with a charter and decision-making authority. Key teams:
| Team | Area |
|---|---|
| **Compiler team** | `rustc` architecture, diagnostics, performance. |
| **Lang team** | Language design, editions, feature stabilization. |
| **Library team** | The standard library API. |
| **Cargo team** | Cargo and the registry/resolver. |
| **rustdoc team** | The documentation tool. |
| **Infrastructure team** | CI, crates.io, bots. |
| **Moderation team** | Code of conduct enforcement. |
| **Release team** | The 6-week release train and stable/beta/nightly. |
| **Clippy / rustfmt / IDEs (rust-analyzer) teams** | Tooling. |
| **Security Response WG** | Vulnerability response and the RustSec advisory process. |

Teams make decisions by consensus (often via RFCs); leadership roles rotate. List: https://www.rust-lang.org/governance.

## 2. The RFC process

Significant changes go through **RFCs (Requests for Comments)** in the `rust-lang/rfcs` repository: https://github.com/rust-lang/rfcs.

- An RFC is a Markdown document describing motivation, detailed design, drawbacks, and alternatives.
- It is discussed in GitHub, merged into the RFCs repo (not necessarily implemented), and tracked.
- Implementation is tracked in a **tracking issue** and a **feature gate** on nightly.

This is Rust's equivalent of PEPs (Python), JEPs (Java), or TC39 proposals. Conventional Commits etc. are not governance; this is.

## 3. Editions — evolving without breaking the ecosystem

Rust uses **editions** to introduce opt-in, potentially-breaking changes without fragmenting the ecosystem. A crate declares its edition in `Cargo.toml`; crates of different editions can interoperate in the same build.

| Edition | Shipped with | Key changes |
|---|---|---|
| **2015** | Rust 1.0 | The original. |
| **2018** | Rust 1.31 | Module system overhaul, `dyn Trait`, non-lexical lifetimes (later), `async`/`await` groundwork. |
| **2021** | Rust 1.56 | Prelude additions (`IntoIterator for arrays`), disjoint closure captures, panic macros consistency. |
| **2024** | Rust **1.85** (2025-02-20) | `gen`/`Gen` reserved, `unsafe` on `extern` blocks, edition defaulting to new resolver behavior, stricter object-safety, "tail expression" temporaries. |

- Migrations are **automated** via `cargo fix --edition`.
- An edition bump is per-crate and **does not break downstream consumers** (interoperability is preserved at the FFI/module level).

Canonical source: https://doc.rust-lang.org/edition-guide/.

✅ Migrate with `cargo fix --edition` and bump `edition` in `Cargo.toml`; verify your MSRV.

❌ Fearing an edition bump as a "Python 2→3" event — Rust editions are explicitly designed to avoid that pain.

## 4. The 6-week release train

Rust ships on a **6-week cadence**, regardless of whether your feature made it. Three channels:

| Channel | Cadence | Use |
|---|---|---|
| **Nightly** | Every night | Bleeding-edge; required for unstable features (`-Z`), some rustfmt/clippy options, sanitizers. |
| **Beta** | Promoted from nightly every 6 weeks | Stabilization; bug fixes only. |
| **Stable** | Promoted from beta every 6 weeks | Production; the channel most projects use. |

New stable every 6 weeks; only the latest stable receives patches (no formal LTS — see MSRV instead). Track at https://releases.rs and the official release notes: https://blog.rust-lang.org/release-notes.

✅ Pin your project's toolchain with `rust-toolchain.toml` and review the release notes every 6 weeks.

❌ Pinning a specific nightly and never updating — nightlies drift; libraries drop support.

## 5. Stabilization and breaking changes

Rust's stability promise: **code that compiles on stable keeps compiling** (with narrow, well-communicated exceptions). Breaking changes are:
- Gates behind a new **edition** (opt-in).
- Lint-only (a new `future_incompatible` warning that later becomes an error).
- Documented in a "Breaking Changes" section of the release notes, almost always with a documented migration.

So a project should: subscribe to release notes and run `cargo fix` after upgrades.

## 6. Tracking and adopting new Rust versions (project-level)

A conventional project-level process:

1. **Toolchain pinning** — `rust-toolchain.toml` pins `channel = "stable"` (or a specific `1.x.0`).
2. **Cadence review** — every 6 weeks (on each stable release), review the release notes; run `rustup update` and `cargo test`.
3. **MSRV policy** — decide and document (e.g., "supports the last 5 stable releases" or a pinned version); declare `rust-version` in `Cargo.toml`.
4. **Edition migration** — when adopting a new edition, run `cargo fix --edition`, review changes, bump `edition`.

## 7. MSRV policy

The **Minimum Supported Rust Version** is project policy, not a language rule. Recognized options:

| Policy | Trade-off |
|---|---|
| **Latest stable only** | Simplest; smallest support burden; locks out users on older toolchains. |
| **Last N stable releases** | Common; balances reach and maintenance. |
| **Fixed to a long-stable version** (e.g., "1.74") | Maximum reach for libraries; requires CI against that version and CI to enforce feature-gating of newer APIs. |

Declare it in `Cargo.toml` (`rust-version`) and the README's `## MSRV` section. Cargo 1.84+ can also help the resolver respect MSRV (`resolver.incompatible-rust-versions`).

## 8. Keeping the standard up to date (signals to subscribe to)

To keep *this* standard current, subscribe to:
- **This Week in Rust** — https://this-week-in-rust.org (weekly digest).
- **Rust Blog** — https://blog.rust-lang.org (release notes, edition announcements, governance).
- **Inside Rust Blog** — https://blog.rust-lang.org/inside-rust (team-level updates).
- **rust-lang/rfcs** — for significant upcoming changes.
- **The Cargo Book / rustdoc book / Reference / Nomicon** — for canonical doc updates.
- **RustSec** — https://rustsec.org/advisories/ for security advisories affecting dependencies.

## 9. Governance of THIS document

This StandardLibrary document is **not** governed by the Rust project; it is a catalog maintained inside the StandardLibrary repository. Per the repository's `/standard_research` workflow:

| Aspect | Convention |
|---|---|
| **Owner** | The repository maintainers (architect / lead). Optionally a "Rust standards working group". |
| **Cadence** | Re-review at least **every 6 months** (Rust ships 8-9 stable releases in that window), and **immediately after** a new edition or a major ecosystem shift (e.g., a new dominant async runtime, a security incident). |
| **Versioning** | Bump the front-matter `version` (semver: patch for typo/clarification, minor for additive content, major for removed/changed guidance) and refresh `updated` on every edit. |
| **Change process** | Open a PR; reference the authoritative source (Rust Blog, RFC, Cargo Book). Document alternatives side by side — do not pick a winner (this is a catalog). |
| **Changelog** | Maintain a `## Changelog` section or rely on git history; record the "why" for non-trivial changes. |
| **Exceptions** | If a project must deviate from a documented convention, use a "Rule Exception Request" (rule reference, justification, mitigation, approver, expiry) recorded in the project, not here. |

### Review checklist (run every review cycle)

- [ ] Confirm the latest stable Rust version and edition; update any version-specific examples.
- [ ] Verify every linked canonical source still resolves (Cargo Book, RFCs, crates.io, RustSec).
- [ ] Re-check each "side by side alternatives" section — has the ecosystem converged? Document the convergence (still without picking a winner unless one is now clearly deprecated).
- [ ] Refresh tool versions/commands (e.g., `rustfmt`/`clippy` options that moved from nightly to stable).
- [ ] Scan **This Week in Rust** and **RustSec** for material changes since `updated`.
- [ ] Bump `version` and `updated` in the front matter.

### Deprecation lifecycle for a documented rule

When a rule in this document becomes obsolete (e.g., a tool is superseded), apply:

```
Phase 1 – Warn:        mark the rule "deprecated" inline; point to the replacement; keep it.
Phase 2 – Soft remove: remove the rule from the main body; note the removal in the Changelog.
Phase 3 – Hard remove: drop all mentions; the rule is historical.
```

## Sources

| Topic | Source |
|---|---|
| Rust Foundation | https://foundation.rust-lang.org |
| Teams / governance | https://www.rust-lang.org/governance |
| RFCs repo | https://github.com/rust-lang/rfcs |
| Edition Guide | https://doc.rust-lang.org/edition-guide/ |
| Rust 1.85 / Edition 2024 announcement | https://blog.rust-lang.org/2025/02/20/Rust-1.85.0/ |
| Release cycle | https://doc.rust-lang.org/book/ch01-03-how-rust-is-made-and-nightly-rust.html |
| Release notes | https://blog.rust-lang.org/release-notes |
| This Week in Rust | https://this-week-in-rust.org |
| Inside Rust | https://blog.rust-lang.org/inside-rust |
| RustSec advisories | https://rustsec.org/advisories/ |
| `rust-version` (MSRV) | https://doc.rust-lang.org/cargo/reference/manifest.html#the-rust-version-field |
