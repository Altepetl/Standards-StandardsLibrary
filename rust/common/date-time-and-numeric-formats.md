---
title: Rust - Date, Time, Time Zones, and Numeric Formats
status: draft
version: 0.0.1
created: 2026-07-06
updated: 2026-07-06
---

# Rust — Date, Time, Time Zones, and Numeric Formats

Covers how Rust handles instants, wall-clock time, durations, time zones, and the formatting/parsing of dates and numbers. Mishandled time is a frequent source of bugs; this section documents the idioms the ecosystem has settled on.

## 1. What is in `std::time`

`std::time` ships only two time types and a duration:

| Type | Meaning |
|---|---|
| `std::time::Instant` | A **monotonic** timestamp suitable for measuring elapsed time. Not wall-clock. Never goes backwards. Use for benchmarks and timeouts. |
| `std::time::SystemTime` | A **wall-clock** instant, anchored to the Unix epoch. Can go backwards (NTP adjustments, user changes clock). Use for "what time is it now". |
| `std::time::Duration` | A span of time (`new(secs, nanos)`). Returned by `Instant::elapsed()` and `Instant::duration_since()`. |

### ✅ Use `Instant` for elapsed time

```rust
use std::time::Instant;

let start = Instant::now();
do_expensive_work();
let elapsed = start.elapsed();
println!("took {:?}", elapsed);   // e.g. "took 12.345ms" (Debug formats durations nicely)
```

### ❌ Do not use `SystemTime` for elapsed time

```rust
// ❌ Wall clock can jump backwards (NTP, user, DST):
let start = std::time::SystemTime::now();
do_expensive_work();
let elapsed = start.elapsed().unwrap();   // can panic or be wrong
```

`std::time` deliberately has **no calendar, no date, and no human format**. For dates, years, months, time zones, and formatting, you need a crate.

## 2. `chrono` — the de facto standard

