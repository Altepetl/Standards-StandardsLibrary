---
title: Rust - Scope and Purpose
status: draft
version: 0.0.1
created: 2026-07-06
updated: 2026-07-06
---

# Rust — Scope and Purpose

## What this standard covers

This document establishes the scope of the Rust programming standards cataloged in this repository. It defines **what "well-written Rust" means**, the language's positioning, and the core mental model that every other section in this catalog builds on. It is intentionally non-prescriptive about ecosystem choices where multiple recognized alternatives exist — those are documented side by side in their respective sections.

The standard covers the concerns shared by essentially all Rust codebases:

- **Naming conventions** (Section 2) — type vs. value vs. constant casing.
- **Formatting and source structure** (Section 3) — `rustfmt` and the Rust Style Guide.
- **Project layout** (Section 4) — the Cargo project model, workspaces, and module organization.
- **Language idioms** (Section 5) — ownership/borrowing/lifetimes, error handling, iterators, `unsafe`.
- **Documentation** (Section 6) — `rustdoc`, doc comments, doctests.
- **Error handling and logging** (Section 7) — `Result`/`Option`, `thiserror` vs. `anyhow`, `tracing`.
- **API design** (Section 8) — the Rust API Guidelines, conversions, `Send`/`Sync`, `serde`.
- **Security and privacy** (Section 9) — memory-safety guarantees, `unsafe` auditing, supply chain, secrets.

Sections outside this batch (dependencies, testing, CI/CD, concurrency, etc.) are cataloged separately under the same language directory.

## Governance and standard evolution

Rust does not have a formal ISO/ANSI specification. Instead, it is governed by a **team-based structure** (e.g., Language Team, Library Team, Cargo Team) operating under the umbrella of the **Rust Project**, with support from the **Rust Foundation**.

Technical decisions and language features evolve through a rigorous **RFC (Request for Comments)** process. Once accepted, they are implemented and eventually stabilized. The official definitions of "standard Rust" are found in:
- **The Rust Reference:** Describes the syntax and semantics of the language.
- **The RFCs:** The detailed historical record of why features exist and how they should be used.
- **The Rustonomicon:** The canonical guide for unsafe Rust.

## Applicability dimensions and lifecycle

Rust standards apply differently depending on the nature and phase of a project:

- **Lifecycle phases:**
  - **New code:** Should strictly adhere to the latest stable edition and maintain zero Clippy warnings.
  - **Maintenance:** Legacy code migrations to new editions are handled mechanically via `cargo fix --edition`. Gradual adoption of idiomatic patterns (e.g., moving from indexing to iterators) is preferred during refactoring.
- **Project types and layers:**
  - **Applications (binaries):** Focus on business logic, observability, and flexible error handling (often using `anyhow`).
  - **Libraries (crates):** Strict adherence to the Rust API Guidelines. Must prioritize predictable performance, zero-cost abstractions, non-panicking APIs, and structured error types (e.g., using `thiserror`).
  - **FFI boundaries:** Code interfacing with C/C++ requires specific data layouts (`#[repr(C)]`) and careful `unsafe` auditing.

### Community-specific interpretations (`#![no_std]`)

Rust is deployed across wildly different environments, changing which parts of the ecosystem are accessible:
- **Standard (`std`):** The default environment (OS present) supporting file I/O, threading, and dynamic allocation.
- **Embedded and bare-metal (`#![no_std]`):** For microcontrollers, OS kernels, or certain WebAssembly targets, the standard library is disabled. Code relies solely on the `core` library (which makes no assumptions about the environment) and optionally `alloc`. Standards in this domain emphasize static allocation, deterministic memory usage, and hardware-specific safety constraints.

## Language positioning

Rust is a **systems programming language** that provides memory safety and thread safety **without a garbage collector**. Its three load-bearing value propositions, repeated across the official documentation and the Rust programming book, are:

1. **Memory safety without GC.** Ownership, borrowing, and lifetimes are checked at compile time by the **borrow checker**. Use-after-free, double-free, dangling pointers, and data races are prevented for *safe* Rust — statically, with no runtime cost.
2. **Fearless concurrency.** Because the same ownership rules govern data access across threads, the compiler rejects data races at compile time (`Send` and `Sync` are the marker traits that encode thread-safety, covered in Section 8 and Section 16). Threads can share data only through the proven-safe channels (`Arc`, `Mutex`, channels, etc.).
3. **Zero-cost abstractions.** Per B. Stroustrup's definition Rust adopts: "What you don't use, you don't pay for. What you do use, you couldn't write better by hand." Higher-order functions, iterators, generics (with monomorphization), and traits are designed so that idiomatic high-level code compiles down to hand-efficient machine code.

Rust targets the same problem space as C and C++ (operating systems, embedded, game engines, browsers, databases, networking, CLI tools) while also being widely used for web services, WebAssembly, and CLI applications where developer productivity matters.

### The core model: ownership, borrowing, lifetimes

Everything else in this standard rests on three intertwined concepts:

