---
title: Rust - Security and Privacy
status: draft
version: 0.0.1
created: 2026-07-06
updated: 2026-07-06
---

# Rust — Security and Privacy

Rust's headline security property is **memory safety without a garbage collector**, enforced statically by the borrow checker for all *safe* code. This section catalogs what that buys you, where it stops buying you anything (`unsafe`), and the supply-chain, secrets, and cryptography conventions that round out a security posture. The OWASP concepts (injection, broken access control, cryptographic failures, vulnerable dependencies, security misconfiguration) translate naturally; this section maps each to its Rust-specific mitigation.

## Memory safety and why it matters

A large majority of CVEs in C/C++ codebases (industry studies and Microsoft's own reporting put the figure near **70%**) are memory-safety bugs: buffer overflows, use-after-free, double-free, type confusion, uninitialized reads, data races. Safe Rust makes these **impossible to express** — the borrow checker rejects them at compile time, with no runtime cost.

What "safe Rust" rules out statically:

- Buffer overruns and out-of-bounds indexing (slices carry their length; `xs[i]` panics rather than overruns; raw pointer arithmetic requires `unsafe`).
- Use-after-free and double-free (ownership is single; drop runs once).
- Null dereferences (there is no null; absence is `Option<T>`).
- Data races (`Send`/`Sync` plus the aliasing rule make them compile errors).
- Uninitialized memory (`MaybeUninit` is the explicit, `unsafe` escape hatch).

This is the single biggest reason projects adopt Rust for security-critical components (parsers, network endpoints, sandbox boundaries, cryptographic primitives). It does **not** make Rust code immune to the rest of the OWASP Top 10.

## `unsafe` Rust — where safety stops

`unsafe` (Section 5 lists the five superpowers) is the escape hatch for FFI, raw memory manipulation, and performance-critical intrinsics. It is **not** a free pass — every `unsafe` block is a place where the compiler's guarantees have been handed back to the programmer, who must prove soundness.

Community norms:

- Every `unsafe` block carries a `// SAFETY:` comment explaining the proof (now enforced by Clippy `clippy::undocumented_unsafe_blocks`).
- `unsafe` should be **encapsulated** in a small module or crate that exposes a *safe* API; the rest of the codebase stays in safe Rust.
- Prefer `unsafe`'s narrow, reviewed form over scattering it through business logic.

```rust
// ✅ Encapsulated, documented unsafe behind a safe API.
pub fn split_at_mut(xs: &mut [i32], mid: usize) -> (&mut [i32], &mut [i32]) {
    let len = xs.len();
    assert!(mid <= len);
    // SAFETY: mid <= len, so the two halves are disjoint and cover xs exactly.
    let p = xs.as_mut_ptr();
    unsafe {
        (
            std::slice::from_raw_parts_mut(p, mid),
            std::slice::from_raw_parts_mut(p.add(mid), len - mid),
        )
    }
}
```

❌ **Don't** — `unsafe` used to silence the borrow checker without a proof:

```rust
let r;
{
    let x = 42;
    r = unsafe { &*(&x as *const i32) };   // ❌ dangling reference — UB after the block
}
println!("{}", r);
```

### Auditing `unsafe`

Tooling (all documented side by side; none is a single winner):

- **`cargo-audit`** — audits `Cargo.lock` against the RustSec advisory database (see below). The original vulnerability scanner; its primary maintainer has publicly discussed stepping back, which is why `cargo-deny` is increasingly the recommended frontend.
- **`cargo-deny`** (Embark Studios) — superset: advisory checks **plus** license/ban/source-of-deps checks; runs in CI as `cargo deny check`. The RustSec team has discussed making `cargo-deny` the official frontend (GitHub issue EmbarkStudios/cargo-deny#194).
- **`cargo-geiger`** — counts `unsafe` blocks in your dependency tree; useful to spot crates that pull in surprising amounts of unsafe.
- **MIRI** — an interpreter for Rust's mid-level IR that catches Undefined Behavior in `unsafe` code (use after free, invalid pointer use, out-of-bounds, data races). Runs your tests under MIRI: `cargo +nightly miri test`.
- **`cargo-fuzz`** — libFuzzer-based fuzzing for input parsers.

### Supply chain — the RustSec advisory database

**RustSec** (`rustsec.org`) is the ecosystem's CVE/advisory database. `cargo-audit` and `cargo-deny check advisories` read it and fail CI when a dependency has a known advisory (identified by `RUSTSEC-YYYY-NNNN`).

```sh
# Install once:
cargo install cargo-audit
# Or use cargo-deny, which also covers licenses/sources:
cargo install cargo-deny && cargo deny init
```

A `deny.toml` for `cargo-deny` typically pins:

```toml
[advisories]
# Treat unmaintained crates as warnings, vulnerabilities as errors.
vulnerability = "deny"
unmaintained  = "warn"
yanked        = "deny"

[licenses]
# Fail on copyleft / unapproved licenses.
allow = ["MIT", "Apache-2.0", "BSD-3-Clause", "ISC", "Unicode-DFS-2016"]
confidence-threshold = 0.93

[bans]
# Optionally forbid specific crates, duplicates, etc.
multiple-versions = "warn"

[sources]
# Only allow crates from crates.io, not arbitrary git registries.
unknown-registry = "deny"
unknown-git      = "deny"
allow-registry   = ["https://github.com/rust-lang/crates.io-index"]
```

This is the Rust analogue of OWASP **A06:2021 Vulnerable and Outdated Components** and **A08:2021 Software and Data Integrity Failures**.

### Dependency provenance

- Pin the toolchain with `rust-toolchain.toml` and dependencies in `Cargo.lock` so builds are reproducible.
- Prefer crates from `crates.io` with active maintenance and a clear owner set; review new transitive dependencies before bumping.
- `cargo-deny`'s `[sources]` block can forbid pulling from arbitrary git URLs.
- For high-assurance code, vendoring (`cargo vendor`) plus a reproducible build lets you audit the exact code that ships.

## Secrets handling

Three orthogonal concerns:

### 1. Do not log secrets

The simplest and most important rule: never log tokens, passwords, API keys, PII, or session identifiers at any log level. Treat every `info!`/`debug!`/`tracing::` call as if it will end up in an aggregator — because it usually will.

✅ **Do** — log identifiers, not values:

```rust
tracing::info!(user_id = %session.user_id, "authenticated");
// never: tracing::info!(token = %session.bearer_token, ...);
```

### 2. Wrap secrets so they cannot be displayed accidentally — `secrecy`

The `secrecy` crate (used by many Rust crypto/web stacks) wraps a secret in a `Secret<T>` that does **not** implement `Display`/`Debug` (it implements `Debug` as `Secret([REDACTED])`). To use the value you must call `.expose_secret()`, making accidental logging a visible, reviewable event.

```rust
use secrecy::{Secret, ExposeSecret};

pub struct Config {
    pub api_token: Secret<String>,     // not Debug-printable as the value
}

fn call_api(cfg: &Config) {
    let token: &str = cfg.api_token.expose_secret();   // explicit, auditable
    /* ... */
}
```

### 3. Clear sensitive memory after use — `zeroize`

Memory that held a key, password, or secret should be overwritten before it is dropped, so it does not linger in freed heap memory or in a core dump. The `zeroize` crate does this in a way the compiler cannot optimize away.

```rust
use zeroize::Zeroize;

let mut key = [0u8; 32];
fill_key(&mut key);
use_key(&key);
key.zeroize();              // overwrite before going out of scope
```

`Zeroize` can be derived on structs whose fields are themselves zeroizable, and `ZeroizeOnDrop` runs the wipe on drop automatically.

## Cryptography — never hand-roll it

**Do not implement your own cryptography.** This is the same rule in every language, and Rust is no exception. Use audited crates:

| Need | Crate | Notes |
|---|---|---|
| Hashing (SHA-2, SHA-3, BLAKE2/3) | `sha2`, `sha3`, `blake2`, `blake3` | RustCrypto family. |
| Symmetric (AES, ChaCha20) | `aes`, `chacha20poly1305` | AEAD modes preferred. |
| Public-key (RSA, Ed25519, X25519) | `rsa`, `ed25519-dalek`, `x25519-dalek` | |
| TLS | `rustls` | Pure-Rust TLS; preferred over native OpenSSL bindings for new code. |
| Password hashing | `argon2`, `bcrypt`, `scrypt` | Never MD5/SHA for passwords. |
| Randomness | `rand` (with `getrandom` backend) | Use `OsRng` / `thread_rng()` for keys. |
| HMAC, constant-time ops | `subtle`, `hmac` | See below. |

### Constant-time comparisons — `subtle`

Any comparison that gates on a secret (a MAC tag, an auth token) must run in **constant time**, otherwise timing leaks the secret byte-by-byte. Never use `==` to compare MACs or tokens.

✅ **Do:**

```rust
use subtle::ConstantTimeEq;

fn verify_mac(expected: &[u8], provided: &[u8]) -> bool {
    bool::from(expected.ct_eq(provided))
}
```

❌ **Don't:**

```rust
fn verify_mac(expected: &[u8], provided: &[u8]) -> bool {
    expected == provided           // ❌ early-return timing leak
}
```

## Input validation

Rust's strong types do a lot of validation work for you (an `enum` cannot hold an out-of-range status; a `u16` cannot hold a negative port). For values that come from outside the trust boundary, validate explicitly:

✅ **Do** — parse, don't validate (the "Parse, don't validate" idiom):

```rust
pub struct Port(u16);

impl Port {
    pub fn parse(n: u16) -> Option<Port> {
        // Reserved or otherwise invalid ranges rejected at the boundary.
        (1024..=65535).contains(&n).then_some(Port(n))
    }
    pub fn value(&self) -> u16 { self.0 }
}

// Once you have a Port, the rest of the code does not re-check.
fn dial(p: Port) { /* uses p.value() */ }
```

The **newtype + fallible constructor** pattern (Section 8) means validation happens once, at the boundary, and the rest of the program trusts the type. This maps to OWASP **A03:2021 Injection** (typed inputs cannot carry injection payloads) and **A01:2021 Broken Access Control** (validated domain types prevent confused-deputy bugs).

For untrusted input that becomes SQL/HTML/shell, use parameterized APIs:

- SQL: `sqlx`/`diesel`/`sea-orm` queries with bound parameters — never `format!` strings into SQL.
- HTML: `askama`/`maud` with contextual escaping, or `ammonia` for sanitizing untrusted HTML.
- Shell: avoid `std::process::Command::new("sh").arg("-c")` with interpolation; pass `argv` directly.

## Cross-references to OWASP Top 10 (2021)

| OWASP category | Rust-specific mitigation in this section |
|---|---|
| A01 Broken Access Control | Typed domain objects (newtype + fallible parse); explicit auth checks. |
| A02 Cryptographic Failures | Audited crypto crates; constant-time `subtle`; `zeroize`. |
| A03 Injection | Typed inputs; parameterized SQL/HTML; `ammonia`. |
| A04 Insecure Design | The API Guidelines + error-type discipline (Section 8). |
| A05 Security Misconfiguration | `cargo-deny` `[sources]`; pinned toolchain; reviewed features. |
| A06 Vulnerable & Outdated Components | `cargo-audit` / `cargo-deny` against RustSec. |
| A07 Identification & Auth Failures | Constant-time comparison; no token logging; `secrecy`. |
| A08 Software & Data Integrity Failures | `Cargo.lock` committed; reproducible builds; vendoring for high assurance. |
| A09 Logging & Monitoring Failures | `tracing` structured logs; secrets redaction (Section 7). |
| A10 SSRF | Validate URLs against an allowlist; restrict outbound hosts. |

## Summary checklist

- [ ] All `unsafe` blocks encapsulated, with `// SAFETY:` proofs; MIRI run on `unsafe`-bearing tests.
- [ ] `cargo-deny` (or `cargo-audit`) runs in CI against RustSec; `deny.toml` pins licenses/sources.
- [ ] No secrets logged; secret-bearing fields wrapped in `secrecy::Secret`.
- [ ] Sensitive memory cleared with `zeroize`.
- [ ] Crypto from audited crates; constant-time comparisons via `subtle`.
- [ ] Untrusted input parsed at the boundary into typed domain objects.
- [ ] Parameterized SQL/HTML/shell; never interpolated strings.

## References

- RustSec Advisory Database — `rustsec.org/`.
- `cargo-audit` — `crates.io/crates/cargo-audit`.
- `cargo-deny` — `github.com/EmbarkStudios/cargo-deny`.
- `cargo-geiger` — `github.com/rust-secure-code/cargo-geiger`.
- MIRI — `github.com/rust-lang/miri`.
- The RustSecure Code guidelines — `rust-secure-code.github.io/security/`.
- The Rustonomicon — `doc.rust-lang.org/nomicon/` (`unsafe` obligations).
- `secrecy` — `docs.rs/secrecy`.
- `zeroize` — `docs.rs/zeroize`.
- `subtle` — `docs.rs/subtle`.
- RustCrypto — `github.com/RustCrypto`.
- OWASP Top 10 — `owasp.org/Top10/`.
