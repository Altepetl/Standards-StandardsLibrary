---
title: Rust - Character Encoding and Unicode
status: draft
version: 0.0.1
created: 2026-07-06
updated: 2026-07-06
---

# Rust — Character Encoding and Unicode

Covers how Rust models text. **Rust is UTF-8 by default and at the type level** — this single design decision removes entire categories of bugs that plague other languages. The sharp edges that remain are: the `char`-vs-grapheme distinction, OS strings, and legacy-encoding conversion.

## 1. Rust strings are valid UTF-8

- `String` and `&str` are **sequences of bytes that are guaranteed to be valid UTF-8** at all times. The type system enforces this: any operation that would produce invalid UTF-8 panics or returns an error.
- Source files are UTF-8 by default; the compiler rejects files that are not valid UTF-8.
- A Rust **`char`** is a **Unicode scalar value** (a code point minus the surrogates: `U+0000`–`U+D7FF` and `U+E000`–`U+10FFFF`). It is **4 bytes** (`std::mem::size_of::<char>() == 4`), not 1 byte.
- A `&str` is a slice of bytes (`&[u8]` with a UTF-8 invariant). `s.len()` returns the **byte length**, not the character count.

```rust
let s = "héllo";
assert_eq!(s.len(), 6);                 // bytes: h(1) + é(2) + l(1) + l(1) + o(1)
assert_eq!(s.chars().count(), 5);       // scalar values
assert_eq!(s.as_bytes().len(), 6);      // raw bytes
```

## 2. The `char` vs. grapheme cluster distinction (the most common Rust bug)

A `char` is a **scalar value**, **not a user-perceived character** (a "grapheme cluster"). For text that contains combining marks, emoji, or ZWJ sequences, `chars().count()` over-counts.

### ✅ Count user-visible characters with `unicode-segmentation`

```rust
use unicode_segmentation::UnicodeSegmentation;

let family = "👨‍👩‍👧‍👦";          // family emoji: ZWJ sequence of 4 emoji + 3 joiners
assert_eq!(family.chars().count(), 7);          // 7 scalar values
assert_eq!(family.graphemes(true).count(), 1);  // 1 user-perceived character

let combined = "é";   // precomposed U+00E9
assert_eq!(combined.chars().count(), 1);

let decomposed = "e\u{0301}";   // 'e' + combining acute accent
assert_eq!(decomposed.chars().count(), 2);       // two scalar values!
assert_eq!(decomposed.graphemes(true).count(), 1);   // still one grapheme
```

