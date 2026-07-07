---
title: Rust - Naming Conventions
status: draft
version: 0.0.1
created: 2026-07-06
updated: 2026-07-06
---

# Rust — Naming Conventions

Rust's naming conventions are finalized in **RFC 430** (`rust-lang.github.io/rfcs/0430-finalizing-naming-conventions.html`) and elaborated by the **Rust API Guidelines — Naming** chapter (`rust-lang.github.io/api-guidelines/naming.html`). They are enforced, in part, by built-in compiler lints (`non_snake_case`, `non_upper_case_globals`, `non_camel_case_types`) and by Clippy.

## Core rules

The single organizing principle: **type-level constructs use `UpperCamelCase`; value-level constructs use `snake_case`.**

| Construct | Convention | Example |
|---|---|---|
| Crates | lowercase, digits, `_` or `-`; prefer kebab-case in `Cargo.toml`, underscores in `use` | `serde-json` → `serde_json` |
| Modules | `snake_case` | `std::collections` |
| Types (`struct`, `enum`, `union`) | `UpperCamelCase` | `HttpClient`, `OrderStatus` |
| Traits | `UpperCamelCase` | `Display`, `IntoIterator` |
| Type parameters | `UpperCamelCase`, single letter or short word | `T`, `K`, `V`, `Element` |
| Functions / methods | `snake_case` | `read_to_string`, `as_str` |
| Local variables / parameters | `snake_case` | `request_count`, `buf` |
| Constants / statics / enum variants | `SCREAMING_SNAKE_CASE` | `MAX_CONNECTIONS`, `Color::RED` |
| Lifetimes | short lowercase, `'a` for generic, descriptive for clarity | `'a`, `'src`, `'input` |
| Macros | `snake_case` (functions), `UpperCamelCase` if like a type | `println!`, `vec!` |

### Acronyms

Per RFC 430 and the API Guidelines:

- In `UpperCamelCase`, **acronyms and contractions count as one word**: `Uuid` (not `UUID`), `Usize` (not `USize`), `Stdin` (not `StdIn`), `HttpRequest` (not `HTTPRequest`).
- In `snake_case`, acronyms are fully lower-cased: `is_xid_start`, `parse_uuid`, `to_ascii_lowercase`.

> Note: the `Uuid`-over-`UUID` recommendation is one of the more contested points in the guidelines; some codebases keep the all-caps acronym for readability. Either way, be **consistent within a crate**.

## Type-level naming details

✅ **Do** — `UpperCamelCase` for all type-level constructs:

```rust
// Types, traits, enum variants, type params all UpperCamelCase.
struct HttpClient { /* ... */ }
enum OrderStatus { Pending, Shipped, Delivered }
trait IntoBytes { fn into_bytes(self) -> Vec<u8>; }
fn parse<T: FromStr>(input: &str) -> Option<T> { /* ... */ unimplemented!() }
```

❌ **Don't** — `non_camel_case_types` rejects these:

```rust
struct HTTP_Client { /* ... */ }      // wrong casing + acronym
enum order_status { pending }          // not UpperCamelCase
trait into_bytes { }                   // trait names are UpperCamelCase
```

### Getters / setters (API Guidelines C-GETTER)

Per the API Guidelines, **accessor methods do not take a `get_` prefix**:

```rust
impl Person {
    // ✅ Idiomatic
    pub fn name(&self) -> &str { &self.name }
    pub fn first_name(&self) -> &str { &self.first_name }

    // ❌ Don't prefix with get_
    pub fn get_name(&self) -> &str { &self.name }
}
```

A setter for field `x` is named `set_x`. The Clippy lint `wrong_self_convention` and `get_first`/`get_last` style checks back this up. Property-style words (`is_`, `has_`, `with_`) are fine for predicates and builders.

### Conversion methods (API Guidelines C-COMMON-TRAITS, C-CONV)

| Conversion | Convention | Source trait / mechanism |
|---|---|---|
| `T::from(x)` | `from` | `From` |
| `x.into()` | `into` | `Into` (often the reciprocal) |
| `x.as_ref()` / `x.as_mut()` | `as_ref` / `as_mut` | `AsRef` / `AsMut` |
| `x.as_str()`, `x.as_bytes()` | `as_<noun>` | inherent method |
| `x.to_string()` | `to_string` | `ToString` / `Display` |
| `x.to_owned()` | `to_owned` | `ToOwned` |
| `T::try_from(x)` | `try_from` | `TryFrom` |

