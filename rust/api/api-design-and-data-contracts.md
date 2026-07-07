---
title: Rust - API Design and Data Contracts
status: draft
version: 0.0.1
created: 2026-07-06
updated: 2026-07-06
---

# Rust — API Design and Data Contracts

The canonical reference for Rust API design is the **Rust API Guidelines** (`rust-lang.github.io/api-guidelines/`), maintained by the Rust library team. It is organized into four pillars — **Interoperability, Predictability, Debuggability, and Type System / Documentation** — and codifies conventions that, when followed, make a crate feel "rusty" to its users. This section catalogs the load-bearing rules and the data-contract patterns (`serde`, `From`/`Into`, builder, newtype, `Send`/`Sync`, error types).

## The four pillars of the API Guidelines

1. **Interoperability** — your types play well with the ecosystem: `serde`, `From`/`Into`, `AsRef`, `IntoIterator`, `Send + Sync`.
2. **Predictability** — behavior matches the type signature; no surprising panics, hidden allocations, or blocking I/O on async-looking functions.
3. **Debuggability** — public types implement `Debug`; errors carry enough context to diagnose.
4. **Documentation / Type System** — public items are documented; types encode invariants (newtypes, builder, sealed traits).

## Naming in public APIs

(Section 2 covers naming in detail; the API-Guidelines-specific points belong here.)

- `new` is the constructor name when there is exactly one obvious way to construct; otherwise use descriptive inherent constructors (`from_reader`, `with_capacity`).
- Accessors do **not** take a `get_` prefix (`fn name(&self)`, not `fn get_name`).
- Conversion naming: `as_` (cheap/borrowed), `to_` (expensive/owned), `into_` (consuming).
- Traits are nouns (`Iterator`) or adjectives (`Display`, `Clone`).

## Conversions: `From` / `Into` / `AsRef` / `TryFrom`

✅ **Do** — implement `From` so callers get `Into` for free, and so `?` can widen errors automatically:

```rust
#[derive(Debug, thiserror::Error)]
pub enum AppError {
    #[error(transparent)]
    Io(#[from] std::io::Error),

    #[error(transparent)]
    Parse(#[from] std::num::ParseIntError),
}

impl From<std::io::Error> for AppError { /* #[from] generates this */ unimplemented!() }

// Because From<io::Error> is implemented, ? widens automatically:
fn read_n() -> Result<u64, AppError> {
    let s = std::fs::read_to_string("n")?;   // io::Error -> AppError via From
    Ok(s.trim().parse()?)                     // ParseIntError -> AppError via From
}
```

✅ **Do** — accept generic conversions at the API boundary with `impl Into<...>` / `impl AsRef<...>`:

```rust
pub struct Client { host: String }

impl Client {
    // Accepts &str, String, &'static str, anything that is Into<String>.
    pub fn new(host: impl Into<String>) -> Self {
        Self { host: host.into() }
    }

    // Accepts &[u8], Vec<u8>, &str, anything AsRef<[u8]>.
    pub fn send(&self, body: impl AsRef<[u8]>) {
        let bytes: &[u8] = body.as_ref();
        /* ... */
    }
}
```

❌ **Don't** — over-constrain callers to a single owned type:

```rust
impl Client {
    pub fn new(host: String) -> Self { /* forces callers to allocate */ unimplemented!() }
    pub fn send(&self, body: Vec<u8>) { /* forces callers to own */ unimplemented!() }
}
```

## Iterability — `IntoIterator` and `Iterator` return types

Public collection types should implement `IntoIterator` for `&T`, `&mut T`, and `T` (by value), so callers can iterate by reference or by value with `for x in &collection`.

```rust
pub struct Users(Vec<User>);

impl<'a> IntoIterator for &'a Users {
    type Item = &'a User;
    type IntoIter = std::slice::Iter<'a, User>;
    fn into_iter(self) -> Self::IntoIter { self.0.iter() }
}
```

Returning `impl Iterator<Item = T>` from a function lets you hide the concrete iterator type while still exposing a composable iterator — preferable to `Vec<T>` when you only mean to be iterated.

