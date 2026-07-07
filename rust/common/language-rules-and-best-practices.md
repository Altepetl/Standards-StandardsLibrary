---
title: Rust - Language Rules and Best Practices
status: draft
version: 0.0.1
created: 2026-07-06
updated: 2026-07-07
---

# Rust — Language Rules and Best Practices

This section catalogs the idioms that distinguish idiomatic Rust from code that merely compiles. The throughline: **work with the borrow checker, treat errors as values, prefer combinators over imperative plumbing, and use `unsafe` only with a safety proof.** Clippy (`rust-lang.github.io/rust-clippy/master/`) is the de facto enforcer of these idioms.

## Typing Model Overview

Rust is a statically typed, strongly typed language with type inference.
- **Static and Strong:** All types are known at compile time, and implicit type conversions (coercions) are strictly limited (e.g., deref coercion). You cannot silently mix mismatched types.
- **Type Inference:** The compiler infers types for most local variables. Explicit type annotations are typically only required in function signatures, `const`/`static` declarations, or when an expression is ambiguous (e.g., using the turbofish syntax `"42".parse::<u32>().unwrap()`).
- **Traits as Interfaces:** Rust does not use classical inheritance. Instead, behavior is shared using traits. A type can implement a trait, and functions can accept generic types constrained by traits.
- **Algebraic Data Types (ADTs):** Enums in Rust can contain data, allowing for powerful representation of state (e.g., `Option<T>` and `Result<T, E>`).

Idiomatic Rust leverages the type system to enforce invariants at compile time (e.g., the Type State pattern or using newtypes).

## Ownership, borrowing, lifetimes

### Borrow instead of clone

The single most common Rust smell is unnecessary `.clone()`. Cloning is sometimes correct (you need an owned value) and sometimes a band-aid that masks a borrow-checker fight.

✅ **Do** — borrow where you only need to read:

```rust
fn total_length(names: &[String]) -> usize {
    names.iter().map(|s| s.len()).sum()
}

let names = vec![String::from("alice"), String::from("bob")];
println!("{}", total_length(&names));     // borrow, names still usable
```

❌ **Don't** — clone just to satisfy the checker:

```rust
fn total_length(names: Vec<String>) -> usize {   // takes ownership unnecessarily
    names.iter().map(|s| s.len()).sum()
}

let names = vec![String::from("alice"), String::from("bob")];
println!("{}", total_length(names.clone()));     // wasteful clone
// `names` is gone here even though we only read it.
```

Clippy lints to enable: `clippy::redundant_clone`, `clippy::unnecessary_to_owned`, `clippy::cloned_refinstead_of_copied`.

### Where a clone *is* correct

```rust
// Spawning a thread requires 'static data; Arc or clone is legitimate.
let snapshot = state.clone();
std::thread::spawn(move || process(snapshot));

// You genuinely need two owned copies.
let mut a = String::from("hello");
let b = a.clone();
a.push_str(" world");
```

The rule of thumb: clone when you **need** ownership; borrow when you only **read**.

### Prefer `&str` and `&[T]` over `&String` and `&Vec<T>`

`&str` and `&[T]` are the borrowed slices; `String` and `Vec<T>` are the owned containers. Function arguments should almost always take the slice form — it accepts more types via deref coercion.

✅ **Do:**

```rust
fn greet(name: &str) { println!("hi {name}"); }
fn sum(xs: &[i32]) -> i32 { xs.iter().sum() }

greet(&String::from("alice"));  // works
greet("alice");                 // works
sum(&vec![1, 2, 3]);            // works
sum(&[1, 2, 3]);                // works
```

❌ **Don't** — over-constrain callers:

```rust
fn greet(name: &String) { /* ... */ }   // rejects &str, only &String
fn sum(xs: &Vec<i32>) -> i32 { /* ... */ }
```

Clippy: `clippy::ptr_arg` flags `&Vec<T>` and `&String` in function signatures.

### `Cow` for borrowed-or-owned

Use `Cow<'a, B>` when a function *might* need to allocate but usually doesn't — typical for parsers/validators that return the input untouched when valid and an owned, modified copy otherwise.

```rust
use std::borrow::Cow;

fn normalize(input: &str) -> Cow<'_, str> {
    if input.chars().any(|c| c.is_uppercase()) {
        Cow::Owned(input.to_lowercase())   // allocation needed
    } else {
        Cow::Borrowed(input)               // zero allocation
    }
}
```

### Lifetimes: prefer elision, be descriptive when not

Lifetime elision handles the common cases; reach for explicit annotations only when the compiler requires them or when they carry meaning.

```rust
// Elided — clear enough.
fn first_word(s: &str) -> &str { /* ... */ "" }

// Explicit + descriptive when meaning matters.
struct Parser<'src> { source: &'src str }
impl<'src> Parser<'src> {
    fn tokens(&self) -> impl Iterator<Item = Token<'src>> { /* ... */ unimplemented!() }
}
```

Clippy: `clippy::needless_lifetimes` flags explicit lifetimes the compiler could infer.