[`chrono`](https://crates.io/crates/chrono) is the long-standing de facto standard for date and time in Rust. It is feature-complete and battle-tested. Core types:

| Type | Meaning |
|---|---|
| `DateTime<Utc>` | A point in time, UTC. The default for storage. |
| `DateTime<FixedOffset>` | A point in time with a fixed UTC offset (e.g. `+02:00`). |
| `DateTime<Tz>` (e.g. `DateTime<chrono_tz::Tz>`) | A point in time with a named IANA time zone (with DST handling). |
| `NaiveDateTime` | A wall-clock date+time **without** a timezone. For parsing/display only; never store or compare across zones. |
| `NaiveDate`, `NaiveTime` | Calendar date / time of day, no zone. |
| `chrono::Duration` (alias of `TimeDelta`) | A span of time that can be negative. |

### ✅ Store and transmit in UTC, convert at the boundary

```rust
use chrono::{DateTime, Utc};

let now: DateTime<Utc> = Utc::now();                       // store this
let iso8601 = now.to_rfc3339();                            // "2026-07-06T14:30:00.123+00:00"
let parsed: DateTime<Utc> = DateTime::parse_from_rfc3339(&iso8601)?.with_timezone(&Utc);
```

### ❌ Do not store naive datetimes when a zone is involved

```rust
use chrono::NaiveDateTime;

// ❌ Ambiguous — is this UTC, local, or PST?
let s = "2026-03-15T14:00:00";
let dt = NaiveDateTime::parse_from_str(s, "%Y-%m-%dT%H:%M:%S").unwrap();
```

`chrono` is the dominant choice; it has no official governance home but is widely trusted (used by serde, sqlx, reqwest, etc.).

## 3. `time` — the Rust-platform effort

[`time`](https://crates.io/crates/time) is the newer crate maintained under the `rust-lang` org and is the official "platform" effort to replace `chrono`. It has a cleaner API, no panicking parsing, and is fully `#[no_std]`-compatible.

```rust
use time::{OffsetDateTime, macros::datetime};

let now: OffsetDateTime = OffsetDateTime::now_utc();
let dt = datetime!(2026-07-06 14:30:00 UTC);   // compile-time-checked literal
let formatted = now.format(&time::format_description::well_known::Rfc3339)?;
```

**`chrono` vs. `time`:** both are excellent and widely used. `chrono` has the larger ecosystem footprint (more downstream crates depend on it); `time` has cleaner internals and the institutional backing of the libs team. Pick whichever your dependency graph already uses; do not introduce both. (For a new greenfield project with no other constraints, `time` is the more future-proof choice.)

## 4. Time zones: `chrono-tz` and the IANA database

- [`chrono-tz`](https://crates.io/crates/chrono-tz) bundles the IANA tz database and provides `Tz` values for every zone (`chrono_tz::America::Mexico_City`, `chrono_tz::Europe::Berlin`, etc.).
- [`tzfile`](https://crates.io/crates/tzfile) reads the system's tzdata at runtime instead of bundling it.
- `time` does not ship a tz database; combine it with [`time-tz`](https://crates.io/crates/time-tz) for named zones.

### ✅ Convert an instant to a user's zone only for display

```rust
use chrono::{TimeZone, Utc};
use chrono_tz::America::Mexico_City;

let now = Utc::now();
let local = now.with_timezone(&Mexico_City);
println!("local: {}", local);                 // display
// store/transmit `now`, not `local`
```

## 5. Parsing and formatting

| Convention | Crate |
|---|---|
| **RFC 3339 / ISO 8601** for wire/storage | `chrono::DateTime::parse_from_rfc3339`, `to_rfc3339`; `time`'s `Rfc3339` format |
| `strftime`-style format strings | `chrono::format::strftime` (`%Y-%m-%d %H:%M:%S`); `time::format_description` (a different, compile-time-checkable mini-language) |
| Human durations ("2d 3h") | [`humantime`](https://crates.io/crates/humantime) — parses and formats `Duration` |
| Locale-aware formatting | the `icu_datetime` crate from ICU4X (cross-reference Section 21) |

```rust
use chrono::NaiveDate;

let date = NaiveDate::parse_from_str("2026-07-06", "%Y-%m-%d")?;
let pretty = date.format("%A, %B %-d, %Y").to_string();   // "Monday, July 6, 2026"
```

## 6. Pitfalls and idiomatic mitigations

| Pitfall | Mitigation |
|---|---|
| Using `SystemTime` for elapsed time (non-monotonic). | Use `Instant`. |
| Storing `NaiveDateTime` across zones. | Store `DateTime<Utc>` / RFC 3339. |
| Hand-rolling timestamp arithmetic. | Use `Duration`; `DateTime::checked_add_signed`. |
| Assuming a year has 365 days / a day has 24h. | Use `Duration::days` only for *calendar-display*; for monotonic spans, use seconds. DST transitions make "tomorrow same time" ambiguous — use the tz-aware API. |
| Trusting the client clock. | Server timestamps from `Utc::now()`; reject client-sent timestamps unless you can verify. |
| Mixing `chrono` and `time` in the same project. | Pick one; convert at the boundary (`time::OffsetDateTime` ↔ `chrono::DateTime<Utc>` via epoch seconds). |

## 7. Numeric formatting

### `format!`, `{}`, `{:?}`

Rust's built-in formatting (`std::fmt`) covers most needs:

```rust
let n = 3.14159265;
println!("{}", n);        // 3.14159265
println!("{:.2}", n);     // 3.14         (precision)
println!("{:0>5}", 42);   // 00042        (padding)
println!("{:>8}", "hi");  //       hi     (alignment)
println!("{:e}", n);      // 3.14159265e0 (scientific)
println!("{:?}", n);      // 3.14159265   (Debug)
println!("{:#?}", vec![1,2,3]);   // pretty-printed Debug
```

There is no built-in thousands separator. For that, the ecosystem provides:

| Crate | Use case |
|---|---|
| [`num-format`](https://crates.io/crates/num-format) | Locale-aware grouping (`1,234,567` or `1.234.567`). |
| [`icu_decimal`](https://crates.io/crates/icu_decimal) | ICU4X decimal formatting (locale-aware, CLDR data). |
| [`printf_wrap`](https://crates.io/crates/printf_wrap) | FFI to C's `printf` for legacy format strings — niche. |

### ✅ Locale-aware number formatting

```rust
use num_format::{Locale, ToFormattedString};

assert_eq!(1_234_567u64.to_formatted_string(&Locale::en), "1,234,567");
assert_eq!(1_234_567u64.to_formatted_string(&Locale::de), "1.234.567");
```

### ❌ Hand-rolled grouping

```rust
// ❌ Reinvents locale rules; wrong for non-English locales:
fn thousands(s: String) -> String { /* ... */ }
```

## 8. Monetary values

Rust has no `Decimal` type in `std`. The community conventions:

- **`rust_decimal`** — fixed-point decimal (96-bit), the de facto choice for money.
- **`bigdecimal`** — arbitrary-precision, when you need more than 96 bits.
- Never store money as `f64` (cross-reference Section 9).

```rust
use rust_decimal::Decimal;
use rust_decimal_macros::dec;

let price: Decimal = dec!(19.99);
let total = price * dec!(3);      // exact, no float error
```

## Completion checklist

- [x] `std::time` (`Instant`, `SystemTime`, `Duration`) documented.
- [x] `chrono` (de facto standard) documented with core types.
- [x] `time` (Rust-platform crate) documented side by side.
- [x] Time-zone handling (`chrono-tz`) documented.
- [x] UTC-storage / RFC 3339 boundary convention documented.
- [x] Pitfalls and mitigations documented.
- [x] Numeric formatting (`format!`, `num-format`, `icu_decimal`) documented.
- [x] Monetary-value convention (`rust_decimal`) documented.

### References

- `std::time` docs — https://doc.rust-lang.org/std/time/
- `chrono` docs — https://docs.rs/chrono
- `time` docs — https://time-rs.github.io/book/
- `chrono-tz` — https://docs.rs/chrono-tz
- `humantime` — https://docs.rs/humantime
- `num-format` — https://docs.rs/num-format
- RFC 3339 — https://www.rfc-editor.org/rfc/rfc3339
