---
title: Rust - Testing and Quality Validation
status: draft
version: 0.0.1
created: 2026-07-06
updated: 2026-07-06
---

# Rust — Testing and Quality Validation

> Rust ships a **built-in test framework** in the standard library (`#[test]`, `assert!`, etc.) run by `cargo test`. The ecosystem adds property testing, mocking, faster runners, coverage, mutation testing, and fuzzing. This document covers the built-in framework first, then the recognized third-party crates side by side. Canonical source: The Rust Book ch. 11 — https://doc.rust-lang.org/book/ch11-00-testing.html.

## 1. The built-in test framework

Every `cargo test` run executes three kinds of tests: unit tests (in `src/`), integration tests (in `tests/`), and doctests (in `///` comments). No external test runner is required.

### Test attributes and macros

| Element | Purpose |
|---|---|
| `#[test]` | Marks a function as a test. |
| `#[cfg(test)]` | Compiles the module only during `cargo test` (zero cost in release). |
| `#[should_panic(expected = "...")]` | Asserts the test panics (optionally matching a substring). |
| `#[ignore]` | Skips the test unless run with `--ignored`. |
| `assert!(cond)` | Boolean assertion. |
| `assert_eq!(a, b)` | Equality (requires `PartialEq` + `Debug`). |
| `assert_ne!(a, b)` | Inequality. |
| `#[derive(Debug, PartialEq)]` | Common companion so assertions can print failures. |

## 2. Unit tests — colocated `mod tests`

The idiomatic Rust convention is a **`#[cfg(test)] mod tests` block at the bottom of each source file**, testing private items in the same module.

✅ Colocated unit tests (canonical pattern):
```rust
pub fn add(a: i32, b: i32) -> i32 {
    a + b
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn add_positive() {
        assert_eq!(add(2, 2), 4);
    }

    #[test]
    #[should_panic(expected = "overflow")]
    fn add_panics_on_overflow() {
        // ... something that panics with "overflow"
    }

    #[test]
    #[ignore = "slow: only run with --ignored"]
    fn long_running() { /* ... */ }
}
```

The `use super::*;` brings the parent module's items into scope; because the module is `#[cfg(test)]`, it never ships in the released binary.

❌ Putting unit tests in `tests/` — that directory is reserved for **integration** tests, which can only access the *public* API, so private-function tests will not compile there.

## 3. Integration tests — `tests/`

Files in the top-level `tests/` directory are compiled as separate crates that depend on your crate as an external user would. Each file is its own test target.

✅ `tests/api.rs` exercises the public API:
```rust
use my_crate::add;

#[test]
fn public_api_works() {
    assert_eq!(add(2, 2), 4);
}
```

For shared helpers across integration tests, use a module: `tests/common/mod.rs` (Rust treats top-level files in `tests/` as separate crates, but a `mod common;` submodule is shared, not a separate target).

## 4. Doctests — `cargo test --doc`

Rust's `///` documentation comments are compiled and run as tests by default, ensuring examples in docs never rot.

✅ Runnable doctest:
```rust
/// Adds two integers.
///
/// # Examples
///
/// ```
/// use my_crate::add;
/// assert_eq!(add(2, 2), 4);
/// ```
pub fn add(a: i32, b: i32) -> i32 { a + b }
```

To skip a doctest that is illustrative only:
```rust
/// ```no_run
/// // not executed, only compiled
/// ```
```

Other markers: `ignore`, `should_panic`, `compile_fail`, `edition2018`/`edition2021`/`edition2024`.

❌ Doctests that depend on private items — doctests run against the crate's public API only.

## 5. Test structure — AAA / Given-When-Then

Rust has no imposed structure; the community generally follows **Arrange-Act-Assert** within each `#[test]` fn. Naming is conventionally `snake_case` and behavior-descriptive (e.g., `fn returns_zero_for_empty_input()`). There is no universal `should_..._when_...` rule.

✅ Descriptive test names:
```rust
#[test]
fn empty_stack_pop_returns_none() { /* ... */ }
```

❌ `#[test] fn test1()` — meaningless name.

## 6. Property-based testing — `proptest`

The dominant crate for **property testing** is **proptest** (inspired by QuickCheck). It generates many random inputs and checks invariants, shrinking failures to a minimal reproducer.

✅ Property test:
```rust
use proptest::prelude::*;

proptest! {
    #[test]
    fn add_doesnt_overflow(a in -1000i32..1000, b in -1000i32..1000) {
        let _ = a + b; // invariant: never panics in this range
    }
}
```