## Error handling — values, not exceptions

(Section 7 covers this in depth; the headline rules belong here too.)

- Operations that can fail return `Result<T, E>` or `Option<T>`, never panic.
- Propagate with `?`.
- `unwrap()` / `expect()` are for **tests, prototypes, and invariants you can prove** — never for fallible I/O in production paths.

✅ **Do:**

```rust
use std::num::ParseIntError;

fn parse_port(s: &str) -> Result<u16, ParseIntError> {
    s.parse::<u16>()
}

fn load_port() -> Result<u16, ParseIntError> {
    let raw = std::env::var("PORT").unwrap();   // ❌ env::var can fail
    raw.parse::<u16>()                          // ❌ parse can fail
}
```

❌ The same code done right:

```rust
fn load_port() -> Result<u16, Box<dyn std::error::Error>> {
    let raw = std::env::var("PORT")?;           // ? propagates VarError
    Ok(raw.parse::<u16>()?)                     // ? propagates ParseIntError
}
```

## Pattern matching

`match` is exhaustive; the compiler forces every case (or a wildcard) to be handled. Prefer it to chains of `if let` when there are ≥2 cases.

✅ **Do** — exhaustive match on enums:

```rust
fn label(s: Status) -> &'static str {
    match s {
        Status::Active => "active",
        Status::Paused => "paused",
        Status::Stopped => "stopped",
    }
}
```

❌ **Don't** — fragile wildcard that silently drops new variants:

```rust
fn label(s: Status) -> &'static str {
    match s {
        Status::Active => "active",
        _ => "other",          // adding Status::Crashed is silently mislabeled
    }
}
```

Clippy: `clippy::wildcard_enum_match_arm`, `clippy::match_same_arms`. Bindings, guards, and `@` bindings are idiomatic.

## Iterator chains over explicit loops

Iterators are zero-cost (they monomorphize and LLVM optimizes the chain). Prefer `map`/`filter`/`fold`/`collect` over index-based `for` loops where the chain reads clearly.

✅ **Do:**

```rust
let squares: Vec<i32> = (1..=10).map(|n| n * n).collect();
let sum: i32 = (1..=100).filter(|n| n % 2 == 0).sum();
let by_id: HashMap<u64, User> = users.into_iter().map(|u| (u.id, u)).collect();
```

❌ **Don't** — imperative plumbing the iterator would replace:

```rust
let mut squares = Vec::new();
let mut n = 1;
while n <= 10 {
    squares.push(n * n);
    n += 1;
}
```

For loops *are* idiomatic when the body has side effects or early returns and there is no natural collection to build:

```rust
for line in stdin.lines() {
    let line = line?;
    if line.starts_with("STOP") { break; }
    process(&line)?;
}
```

Clippy: `clippy::manual_map`, `clippy::manual_filter`, `clippy::needless_range_loop`.

## The newtype pattern

A tuple struct wrapping a single value gives it a distinct type, prevents mixing units, and is zero-cost.

```rust
pub struct UserId(pub u64);
pub struct OrderId(pub u64);

fn fetch_order(user: UserId, order: OrderId) -> Order { /* ... */ unimplemented!() }

// The compiler now rejects:
// fetch_order(OrderId(1), UserId(2));   // type error — units can't be swapped
```

Per *Effective Rust* (Item 6) and the API Guidelines, newtypes are the idiomatic way to give semantic meaning to primitives and to opt into/out of trait implementations.

## The builder pattern

For constructors with many optional parameters, a builder (manual or via `derive_builder` / `bon` / `typed-builder`) is idiomatic.

```rust
pub struct Server {
    host: String,
    port: u16,
    tls: bool,
}

impl Server {
    pub fn builder() -> ServerBuilder { ServerBuilder::default() }
}

#[derive(Default)]
pub struct ServerBuilder {
    host: Option<String>,
    port: Option<u16>,
    tls: Option<bool>,
}

impl ServerBuilder {
    pub fn host(mut self, host: impl Into<String>) -> Self { self.host = Some(host.into()); self }
    pub fn port(mut self, port: u16) -> Self { self.port = Some(port); self }
    pub fn tls(mut self, tls: bool) -> Self { self.tls = Some(tls); self }

    pub fn build(self) -> Result<Server, &'static str> {
        Ok(Server {
            host: self.host.ok_or("host is required")?,
            port: self.port.unwrap_or(8080),
            tls: self.tls.unwrap_or(false),
        })
    }
}
```

Recognized builder variants: **owned builder** (the above, returns `Self`), **mutable builder** (`&mut self` setters), **type-state builder** (a `typed-builder`/`bon` pattern that makes required fields compile-time-checked).

## `unsafe` Rust

`unsafe` does **not** turn off the borrow checker; it unlocks five superpowers (per the Rustonomicon — `doc.rust-lang.org/nomicon/`):

1. Dereference a raw pointer.
2. Call an `unsafe` function or method.
3. Implement an `unsafe` trait.
4. Access or modify a mutable static.
5. Access fields of a `union`.