## `Send` and `Sync` (API Guidelines C-SEND-SYNC)

`Send` (movable across threads) and `Sync` (sharable across threads via `&T`) are **auto traits**: the compiler implements them automatically when all fields are `Send`/`Sync`. The Guideline: **types should be `Send + Sync` wherever possible**. Opt out only with a good reason (e.g. `Rc<T>` is intentionally `!Send`/`!Sync`; raw pointers are `!Send`/`!Sync`).

```rust
// ✅ Default — Send + Sync because all fields are.
pub struct Config {
    values: std::collections::HashMap<String, String>,
}

// ❌ Opting out without justification — adds friction for callers.
pub struct Config {
    values: std::collections::HashMap<String, String>,
    _not_sync: std::cell::Cell<()>,   // makes Config !Sync for no reason
}
```

When you must opt out (e.g. you hold an `Rc`), document it so callers know they cannot move the type across threads.

## Serde — serialization contracts

`serde` is the de facto serialization framework. The `Serialize`/`Deserialize` derive macros implement the traits with attributes for renaming, skipping, defaults, and transparent wrappers.

✅ **Do** — derive `Serialize`/`Deserialize` for data types and pin a stable representation:

```rust
use serde::{Serialize, Deserialize};

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct UserDto {
    #[serde(rename = "userId")]
    pub user_id: u64,
    #[serde(rename = "displayName")]
    pub display_name: String,
    #[serde(default, skip_serializing_if = "Option::is_none")]
    pub email: Option<String>,
}
```

- `rename` decouples the wire field name from the Rust field name (snake_case in Rust, camelCase in JSON).
- `default` lets you add new optional fields without breaking older clients.
- `skip_serializing_if` avoids emitting `null`.
- `#[serde(deny_unknown_fields)]` makes the contract strict; omit it for forward compatibility.

### Format crates (side by side)