[`unicode-segmentation`](https://crates.io/crates/unicode-segmentation) (rust-lang org) implements UAX #29 grapheme/word/sentence segmentation.

### ❌ Substring slicing by char index panics

```rust
let s = "héllo";
let _ = &s[1..3];          // ❌ PANICS: byte index 1 is not a char boundary
let _ = s.split_at(1);     // ❌ same problem
```

`&str` indexing by `[a..b]` is **byte-range** slicing. Use `char_indices()` or `graphemes(true)` to iterate to a safe boundary first:

### ✅ Safe iteration

```rust
// Iterate scalar values:
for c in "héllo".chars() { /* ... */ }

// Iterate (byte_offset, char) pairs — for byte-safe slicing:
for (i, c) in "héllo".char_indices() { /* ... */ }

// Iterate graphemes for user-visible boundaries:
for g in "héllo👨‍👩‍👧‍👦".graphemes(true) { /* ... */ }
```

## 3. OS strings and paths

The OS layer does not guarantee UTF-8:

- On Unix, file paths and environment strings are arbitrary bytes (`[u8]`).
- On Windows, they are potentially-invalid UTF-16 (`[u16]`).

Rust models this with dedicated types:

| Type | Holds |
|---|---|
| `OsStr` / `OsString` | Platform-native string (bytes on Unix, WTF-8 on Windows). Can hold non-UTF-8. |
| `Path` / `PathBuf` | A wrapper over `OsStr` for filesystem paths. Cross-platform. |
| `CString` / `CStr` | NUL-terminated bytes for FFI. |

```rust
use std::path::Path;

let p = Path::new("/tmp/日本語.txt");
println!("{}", p.display());             // lossy display
let s: &str = p.to_str().expect("non-UTF-8 path");   // None if invalid UTF-8
```

### ❌ Do not assume a path is UTF-8

```rust
fn bad(p: &Path) {
    let s: &str = p.to_str().unwrap();   // ❌ panics on Linux with non-UTF-8 path
}
```

### ✅ Handle the non-UTF-8 case explicitly

```rust
fn good(p: &Path) {
    match p.to_str() {
        Some(s) => println!("utf-8: {}", s),
        None => println!("non-utf8 path: {}", p.display()),
    }
}
```

## 4. Byte strings

The `b"..."` literal produces a `&[u8; N]`, not a `&str`:

```rust
let bytes: &[u8] = b"hello";           // 5 ASCII bytes
let crlf: &[u8] = b"\r\n";

// Byte string with raw bytes / hex:
let raw: &[u8] = &[0xFF, 0xFE, 0x00];
```

Use byte strings for binary protocols, file magic numbers, and any context where the data is bytes, not text. The [`bstr`](https://crates.io/crates/bstr) crate provides byte-oriented string operations (search, split, regex) that work on `&[u8]` directly without requiring UTF-8.

## 5. Decoding and encoding bytes ↔ strings

| Function | Behaviour |
|---|---|
| `String::from_utf8(Vec<u8>)` | `Result<String, FromUtf8Error>` — fails on invalid UTF-8. |
| `String::from_utf8_lossy(&[u8])` | `Cow<str>` — replaces invalid bytes with `U+FFFD`. Never fails. |
| `String::from_utf8_unchecked` | **`unsafe`** — caller must guarantee validity. Only for verified hot paths. |
| `str::from_utf8(&[u8])` | `Result<&str, Utf8Error>` — zero-copy borrow. |
| `str::as_bytes()` | The inverse — always free (a `&str` *is* a `&[u8]`). |

### ✅ Use `from_utf8_lossy` for untrusted input

```rust
let raw: &[u8] = &[0x68, 0x65, 0x6C, 0x6C, 0x6F, 0xFF];
let s: &str = std::str::from_utf8(raw).unwrap_or("invalid");
let s2: std::borrow::Cow<str> = String::from_utf8_lossy(raw);   // "hello\u{FFFD}"
```

### ❌ Do not use `unsafe from_utf8_unchecked` on untrusted data

```rust
// ❌ Undefined behaviour if `bytes` is not valid UTF-8:
let s: &str = unsafe { std::str::from_utf8_unchecked(bytes) };
```

## 6. Legacy encodings — `encoding_rs`

[`encoding_rs`](https://crates.io/crates/encoding_rs) is the de facto web standard for legacy character encodings (Shift_JIS, GB18030, ISO-8859-*, Windows-1252, …). It is what Firefox uses; it implements the [Encoding Standard](https://encoding.spec.whatwg.org/) exactly.

```rust
use encoding_rs::SHIFT_JIS;

let (cow, _, had_errors) = SHIFT_JIS.decode(b"\x82\xb1\x82\xf1\x82\xc9\x82\xbf\x82\xcd");
assert_eq!(&cow, "こんにちは");
```

- It is streaming-capable, allocation-free where possible, and the spec-correct way to decode web legacy content.
- For pure UTF-8 use `String::from_utf8_*`; reach for `encoding_rs` only when you must consume non-UTF-8 input.

## 7. Normalization — NFC / NFD / NFKC / NFKD

The **`unicode-normalization`** crate (rust-lang org) implements UAX #15.

```rust
use unicode_normalization::UnicodeNormalization;

let nfd: String = "é".nfd().collect();      // "e" + U+0301
let nfc: String = "e\u{0301}".nfc().collect();   // "é"
```

**Conventions:**

- **Store and compare identifiers (user names, lookup keys) in NFC** to avoid duplicate-looking-but-different records.
- Use NFD *before* decomposition-sensitive operations (e.g., some legacy collations).
- Never compare un-normalized strings for equality.

## 8. Collation and locale-aware sorting

Standard Rust comparisons (`<`, `>`, `==`, `sort()`) on strings compare them by their **byte values**. This is equivalent to comparing Unicode scalar values directly, which is generally incorrect for human-facing text.

For example, a byte-wise sort puts `"Zebra"` before `"apple"`, and it completely mis-sorts accented characters. Furthermore, sorting rules depend on the locale (e.g., in Swedish, `ä` sorts after `z`, but in German, `ä` sorts like `a`).

### ✅ Use ICU4X for locale-aware collation

For user-facing sorting, use the [`icu`](https://crates.io/crates/icu) crate (specifically `icu::collator`) from the [ICU4X](https://github.com/unicode-org/icu4x) project.

```rust
use icu::collator::{Collator, CollatorOptions};
use icu::locid::locale;

// Note: Error handling omitted for brevity
let collator = Collator::try_new(&locale!("sv").into(), CollatorOptions::new()).unwrap();
let mut words = vec!["äpple", "zebra", "banan"];

// Sort using Swedish rules (ä sorts after z)
words.sort_by(|a, b| collator.compare(a, b));
assert_eq!(words, vec!["banan", "zebra", "äpple"]);
```

**Conventions:**
- **Binary sort vs. Collation:** Use standard `sort()` when sorting internal identifiers or machine data. Use `icu::collator` when sorting lists presented to a human.
- **Normalization:** ICU4X collation handles normalization internally; you don't need to pre-normalize strings before passing them to the collator.

## 9. Case conversion is Unicode-aware

Rust's built-in `char::to_uppercase` / `to_lowercase` and `str::to_uppercase` / `to_lowercase` are **Unicode-aware** (they consult the Unicode Character Database). They are not ASCII-only.

```rust
assert_eq!("istanbul".to_uppercase(), "ISTANBUL");   // correct (Turkish-awareness depends on locale; the default is generic)
assert_eq!("ß".to_uppercase(), "SS");                 // ✓ Unicode case folding
```

> **Caveat:** Rust's `to_uppercase`/`to_lowercase` are *locale-insensitive*. Turkish I (`İ`/`ı`), Lithuanian dot-above, and similar locale-specific rules are **not** applied. For locale-aware casing, use ICU4X (`icu::casemap`). For most server-side code the locale-insensitive default is correct; the gotcha matters when casing user-visible text in Turkish, Lithuanian, or Azeri.

## 10. Regex and Unicode

The [`regex`](https://crates.io/crates/regex) crate (rust-lang org) is Unicode-aware by default:

- `.` matches a single **Unicode scalar value** (not a byte), and can be configured to match graphemes via the `regex` + `unicode-segmentation` combination or the newer `regex-automata`.
- `\p{L}`, `\p{Greek}`, `\p{Alphabetic}` — Unicode property escapes work out of the box.
- `(?u)` / `(?-u)` flags toggle Unicode mode; the default is on.

```rust
let re = regex::Regex::new(r"\p{Greek}+").unwrap();
assert!(re.is_match("Hello, Κόσμε"));
```

For grapheme-aware matching, consider the [`fancy-regex`](https://crates.io/crates/fancy-regex) crate (backtracking, supports look-around) or pre-segment with `unicode-segmentation`.

## 11. Pitfalls summary

| Pitfall | Mitigation |
|---|---|
| `&s[i..j]` panics on non-boundary. | Iterate `char_indices` or `graphemes(true)`. |
| `s.len()` reports bytes, not characters. | `s.chars().count()` for scalar values; `s.graphemes(true).count()` for user-perceived characters. |
| `s.chars().nth(i)` is O(n). | Convert to `Vec<char>` once, or use `char_indices`. |
| Comparing un-normalized strings. | `nfc()` both sides first. |
| `to_uppercase` ignores locale. | Use `icu::casemap` for Turkish/Lithuanian/Azeri. |
| Assuming OS paths are UTF-8. | Use `Path` / `OsStr`; handle `to_str() == None`. |
| Decoding untrusted bytes with `unsafe`. | Use `from_utf8_lossy` / `from_utf8`. |
| Sorting user-facing text with `sort()`. | Use `icu::collator` for locale-aware sorting. |

## Completion checklist

- [x] Default source/string encoding (UTF-8) and the `String`/`&str` model documented.
- [x] String model: bytes vs. scalar values (`char`) vs. grapheme clusters documented, with ✅/❌.
- [x] OS strings and `Path` documented.
- [x] Byte strings (`b"..."`, `bstr`) documented.
- [x] Decode/encode (`from_utf8`, `from_utf8_lossy`, `from_utf8_unchecked`) documented.
- [x] Legacy encodings (`encoding_rs`) documented.
- [x] Normalization (`unicode-normalization`) documented.
- [x] Collation and locale-aware sorting conventions documented.
- [x] Unicode-aware case conversion (and its locale caveat) documented.
- [x] Regex Unicode support documented.
- [x] Pitfalls and idiomatic mitigations documented.

### References

- The Rust Reference — Strings — https://doc.rust-lang.org/reference/type-layout.html
- `std::str` — https://doc.rust-lang.org/std/str/
- `unicode-segmentation` — https://docs.rs/unicode-segmentation
- `unicode-normalization` — https://docs.rs/unicode-normalization
- `encoding_rs` — https://docs.rs/encoding_rs
- WHATWG Encoding Standard — https://encoding.spec.whatwg.org/
- UAX #29 (text segmentation) — https://www.unicode.org/reports/tr29/
- UAX #15 (normalization) — https://www.unicode.org/reports/tr15/