- **Ownership.** Every value has exactly one owner; when the owner goes out of scope the value is dropped (its destructor runs).
- **Borrowing.** References (`&T` shared, `&mut T` exclusive) let code use a value without taking ownership. The aliasing rule — *many shared references XOR one mutable reference* — is what eliminates data races.
- **Lifetimes.** Compile-time tags (`'a`) that track how long a reference is valid, so the compiler can prove no reference outlives its referent.

This model is enforced by the **borrow checker**, and it is the single biggest adjustment for programmers coming from GC'd languages. Section 5 catalogues the idioms that work *with* the checker rather than against it.

## The role of editions

Per the Rust Book (Appendix E — Editions, `doc.rust-lang.org/book/appendix-05-editions.html`), an **edition** is a backwards-compatible opt-in bundle of language changes. Editions matter because Rust promises that code written under one edition keeps compiling forever, even as the language evolves.

- **Editions only affect parsing.** A crate on edition 2015 and a dependency on edition 2024 interoperate seamlessly; editions are *not* ABI or version breaks.
- An edition is declared in `Cargo.toml`:

  ```toml
  [package]
  edition = "2024"
  ```

- A Rust **edition** (e.g. 2021) is distinct from a Rust **compiler version** (e.g. 1.95.0). The edition gates language features and reserved syntax; the compiler version ships new stable features regardless of edition.

The four editions to date:

| Edition | Highlights |
|---|---|
| **2015** | The baseline edition; establishes Rust's stability promises. |
| **2018** | Async/await groundwork, the new module system (`use crate::`, no more `extern crate` in most cases), `dyn Trait`, raw identifiers (`r#`), non-lexical lifetimes (NLL, also backported). |
| **2021** | Disjoint closure captures, `IntoIterator` for arrays by value, or-patterns in macro rules, prelude additions (`TryInto`, `TryFrom`, `FromIterator`), panic macros consistency. |
| **2024** | The largest edition yet: RPIT lifetime capture rules, `gen` blocks, reserved syntax for future features, tighter unsafe obligations (e.g. `unsafe extern` blocks), `cargo`-level changes. |

**Recommendation.** New code should declare the **latest stable edition** available to its minimum supported Rust version (MSRV). When a project pins an older edition it is usually to keep the MSRV low or to avoid a deliberate migration. Edition migrations are mechanical and can be run with `cargo fix --edition`.

## What "well-written Rust" means in the community

The community converges on a recognizable style, captured in the canonical references cited throughout this catalog:

- **It compiles under `cargo clippy` with few or no warnings.** Clippy (`rust-lang.github.io/rust-clippy/master/`) is the de facto best-practices linter; its lint groups (`correctness`, `suspicious`, `style`, `complexity`, `perf`, `pedantic`) encode community norms.
- **It is formatted by `rustfmt`.** The Rust Style Guide (`rust-lang.github.io/style-guide/`) and its `rustfmt` implementation are the de facto formatting standard; formatted code is non-controversial in review.
- **It follows the Rust API Guidelines** (`rust-lang.github.io/api-guidelines/`) for anything published as a library.
- **It treats `Result`/`Option` seriously** — no `.unwrap()` / `.expect()` / `.panic!()` in production paths except where a panic is genuinely the correct contract (see Section 7).
- **It prefers iterators and combinators over index-based loops** where the iterator form is at least as clear (zero-cost abstraction in action).
- **It works *with* the borrow checker**, not against it — borrowing over cloning, lifetime elision over explicit annotation, owned types where ownership is actually needed.
- **It documents public items** with `rustdoc` comments, including the `# Panics`, `# Errors`, and `# Safety` sections where applicable (Section 6).

Two persistent tensions in the community, documented rather than adjudicated here:

- **`unwrap`/`expect` discipline.** Some teams forbid them entirely outside tests; others permit them with a documented invariant ("this `Option` is `Some` because the config was validated upstream"). Both are recognized positions.
- **`clippy::pedantic` vs. default lints.** Some projects enable pedantic/all lints for maximum strictness; others find them noisy and stick to the default + `clippy::style`. Both are reasonable; the standard is only "have a deliberate `clippy` configuration."

## Non-goals

This standard does **not**:

- Pick a winner between recognized alternatives (e.g. `anyhow` vs. `thiserror`, `log` vs. `tracing`, `tokio` vs. `async-std`). Those are presented with trade-offs in their sections.
- Mandate a specific crate for jobs where the ecosystem is split. Selection happens in the downstream **StandardBuilder** project.
- Cover non-Rust tooling (e.g. Docker, CI runners) beyond Rust-specific concerns.
- Replace the official Rust documentation. It references and condenses it; when in doubt, the upstream source wins.

## References

- The Rust Programming Language — `doc.rust-lang.org/book/` (especially Appendix E: Editions).
- The Rust Reference — `doc.rust-lang.org/reference/`.
- The Rust API Guidelines — `rust-lang.github.io/api-guidelines/`.
- The Rust Style Guide — `rust-lang.github.io/style-guide/`.
- Clippy lint list — `rust-lang.github.io/rust-clippy/master/`.
- The Cargo Book — `doc.rust-lang.org/cargo/`.
- RFC 430 (naming) and edition RFCs — `rust-lang.github.io/rfcs/`.