`as_` performs a **cheap, borrowed** conversion; `to_` performs an **expensive, owned** conversion; `into_` consumes `self`. Clippy's `wrong_self_convention` lint codifies this.

## Value-level naming details

✅ **Do** — `snake_case` for values:

```rust
fn fetch_user(user_id: u64) -> Option<User> {
    let cached_user = CACHE.get(&user_id);
    let retry_count = 0;
    /* ... */ unimplemented!()
}
```

❌ **Don't** — `non_snake_case` warns:

```rust
fn fetchUser(userId: u64) -> Option<User> { /* camelCase */ unimplemented!() }
let UserID = 42;                       // local vars are snake_case
static MaxRetries: u32 = 3;            // statics are SCREAMING_SNAKE_CASE
```

## Constants and statics

```rust
const MAX_CONNECTIONS: u32 = 100;
static DEFAULT_TIMEOUT: Duration = Duration::from_secs(30);

enum HttpStatus {
    Ok = 200,            // ❌ variant should be SCREAMING_SNAKE_CASE
    NotFound = 404,
}

enum HttpStatus {
    OK = 200,            // ✅
    NOT_FOUND = 404,
}
```

> **Module-level singletons** that *behave* like a value of a type (e.g. `std::f64::consts::PI`) are written in `snake_case` because they are static items accessed as values, per the API Guidelines exception. When in doubt, follow the standard library.

## Lifetimes

- Use `'a`, `'b`, `'c` for **unimportant, generic** lifetimes.
- Use a **descriptive** lifetime when it carries meaning: `'src`, `'input`, `'static`.

```rust
// ✅ Generic, unimportant.
fn first_word<'a>(s: &'a str) -> &'a str { /* ... */ unimplemented!() }

// ✅ Descriptive when meaningful.
struct Lexer<'src> { source: &'src str }
fn tokenize<'src>(input: &'src str) -> TokenStream<'src> { /* ... */ unimplemented!() }
```

Prefer **lifetime elision** wherever the compiler allows it — explicit annotations are noise when the elision rules already cover the case.

## Crate naming

On crates.io, crate names are lower-case ASCII and may use dashes or underscores, but **dashes are normalized to underscores** when imported. The community convention (RFC 1105, API Guidelines C-CRATE-NAME):

- Lowercase, digits, dashes or underscores only.
- Avoid `-rs` / `-rust` suffixes (e.g. `serde`, not `serde-rs`).
- Prefer dashes in `Cargo.toml` (`serde-json`); the corresponding `use` is `serde_json`.

```toml
[dependencies]
serde_json = "1"
```

```rust
// Dashes in the crate name become underscores in code.
use serde_json::Value;
```

## Lints that enforce naming

```rust
#![warn(
    non_snake_case,          // functions, vars, modules must be snake_case
    non_upper_case_globals,  // consts/statics must be SCREAMING_SNAKE_CASE
    non_camel_case_types,    // types/traits must be UpperCamelCase
)]
```

Clippy adds style lints such as `wrong_self_convention`, `module_inception`, `enum_variant_names`, `should_implement_trait`, and `new_without_default`. Per the API Guidelines, **`new` is the conventional constructor name** when there is exactly one obvious constructor; otherwise use descriptive inherent constructors (`with_capacity`, `from_reader`, etc.).

## Summary checklist

- [ ] One casing rule per kind of identifier (RFC 430).
- [ ] No `get_` prefix on accessors.
- [ ] Acronyms treated as single words in `UpperCamelCase`.
- [ ] `as_` borrowed/cheap, `to_` owned/expensive, `into_` consuming.
- [ ] Crate name lower-case, no `-rs` suffix.
- [ ] `non_snake_case` / `non_upper_case_globals` / `non_camel_case_types` enabled.

## References

- RFC 430 — Finalizing naming conventions — `rust-lang.github.io/rfcs/0430-finalizing-naming-conventions.html`.
- Rust API Guidelines — Naming — `rust-lang.github.io/api-guidelines/naming.html`.
- Rust API Guidelines — Checklist — `rust-lang.github.io/api-guidelines/checklist.html`.
- Clippy lint list — `rust-lang.github.io/rust-clippy/master/`.