| Format | Crate | Notes |
|---|---|---|
| JSON | `serde_json` | The default for HTTP APIs. |
| Binary, schemaless | `bincode` | Compact, Rust-to-Rust. |
| Binary, schema | `prost` (Protobuf), `capnp` (Cap'n Proto), `flatbuffers` | Cross-language, schema-driven. |
| TOML | `toml` | Config files. |
| YAML | `serde_yaml` | Config files (note: parser is in maintenance). |
| MessagePack | `rmp-serde` | Compact JSON alternative. |

The standard documents these side by side; selection is application-specific.

## Builder pattern for APIs

For types with many optional fields or non-trivial invariants, a builder (Section 5 covers the mechanics) is the standard API surface. Recognized variants:

- **Owned builder** — `fn field(self, v) -> Self`. Most common; lets you chain.
- **Mutable builder** — `fn field(&mut self, v) -> &mut Self`. Useful when shared.
- **Type-state builder** — uses type parameters to make required fields compile-time-checked. Implemented by `typed-builder`, `bon`, `derive_builder`.

```rust
let req = Request::builder()
    .method("POST")
    .url(url)
    .header("accept", "application/json")
    .body(payload)
    .build()?;
```

## Newtype pattern for API boundaries

A newtype gives a primitive a distinct identity, prevents argument-order bugs, and can carry validation:

```rust
pub struct Email(String);

impl Email {
    pub fn parse(s: impl Into<String>) -> Result<Self, ParseEmailError> {
        let s = s.into();
        if is_valid(&s) { Ok(Email(s)) } else { Err(ParseEmailError) }
    }
}

impl AsRef<str> for Email { fn as_ref(&self) -> &str { &self.0 } }
```

The API now cannot accidentally accept an arbitrary `&str` where an `Email` is required.

## Error types in public APIs

Public APIs should expose **typed, matchable errors** (Section 7's `thiserror` pattern), not `anyhow::Error`. Callers need to be able to react differently to `NotFound` vs. `PermissionDenied` vs. `Timeout`.

✅ **Do:**

```rust
#[derive(Debug, thiserror::Error)]
pub enum StoreError {
    #[error("not found: {key}")]
    NotFound { key: String },
    #[error("permission denied")]
    PermissionDenied,
    #[error(transparent)]
    Backend(#[from] io::Error),
}

pub fn get(&self, key: &str) -> Result<Vec<u8>, StoreError> { /* ... */ unimplemented!() }
```

❌ **Don't** — leak `anyhow::Error` across a public API boundary:

```rust
pub fn get(&self, key: &str) -> Result<Vec<u8>, anyhow::Error> { /* callers can't match */ unimplemented!() }
```

Document the error variants in `# Errors` (Section 6).

## Public API stability and semver

Rust's semver convention, codified by the **cargo semver reference** (`doc.rust-lang.org/cargo/reference/semver.html`) and the API Guidelines:

- A **major** bump (1.x → 2.0) is required for any breaking change to the public API. "Public" = anything reachable from a non-`pub(crate)` item in `lib.rs`.
- A **minor** bump (1.0 → 1.1) adds backwards-compatible features: new items, new trait impls, new `#[derive]`s, new optional dependencies.
- A **patch** bump (1.0.0 → 1.0.1) is for bug fixes that don't change the API.

### Non-obvious **breaking** changes (per the cargo semver reference)

These trip up newcomers — they are all **major** bumps:

- Adding a new public variant to an enum (callers' exhaustive `match` breaks).
- Adding a new field to a public `struct` (callers' struct literals break) — *unless* the field is `pub` and has a default via `#[non_exhaustive]` or `Default`.
- Adding a new method to a trait (downstream impls break).
- Tightening a function's generic bounds.
- Implementing a new trait that introduces a method-name collision.
- Changing `#[derive]`s in ways that affect the public surface.

Use `#[non_exhaustive]` on public structs and enums to reserve the right to add fields/variants without a major bump:

```rust
#[non_exhaustive]
pub struct Options {
    pub timeout: Duration,
    // New fields can be added in a minor bump because callers cannot
    // construct this struct with a literal — they must use a builder or Default.
}

Options { timeout: Duration::from_secs(30), ..Default::default() }
```

```rust
#[non_exhaustive]
pub enum Event {
    Click,
    KeyPress(char),
    // New variants can be added in a minor bump; callers must include `_`.
}
```

## Trait design

- **Sealed traits** prevent downstream implementations while still being usable — the standard way to lock down an extension trait:

  ```rust
  mod private { pub trait Sealed {} }
  pub trait MyTrait: private::Sealed { /* ... */ }
  impl private::Sealed for u8 {}
  impl MyTrait for u8 {}
  ```

- Prefer **small, focused traits** over mega-traits; combine via supertrait bounds.
- `impl Trait` in argument position (`fn f(x: impl Into<String>)`) is sugar for a generic; in return position it hides the concrete type and is a stability win (you can change the returned type later).
- Document the contract — a trait is an API; its doc comment should explain the invariants implementers must uphold.

## Summary checklist

- [ ] Public types implement `Debug`, `Clone` where sensible, `Send + Sync` where possible.
- [ ] Conversions via `From`/`Into`/`AsRef`; accept `impl Into<...>`/`impl AsRef<...>` at boundaries.
- [ ] Collections implement `IntoIterator` for `&T`, `&mut T`, `T`.
- [ ] Data types derive `Serialize`/`Deserialize` with stable wire contracts.
- [ ] Public errors are typed (not `anyhow::Error`).
- [ ] `#[non_exhaustive]` on structs/enums that may grow.
- [ ] Semver followed per the cargo semver reference; major bump on any breaking change.

## References

- Rust API Guidelines — `rust-lang.github.io/api-guidelines/` (checklist, interoperability, predictability, debuggability, documentation).
- The Cargo Book — Semver compatibility — `doc.rust-lang.org/cargo/reference/semver.html`.
- Serde — `serde.rs/`.
- The Rust Reference — Traits / impl Trait — `doc.rust-lang.org/reference/`.
- *Effective Rust* — `effective-rust.com/` (newtype, sealed traits, builder).