Every `unsafe` block **must** carry a `// SAFETY:` comment documenting *why* the operation is sound — this is now community norm and enforced by Clippy's `clippy::undocumented_unsafe_blocks` and the `unsafe_op_in_unsafe_fn` lint.

```rust
// ✅ Documented unsafe.
fn split_at_mut(xs: &mut [i32], mid: usize) -> (&mut [i32], &mut [i32]) {
    let len = xs.len();
    assert!(mid <= len);
    // SAFETY: `mid` is bounded by `len`, so the two slices are non-overlapping
    // and together cover the whole input.
    let ptr = xs.as_mut_ptr();
    unsafe {
        (
            std::slice::from_raw_parts_mut(ptr, mid),
            std::slice::from_raw_parts_mut(ptr.add(mid), len - mid),
        )
    }
}
```

❌ **Don't** use `unsafe` to escape the borrow checker when you don't have a proof of soundness — that defeats the language's central guarantee and is the most common source of security bugs in Rust (Section 9).

## Discouraged and Deprecated Features

While Rust guarantees backward compatibility, some patterns and features are strongly discouraged by the community and tooling:
- **`std::mem::uninitialized`:** Officially deprecated. Use `std::mem::MaybeUninit` instead. `uninitialized` is inherently unsafe because it allows the creation of invalid values (e.g., a boolean that is neither true nor false), leading to immediate undefined behavior.
- **`try!` macro:** Deprecated in favor of the `?` operator, which is more concise and chains elegantly.
- **Global Mutable State (`static mut`):** While available, its use is heavily discouraged and typically requires `unsafe` blocks just to access. Prefer `std::sync::Mutex`, `RwLock`, or `std::sync::OnceLock` / `std::sync::LazyLock` for safe global state.

## Complexity Limits

Clippy sets community-recognized defaults for complexity limits. While these can be tuned, the defaults reflect idiomatic expectations:
- **Function Length (`clippy::too_many_lines`):** Functions longer than 100 lines trigger a warning. Idiomatic Rust breaks large logic blocks into smaller, testable helper functions.
- **Argument Count (`clippy::too_many_arguments`):** Functions with more than 7 arguments trigger a warning. Use structs or the Builder pattern to group related parameters.
- **Cognitive Complexity (`clippy::cognitive_complexity`):** Warns when a function's control flow becomes too deeply nested or convoluted (threshold default is 25). Early returns and breaking out logic into separate functions are the preferred mitigations.
- **Type Complexity (`clippy::type_complexity`):** Warns on deeply nested types (e.g., `Vec<Vec<Box<(u32, u32)>>>`). Use type aliases or define named structs/enums to clarify intent.

## Clippy — the best-practices linter

`cargo clippy` runs the rustc lints plus hundreds of style/correctness/perf lints. Lint groups, in increasing strictness:

| Group | Intent |
|---|---|
| `correctness` | Likely-buggy code (enabled by default, deny). |
| `suspicious` | Probably-wrong code (warn). |
| `style` | Idiomatic formatting/structure (warn). |
| `complexity` | Code that could be simpler (warn). |
| `perf` | Code that could be faster (warn). |
| `pedantic` | Opinionated extra checks (allow by default). |
| `restriction` | Forbid specific features (allow by default; opt-in). |

A common project-level config:

```rust
// src/lib.rs or src/main.rs
#![warn(
    clippy::pedantic,
    clippy::unwrap_used,
    clippy::expect_used,
)]
#![allow(
    clippy::module_name_repetitions,   // sometimes clearer
    clippy::must_use_candidate,        // noisy in some codebases
)]
```

Two recognized positions: enable `pedantic` for strictness, or stay on default + `style` for lower noise. The standard is **to have a deliberate `clippy` configuration**, not to leave it at defaults unconsciously.

## Summary checklist

- [ ] Understanding of the strong, static typing model and use of type inference.
- [ ] Borrow instead of clone; `&str` / `&[T]` in signatures.
- [ ] `Result`/`Option` + `?` for fallible code; no `unwrap` in production paths.
- [ ] Exhaustive `match`; avoid fragile wildcards on enums.
- [ ] Iterator chains where they read clearly.
- [ ] Newtype for distinct primitive identities.
- [ ] Builder for multi-optional constructors.
- [ ] Every `unsafe` block has a `// SAFETY:` comment.
- [ ] Avoidance of deprecated features like `std::mem::uninitialized` or `try!`.
- [ ] Adherence to community complexity limits (function length, argument count, cognitive complexity).
- [ ] Deliberate `clippy` configuration in `lib.rs`/`main.rs`.

## References

- The Rustonomicon — `doc.rust-lang.org/nomicon/` (`unsafe` rules).
- Clippy lint list — `rust-lang.github.io/rust-clippy/master/`.
- *Effective Rust* — `effective-rust.com/` (newtype, builders, idioms).
- The Rust API Guidelines — `rust-lang.github.io/api-guidelines/`.
- The Rust Reference — `doc.rust-lang.org/reference/`.