Alternative recognized crate: **quicktest**/**quickcheck** (older, less feature-rich). Document side by side: proptest is more popular for realistic generation strategies; quickcheck is minimal.

## 7. Parametrized testing — `rstest`

For table-driven / parametrized tests, **rstest** is the most widely adopted crate.

✅ Parametrized test:
```rust
use rstest::rstest;

#[rstest]
#[case(2, 2, 4)]
#[case(-1, 1, 0)]
#[case(0, 0, 0)]
fn add_cases(#[case] a: i32, #[case] b: i32, #[case] expected: i32) {
    assert_eq!(my_crate::add(a, b), expected);
}
```

Alternative: write the loop yourself with `Vec` of inputs and a plain `#[test]`. Both are accepted.

## 8. Mocking

| Crate | Use case |
|---|---|
| **mockall** | Trait/struct mocking for unit tests; generates mock implementations via `#[automock]` and `mock!`. The dominant mocking crate in pure-Rust code. |
| **mockito** | HTTP mock server for integration tests against external services. |
| **wiremock** | Async-friendly HTTP mocking (popular with `tokio` projects). |
| **faux** | Mocking struct methods without trait gymnastics. |

✅ Mock a trait with mockall:
```rust
use mockall::*;
#[automock]
pub trait Repository {
    fn find(&self, id: u32) -> Option<String>;
}

#[test]
fn uses_mocked_repo() {
    let mut mock = MockRepository::new();
    mock.expect_find().returning(|_| Some("data".to_string()));
    assert_eq!(mock.find(1), Some("data".to_string()));
}
```

❌ Mocking what you don't own: the London-school rule (don't mock third-party types) applies in Rust too — wrap external clients in your own trait and mock that.

## 9. Test runners

| Tool | Role |
|---|---|
| **`cargo test`** | Built-in; the default. Uses libtest. |
| **`cargo nextest`** | A drop-in, much faster test runner from Meta. Runs tests in parallel by default with better isolation, retry, and JUnit output. Widely adopted in CI. |

✅ `cargo nextest run` in CI for speed and per-test isolation; `cargo test` locally is fine.

## 10. Coverage — `cargo-tarpaulin` vs `cargo-llvm-cov`

The community uses two recognized tools, documented side by side:

| Tool | How | Notes |
|---|---|---|
| **cargo-tarpaulin** | Linux-only, DWARF-based | Simplest setup; no extra toolchain. Linux-only is the main limitation. |
| **cargo-llvm-cov** | Uses LLVM's source-based coverage | Cross-platform; requires `RUSTFLAGS="-Cinstrument-coverage"`. Generally more accurate (branch coverage). The recommended default for portability. |

✅ Run with `cargo llvm-cov --html` to produce an HTML report; integrate with codecov/coveralls.

Community threshold: there is no official Rust minimum, but many projects target **≥ 80% line coverage** for the core library, with critical paths higher. Document your project's threshold; do not assume a universal number.

## 11. Mutation testing — `cargo-mutants`

**cargo-mutants** mutates your source (changes `+` to `-`, flips `>` to `<`, etc.) and checks which mutants survive the test suite. Surviving mutants indicate gaps in test quality.

```bash
cargo install cargo-mutants
cargo mutants
```

✅ Use cargo-mutants periodically (not necessarily in every CI run) to find under-tested branches.

## 12. Fuzzing — `cargo-fuzz` / libFuzzer and `afl.rs`

For security-critical parsers and decoders, Rust supports coverage-guided fuzzing.

| Tool | Engine | Notes |
|---|---|---|
| **cargo-fuzz** | libFuzzer (LLVM) | Most popular; uses in-process fuzzing with `libfuzzer-sys`. Requires nightly for `-Zsanitizer`. |
| **afl.rs** | AFL (American Fuzzy Lop) | Out-of-process; works on stable in more cases. |

✅ Fuzz harness:
```rust
// fuzz/fuzz_targets/parse.rs
#![no_main]
use libfuzzer_sys::fuzz_target;

fuzz_target!(|data: &[u8]| {
    let _ = my_parser::parse(data); // must not panic/abort on any input
});
```

Canonical reference: the Rust Fuzz Book — https://rust-fuzz.github.io/book/.

## 13. Test independence

Rust's borrow checker and the lack of global mutable state by default make test independence easier than in many languages, but it is still a convention:
- Do not share mutable `static`s across tests without synchronization.
- Use fresh fixtures per test; rstest fixtures (`#[fixture]`) and `Default` instances help.
- Integration tests that touch shared external resources (DB, network) must clean up or use test containers.

✅ Each test owns its data:
```rust
#[test]
fn processes_unique_input() {
    let mut buf = Vec::new();   // fresh per test
    // ...
}
```

❌ A `static mut COUNTER: i32 = 0;` shared and mutated across tests — racy and `unsafe`.

## 14. Static analysis (cross-reference)

Quality validation also includes **clippy** (lints) and **rustfmt** — see `rust/automation/linters-and-formatters.md`. `cargo clippy -- -D warnings` is a near-universal CI gate.

## 15. Putting it together — a conventional `cargo test` workflow

```bash
cargo fmt -- --check        # formatting gate
cargo clippy --all-targets -- -D warnings   # lint gate
cargo test                  # unit + integration + doctests
cargo test --doc            # doctests only
cargo llvm-cov              # coverage (optional)
cargo nextest run           # faster runner (optional)
```

## Sources

| Topic | Source |
|---|---|
| Built-in testing | https://doc.rust-lang.org/book/ch11-00-testing.html |
| rustdoc doctests | https://doc.rust-lang.org/rustdoc/write-documentation/doctests.html |
| proptest | https://github.com/proptest-rs/proptest |
| rstest | https://github.com/la10736/rstest |
| mockall | https://github.com/asomers/mockall |
| mockito | https://github.com/lipanski/mockito |
| cargo-nextest | https://nexte.st |
| cargo-tarpaulin | https://github.com/xd009642/tarpaulin |
| cargo-llvm-cov | https://github.com/taiki-e/cargo-llvm-cov |
| cargo-mutants | https://github.com/sourcefrog/cargo-mutants |
| cargo-fuzz | https://github.com/rust-fuzz/cargo-fuzz |
| afl.rs | https://github.com/rust-fuzz/afl.rs |
| Rust Fuzz Book | https://rust-fuzz.github.io/book/ |
