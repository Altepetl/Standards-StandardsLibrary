---
title: Rust - API Design and Data Contracts
status: draft
version: 0.1.0
created: 2026-07-06
updated: 2026-07-06
---

# Rust — API Design and Data Contracts

The canonical reference for Rust API design is the **Rust API Guidelines** (`rust-lang.github.io/api-guidelines/`), maintained by the Rust library team. It is organized into four pillars — **Interoperability, Predictability, Debuggability, and Type System / Documentation** — and codifies conventions that, when followed, make a crate feel "rusty" to its users. This section catalogs the load-bearing rules and the data-contract patterns (`serde`, `From`/`Into`, builder, newtype, `Send`/`Sync`, error types), and extends them to cover **Web API Design** and **REST Data Contracts**.

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

❌ **Don't** — over-constrain callers to a single owned type.

## Iterability — `IntoIterator` and `Iterator` return types

Public collection types should implement `IntoIterator` for `&T`, `&mut T`, and `T` (by value), so callers can iterate by reference or by value with `for x in &collection`.

Returning `impl Iterator<Item = T>` from a function lets you hide the concrete iterator type while still exposing a composable iterator — preferable to `Vec<T>` when you only mean to be iterated.

## `Send` and `Sync` (API Guidelines C-SEND-SYNC)

`Send` (movable across threads) and `Sync` (sharable across threads via `&T`) are **auto traits**: the compiler implements them automatically when all fields are `Send`/`Sync`. The Guideline: **types should be `Send + Sync` wherever possible**. Opt out only with a good reason (e.g. `Rc<T>` is intentionally `!Send`/`!Sync`; raw pointers are `!Send`/`!Sync`).

When you must opt out (e.g. you hold an `Rc`), document it so callers know they cannot move the type across threads.

## Web API Patterns and REST Conventions

When building web services (using frameworks like `axum` or `actix-web`), Rust adopts standard REST and HTTP conventions, leveraging the type system to enforce them.

### URL/Resource Naming Conventions

- **Kebab-case**: Use lowercase kebab-case for URL segments (`/api/v1/user-profiles`).
- **Plural nouns**: Resources should be named with plural nouns (`/users/123/orders`).
- **No verbs in paths**: Use HTTP methods (`GET`, `POST`, `PUT`, `DELETE`) to define actions rather than putting verbs in the URL (except for specific RPC-style actions like `/users/123/activate`).

### Web Versioning Strategies

Web APIs in Rust typically handle versioning at the **URI routing layer**:

- **URI Versioning (`/api/v1/...`)**: Recommended. Easy to route to different handler modules in `axum` or `actix-web`, and simple for clients to understand. Use for major, breaking changes.
- **Header Versioning / Content Negotiation**: Implemented via custom extractors (e.g., checking `Accept: application/vnd.myapi.v1+json`). More flexible but increases routing complexity and caching overhead.

### Contract-First Tooling

- **OpenAPI / Swagger**: The `utoipa` crate is the de facto standard for code-first OpenAPI generation in Rust. It uses procedural macros on struct definitions and handler functions to generate the OpenAPI spec at compile time.
- **Protobuf / gRPC**: The `prost` and `tonic` crates are the canonical tools for contract-first gRPC services. They generate Rust types from `.proto` files via `build.rs`.
- **GraphQL**: The `async-graphql` crate is standard for GraphQL APIs, using a code-first approach where Rust types generate the GraphQL schema.

## Serde — serialization contracts

`serde` is the de facto serialization framework. The `Serialize`/`Deserialize` derive macros implement the traits with attributes for renaming, skipping, defaults, and transparent wrappers.

### Payload Field Casing

Rust uses `snake_case` internally, but JSON APIs typically use `camelCase`. 
✅ **Do** — Use `serde`'s `rename_all` attribute to bridge this gap automatically for web payloads:

```rust
use serde::{Serialize, Deserialize};

#[derive(Debug, Clone, Serialize, Deserialize)]
#[serde(rename_all = "camelCase")]
pub struct UserDto {
    pub user_id: u64,           // Serializes as "userId"
    pub display_name: String,   // Serializes as "displayName"
    #[serde(default, skip_serializing_if = "Option::is_none")]
    pub email: Option<String>,
}
```

- `rename` decouples the wire field name from the Rust field name.
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

## Error types in public APIs and Web Responses

