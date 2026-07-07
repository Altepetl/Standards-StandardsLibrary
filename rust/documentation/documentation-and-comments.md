---
title: Rust - Documentation and Comments
status: draft
version: 0.0.1
created: 2026-07-06
updated: 2026-07-06
---

# Rust — Documentation and Comments

Rust ships **`rustdoc`**, a documentation tool that turns specially-marked comments into rendered HTML documentation, and treats the code blocks inside those comments as **executable tests** (doctests). The conventions here come from the **rustdoc book** (`doc.rust-lang.org/rustdoc/`) and the **Rust API Guidelines — Documentation** chapter (`rust-lang.github.io/api-guidelines/documentation.html`).

## Two kinds of comments

| Syntax | Kind | Purpose |
|---|---|---|
| `//` | Line comment | Internal note, ignored by rustdoc. |
| `/* … */` | Block comment | Internal note, ignored by rustdoc. |
| `///` | Outer doc comment | Documents the **item that follows** (a `fn`, `struct`, `enum`, `const`, module, etc.). |
| `//!` | Inner doc comment | Documents the **enclosing item** — typically the crate root or a module. |

```rust
//! Crate-level documentation for `my_parser`.
//!
//! `my_parser` is a small recursive-descent parser.

/// Parses a single expression from `input`.
///
/// # Errors
/// Returns `Err` if `input` is empty or malformed.
pub fn parse(input: &str) -> Result<Expr, ParseError> { /* ... */ unimplemented!() }
```

- Use `//!` at the top of `src/lib.rs` (the crate root) and at the top of any module whose purpose is not obvious.
- Use `///` above every public item; this is enforced by `#![deny(missing_docs)]` in libraries that want full documentation.

## Markdown in rustdoc

Doc comments are **CommonMark Markdown** with rustdoc extensions. Supported features include:

- Headings (`#`, `##`).
- Code blocks (`` ``` ``) — executed as doctests by default.
- Tables, lists, links.
- **Intra-doc links** — `[`Parser::parse`]` auto-resolves to the item's path; broken links fail the build under `broken_intra_doc_links`.
- `#![feature(doc_cfg)]` / `#[doc(cfg(...))]` to annotate platform/feature availability (nightly; `doc_auto_cfg` makes this automatic).

✅ **Do** — use intra-doc links:

```rust
/// Splits this string at the first space.
///
/// See also [`split_words`].
pub fn first_word(s: &str) -> &str { /* ... */ "" }

/// Splits a string into whitespace-delimited words.
pub fn split_words(s: &str) -> Vec<&str> { s.split_whitespace().collect() }
```

## The standard doc sections

Per the Rust API Guidelines (C-FAILURE and the documentation chapter), public functions/methods should document **what can go wrong** using the conventional heading sections:

| Section | When to include |
|---|---|
| `# Examples` | Almost always; the canonical place for a runnable doctest. |
| `# Parameters` (or `# Arguments`) | When the meaning of an argument is not obvious from its name and type. |
| `# Returns` | When the return value's meaning is not obvious. |
| `# Panics` | **Always**, if the function can `panic!`. Document the precondition that prevents it. |
| `# Errors` | **Always**, if the function returns `Result`. Enumerate the error cases. |
| `# Safety` | **Mandatory** for any `unsafe fn` or function with `unsafe` obligations. Explain the contract the caller must uphold. |
| `# Complexity` | Optional; for algorithms with non-obvious time/memory behavior. |

✅ **Do** — a fully documented fallible function:

```rust
use std::num::ParseIntError;

/// Parses `input` as a port number.
///
/// # Examples
///
/// ```
/// use my_crate::parse_port;
/// assert_eq!(parse_port("8080").unwrap(), 8080);
/// ```
///
/// # Errors
///
/// Returns [`ParseIntError`] if `input` is not a valid `u16`, or if it is empty.
///
/// [`ParseIntError`]: std::num::ParseIntError
pub fn parse_port(input: &str) -> Result<u16, ParseIntError> {
    input.parse::<u16>()
}
```

✅ **Do** — a `# Safety` section on an `unsafe fn`:

```rust
/// Reads 4 bytes from `ptr` as a little-endian `u32`.
///
/// # Safety
///
/// The caller must uphold:
/// - `ptr` is non-null, aligned to 4 bytes, and valid for reads of 4 bytes.
/// - The memory is not concurrently mutated while this call reads it.
pub unsafe fn read_u32_le(ptr: *const u8) -> u32 {
    /* ... */ unimplemented!()
}
```

❌ **Don't** — an `unsafe fn` with no `# Safety`:

```rust
pub unsafe fn read_u32_le(ptr: *const u8) -> u32 { /* ❌ no contract documented */ unimplemented!() }
```

## Doctests — executable examples

Every ```` ``` ````-fenced code block in a `///` comment is compiled and run by `cargo test --doc`. A code block that fails to compile or that panics fails the test suite. This keeps documentation honest.

```rust
/// Adds two numbers.
///
/// # Examples
///
/// ```
/// use my_crate::add;
/// assert_eq!(add(2, 3), 5);
/// ```
pub fn add(a: i32, b: i32) -> i32 { a + b }
```

### Doctest modifiers

- ``` ```no_run ``` — compile but don't execute (e.g. opens a socket).
- ``` ```ignore ``` — compile but don't run; the file does not need to exist at compile time.
- ``` ```text ``` — render as plain text, not a doctest.
- ``` ```compile_fail ``` — assert the example **does not** compile (negative documentation).
- `# hidden lines` — lines starting with `#` are hidden from rendered docs but still compiled. Useful for `use` statements you don't want to show:

```rust
/// ```
/// # use my_crate::Parser;     // hidden from rendered docs, but compiled
/// let p = Parser::new("(+ 1 2)");
/// assert!(p.parse().is_ok());
/// ```
```

### Doctest caveats

- Doctests link against the crate as an **external** user, so they can only see the public API. Items that are `pub(crate)` or private are not accessible.
- Doctests share a single `main` per block; helpers across blocks are awkward — keep each block self-contained.
- Running doctests on every change is slow for large libraries; `cargo test --doc` can be moved to a slower CI job, but it must run somewhere.

## `//` and `// !` — when to use plain comments

- Use `//` for **why** explanations that don't belong in user-facing docs (implementation notes, links to issues, non-obvious algorithm choices).
- Avoid comments that restate the code (`let i = 0; // set i to zero`).
- Reserve `// SAFETY:` comments for `unsafe` blocks (see Section 5); rustc and Clippy expect them.

```rust
// `HashMap` iteration order is non-deterministic; sort before snapshotting
// so the golden test file is reproducible.
let mut keys: Vec<_> = map.keys().collect();
keys.sort_unstable();
```

## Marker Comments

Rust projects commonly use specific marker comments to signal intent for future developers:

- `// TODO: ` — An action item, missing feature, or technical debt that should be addressed later.
- `// FIXME: ` — Broken code or a known bug that needs to be fixed.
- `// HACK: ` — A workaround or suboptimal implementation that should be refactored eventually.
- `// NOTE: ` — Additional information or context about a specific implementation detail.

**Convention:** When using `TODO` or `FIXME`, it is best practice to include a reference to an issue tracker or an author identifier, providing clear accountability and context. For example: `// TODO(#123): Handle edge case for zero`. Some projects enforce this via linters (like `clippy::todo` or custom grep checks in CI) to ensure unresolved issues aren't accidentally forgotten.

## Self-Documenting Code Philosophy

The Rust ecosystem strongly favors **self-documenting code** over extensive inline comments. This philosophy manifests primarily through Rust's expressive type system and ownership rules:

- **Expressive Types:** Instead of writing comments to explain what a variable is, you should rely on types. Use the Newtype pattern (e.g., `struct UserId(u64)`) to enforce domain logic at compile time, eliminating the need to comment that a `u64` represents a user ID.
- **Encoding State:** If a state is impossible, it should be made unrepresentable in the type system rather than guarded with a runtime check and a comment.
- **Error Handling:** If a function might fail, this shouldn't merely be documented in comments—it should be explicitly codified in the signature by returning a `Result`. 
- **Focus on the "Why":** Comments should focus exclusively on the "why" (business rules, context, trade-offs) instead of the "what" or "how" (which the code and types should already clearly convey).

## README Conventions

In the Rust ecosystem, the `README.md` file at the root of a package serves as the primary entry point for users visiting the repository (e.g., on GitHub) or crates.io.

- **crates.io Integration:** The `Cargo.toml` file usually points to the `README.md` (via the `readme = "README.md"` key), and crates.io will render it automatically on the package's page.
- **DRY Documentation:** Since `lib.rs` uses `//!` for crate-level documentation, it is common to avoid duplicating this content in the `README.md`. A prevalent pattern is to write the documentation in `README.md` and include it in `lib.rs` using `#![doc = include_str!("../README.md")]`. Alternatively, tools like `cargo-readme` can generate the `README.md` directly from the `lib.rs` doc comments.
- **Content:** A canonical README for a Rust crate should include:
  - CI status and crates.io version badges.
  - A brief description of the crate's purpose.
  - A simple, copy-pasteable example of use.
  - Instructions for installation (e.g., `cargo add my_crate`) and contributing.

## `cargo doc`

- `cargo doc` — builds HTML docs for the crate into `target/doc/`.
- `cargo doc --open` — builds and opens in a browser.
- `cargo doc --no-deps` — document only this crate, not dependencies.
- `cargo doc --document-private-items` — include private items (useful for internal review).

## rustdoc lints

| Lint | Effect |
|---|---|
| `missing_docs` | Warn on undocumented public items. |
| `missing_docs_in_private_items` | Same, but for private items too. |
| `broken_intra_doc_links` | Error when an intra-doc link target is not found. |
| `private_intra_doc_links` | Allow intra-doc links to private items. |
| `rustdoc::bare_urls` | Warn on URLs that aren't clickable links. |

A library that wants strict docs typically adds to its crate root:

```rust
#![deny(missing_docs)]
#![warn(rustdoc::broken_intra_doc_links)]
```

## Summary checklist

- [ ] `//!` at the top of `lib.rs` and any module needing a header.
- [ ] `///` on every public item.
- [ ] `# Examples` with a runnable doctest on public functions.
- [ ] `# Panics` / `# Errors` / `# Safety` whenever applicable.
- [ ] `missing_docs` and `broken_intra_doc_links` enabled.
- [ ] `cargo test --doc` runs in CI.

## References

- The rustdoc book — `doc.rust-lang.org/rustdoc/`.
- The rustdoc book — Documentation tests — `doc.rust-lang.org/rustdoc/documentation-tests.html`.
- Rust API Guidelines — Documentation — `rust-lang.github.io/api-guidelines/documentation.html`.
- How to write documentation — The Rust Book — `doc.rust-lang.org/book/ch14-02-publishing-to-crates-io.html`.