Public APIs should expose **typed, matchable errors** (Section 7's `thiserror` pattern), not `anyhow::Error`. Callers need to be able to react differently to `NotFound` vs. `PermissionDenied` vs. `Timeout`.

### Standard Error Responses (RFC 7807)

For web APIs, errors should be structured and predictable. The ecosystem widely adopts **RFC 7807 (Problem Details for HTTP APIs)**.

✅ **Do** — Implement `IntoResponse` (in `axum`) or `ResponseError` (in `actix-web`) to map your internal typed errors to a standard JSON problem detail payload:

```rust
use axum::{http::StatusCode, response::{IntoResponse, Response}, Json};
use serde::Serialize;

#[derive(Serialize)]
pub struct ProblemDetails {
    pub r#type: String,
    pub title: String,
    pub status: u16,
    pub detail: String,
}

impl IntoResponse for StoreError {
    fn into_response(self) -> Response {
        let (status, title, detail) = match &self {
            StoreError::NotFound { key } => (
                StatusCode::NOT_FOUND,
                "Not Found",
                format!("The item '{}' was not found.", key),
            ),
            // ... handle other variants
            _ => (StatusCode::INTERNAL_SERVER_ERROR, "Error", "Internal Server Error".to_string()),
        };

        let body = ProblemDetails {
            r#type: "https://example.com/errors/not-found".to_string(), // In practice, map this based on the error variant
            title: title.to_string(),
            status: status.as_u16(),
            detail,
        };

        (status, Json(body)).into_response()
    }
}
```

## Builder pattern for APIs

For types with many optional fields or non-trivial invariants, a builder (Section 5 covers the mechanics) is the standard API surface. Recognized variants:

- **Owned builder** — `fn field(self, v) -> Self`. Most common; lets you chain.
- **Mutable builder** — `fn field(&mut self, v) -> &mut Self`. Useful when shared.
- **Type-state builder** — uses type parameters to make required fields compile-time-checked. Implemented by `typed-builder`, `bon`, `derive_builder`.

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

## Public API stability, semver, and deprecation

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

Use `#[non_exhaustive]` on public structs and enums to reserve the right to add fields/variants without a major bump.

### Deprecation Conventions

When phasing out an API (library or web handler), use Rust's built-in `#[deprecated]` attribute. It emits compiler warnings for downstream consumers.

✅ **Do** — Always include `since` and `note` (with migration instructions):

```rust
#[deprecated(
    since = "1.2.0",
    note = "Use `fetch_user_profile` instead, which handles pagination."
)]
pub fn get_user() { /* ... */ }
```

For Web APIs, deprecation should also be communicated via OpenAPI (e.g., `#[utoipa::path(deprecated)]`) and optionally through the `Deprecation` HTTP header.

## Trait design

- **Sealed traits** prevent downstream implementations while still being usable — the standard way to lock down an extension trait.
- Prefer **small, focused traits** over mega-traits; combine via supertrait bounds.
- `impl Trait` in argument position (`fn f(x: impl Into<String>)`) is sugar for a generic; in return position it hides the concrete type and is a stability win (you can change the returned type later).
- Document the contract — a trait is an API; its doc comment should explain the invariants implementers must uphold.

## Summary checklist

- [x] URL/resource naming conventions documented.
- [x] Payload field casing convention documented (and any divergence from internal language convention noted).
- [x] API versioning strategies documented with trade-offs.
- [x] Standard error response format for this ecosystem documented.
- [x] Contract-first tooling identified (OpenAPI, Protobuf, etc.).
- [x] Backward compatibility and deprecation conventions documented.
- [x] Public types implement `Debug`, `Clone` where sensible, `Send + Sync` where possible.
- [x] Conversions via `From`/`Into`/`AsRef`; accept `impl Into<...>`/`impl AsRef<...>` at boundaries.
- [x] Collections implement `IntoIterator` for `&T`, `&mut T`, `T`.
- [x] Data types derive `Serialize`/`Deserialize` with stable wire contracts.
- [x] Public errors are typed (not `anyhow::Error`).
- [x] `#[non_exhaustive]` on structs/enums that may grow.
- [x] Semver followed per the cargo semver reference; major bump on any breaking change.

## References

- Rust API Guidelines — `rust-lang.github.io/api-guidelines/`
- The Cargo Book — Semver compatibility — `doc.rust-lang.org/cargo/reference/semver.html`
- RFC 7807 (Problem Details for HTTP APIs) — `datatracker.ietf.org/doc/html/rfc7807`
- Utoipa (OpenAPI) — `github.com/juhaku/utoipa`
- Serde — `serde.rs/`
- The Rust Reference — Traits / impl Trait — `doc.rust-lang.org/reference/`
- *Effective Rust* — `effective-rust.com/`
