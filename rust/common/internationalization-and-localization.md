---
title: Rust - Internationalization and Localization
status: draft
version: 0.0.1
created: 2026-07-06
updated: 2026-07-06
---

# Rust — Internationalization and Localization

Covers how Rust supports adapting software for different languages, regions, and cultural conventions. **Applicability varies by project type:** high for web services, GUI, and any user-facing Rust; lower for systems libraries, embedded, and CLIs (where i18n is often just "externalize the strings, don't assume ASCII").

> **Catalog, not mandate.** The ecosystem has several recognized solutions. They are documented side by side; selection happens in StandardBuilder.

## 1. The two recognized solution families

There are two mainstream approaches in the Rust ecosystem:

| Family | Canonical crate | Origin | Style |
|---|---|---|---|
| **Fluent** (message catalog) | [`fluent`](https://crates.io/crates/fluent) + `fluent-bundle` | Mozilla (powers Firefox) | `.ftl` files; message *id* → translation with rich selectors |
| **ICU4X** (full CLDR/Unicode platform) | [`icu`](https://crates.io/crates/icu) | Unicode Consortium | Locale-aware formatting of dates/numbers/lists/collation; data from CLDR |
| **rust-i18n** (lightweight) | [`rust-i18n`](https://crates.io/crates/rust-i18n) | Community | Simple `t!("key")` macro over YAML/TOML/JSON catalogs |

These overlap in scope but not in ambition:

- **Fluent** is the canonical choice for *message externalization with full pluralization, gender, and grammar*. It is the most mature Rust i18n solution.
- **ICU4X** is the canonical choice for *locale-aware formatting* (dates, numbers, lists, collation, segmentation). It is not primarily a *translation catalog* system; pair it with Fluent for messages.
- **rust-i18n** is the pragmatic choice for projects that need basic `t!()` lookups without bringing in Fluent's machinery.

## 2. Fluent — the canonical message system

[Fluent](https://projectfluent.org/) is Mozilla's localization system, used by Firefox and Mozilla web properties. The Rust implementation is the reference implementation of Fluent.

### The `.ftl` file format

```ftl
# greetings.ftl
welcome =
    Welcome, { $user }!
items =
    { $count ->
        [0]   You have no items.
        [one] You have one item.
       *[other] You have { $count } items.
    }
last-login =
    Last login: { DATETIME($date, dateStyle: "medium") }
```

Features: variables (`{ $user }`), **selectors** for plurals/gender (`{ $count -> ... }`), built-in functions (`DATETIME`, `NUMBER`), attributes, and references between messages.

### Using Fluent from Rust

```rust
use fluent::{FluentBundle, FluentResource};
use unic_langid::LanguageIdentifier;

let ftl = r#"
    welcome = Welcome, { $user }!
    items = { $count ->
        [one] You have one item.
       *[other] You have { $count } items.
    }
"#.to_string();
let res = FluentResource::try_new(ftl).expect("parse failed");

let lang: LanguageIdentifier = "en".parse().unwrap();
let mut bundle = FluentBundle::new(vec![lang]);
bundle.add_resource(res).unwrap();

let mut errors = vec![];
let msg = bundle.get_message("welcome").unwrap();
let pattern = msg.value().unwrap();
let value = bundle.format_pattern(pattern, &[("user", "Rommel".into())], &mut errors);
assert_eq!(value, "Welcome, Rommel!");
```

The ecosystem crates around Fluent:

| Crate | Role |
|---|---|
| [`fluent`](https://crates.io/crates/fluent) | The core bundle + formatter. |
| [`fluent-bundle`](https://crates.io/crates/fluent-bundle) | The lower-level API (often re-exported by `fluent`). |
| [`fluent-syntax`](https://crates.io/crates/fluent-syntax) | The `.ftl` parser, if you want to AST-process translations. |
| [`fluent-resmgr`](https://crates.io/crates/fluent-resmgr) | Resource manager that loads `.ftl` files per locale. |
| [`fluent-templates`](https://crates.io/crates/fluent-templates) | Macros (`t!`, `ftl!`) for ergonomic compile-time-embedded `.ftl`. |
| [`i18n-embed`](https://crates.io/crates/i18n-embed) + [`i18n-embed-fluent`](https://crates.io/crates/i18n-embed-fluent) | Embedding + runtime loader for desktop/WASM apps. |

## 3. ICU4X — the Unicode Consortium platform

[ICU4X](https://github.com/unicode-org/icu4x) is the Unicode Consortium's rewrite of ICU, written in Rust with FFI bindings for C/C++/JS. **2.0 was released in May 2025.** It is used by Firefox and is the long-term home for CLDR-backed formatting in Rust.

```rust
use icu::datetime::{DateTimeFormatter, fieldsets::YMD};
use icu::locale::locale;

let dt = icu::calendar::Date::try_new_iso_date(2026, 7, 6).unwrap();
let fmt = DateTimeFormatter::try_new(locale!("es-MX").into(), YMD::medium()).unwrap();
let s = fmt.format_any_calendar(&dt).unwrap();
// "6 jul 2026" (Spanish (Mexico), medium length)
```

ICU4X is the right choice when you need:

- Locale-aware date / number / list / unit / duration formatting.
- Collation (locale-aware sorting).
- Unicode segmentation (graphemes, words, sentences).
- Plural rules (the `icu_plurals` crate, used by Fluent under the hood).

It is **not** a translation-catalog system by itself; pair ICU4X with Fluent.

## 4. Locale identifiers and negotiation

- **`unic-langid`** ([`LanguageIdentifier`](https://crates.io/crates/unic-langid)) is the legacy Rust crate for BCP 47 language identifiers. Still used by Fluent 0.x.
- **`icu::locale`** (the `Locale` type from ICU4X 2.0) is the modern replacement: full BCP 47 with extensions, Unicode locale extensions (`-u-`), and locale *negotiation* (matching a requested locale against the available ones, with fallback chains).

```rust
use icu::locale::locale;
use icu::locid_transform::{LocaleExpander, LocaleDirection};

let expander = LocaleExpander::new();
let mut loc = locale!("es-MX");
expander.maximize(&mut loc);   // → "es-Latn-MX"
```

## 5. RTL and bidirectional text

- The **`unicode-bidi`** crate (rust-lang org) implements the Unicode Bidirectional Algorithm (UAX #9). Use it when you render mixed-direction text (e.g., in a custom UI).
- **`unicode-bidi-sqlite`** style usage is rare; the common case is a UI toolkit that already calls `unicode-bidi` internally.
- ICU4X provides `LocaleDirection` to detect RTL locales (`ar`, `he`, `fa`, `ur`) for layout direction.

```rust
use icu::locid_transform::LocaleDirection;
use icu::locale::locale;

let dir = LocaleDirection::from_locale(&locale!("ar-EG"));
assert_eq!(dir, LocaleDirection::RightToLeft);
```

For web UIs built in Rust (Leptos, Dioxus, Yew), RTL handling is the browser's job at the DOM/CSS layer; the Rust side only needs to set `dir="rtl"` based on the active locale.

## 6. Pluralization: Fluent vs. ICU MessageFormat

Rust has two paths to CLDR-correct pluralization:

| Approach | Example |
|---|---|
| **Fluent selectors** (recommended in Rust) | `items = { $count -> [one] one item *[other] { $count } items }` |
| **`icu_plurals` directly** | For when you only need the plural category (`One`/`Few`/`Many`/`Other`) without a message catalog. |

### ❌ String concatenation fails plurals

```rust
// ❌ Wrong for languages with complex plurals (Russian, Arabic, Polish):
fn items_msg(count: usize) -> String {
    format!("You have {} item{}", count, if count == 1 { "" } else { "s" })
}
```

### ✅ Fluent selector

```ftl
items = { $count ->
    [one] You have one item.
   *[other] You have { $count } items.
}
```

## 7. rust-i18n — the lightweight alternative

[`rust-i18n`](https://crates.io/crates/rust-i18n) provides a `t!()` macro over simple YAML/TOML/JSON catalogs. It is popular for server-side apps that need basic message lookup without Fluent.

```yaml
# locales/en.yml
en:
  welcome: "Welcome, %{name}!"
  items:
    one: "You have one item."
    other: "You have %{count} items."
```

```rust
rust_i18n::i18n!("locales");   // macro at crate root, embeds all catalogs

fn main() {
    rust_i18n::set_locale("es");
    println!("{}", t!("welcome", name = "Rommel"));
}
```

**Trade-off:** simpler ergonomics and YAML/JSON catalogs familiar to translators, but pluralization is more limited than Fluent and locale-aware formatting must be done separately.

## 8. Gettext compatibility

[`gettext`](https://crates.io/crates/gettext) and [`gettext-rs`](https://crates.io/crates/gettext-rs) expose the GNU gettext workflow (`.po`/`.pot`/`.mo`). This is the **legacy** path and is mostly used to port existing C gettext-based applications. For new code, Fluent or rust-i18n are more idiomatic.

## 9. Translation workflow conventions

Regardless of which crate you pick:

- **Externalize every user-facing string.** Never hard-code messages in source.
- **Use stable message IDs**, not English text, as keys (Fluent enforces this; rust-i18n permits either).
- **Run pseudo-localization** (`[Кэömэ, Römmэl!]`) in CI to catch hard-coded strings and layout breakage.
- **Define a locale fallback chain** and test it (`es-MX` → `es` → `en`).
- **Ship CLDR data** only for the locales you actually support, to keep binary size down (ICU4X supports this via its `compiled_data` features).

## Completion checklist

- [x] Canonical i18n libraries documented side by side (Fluent, ICU4X, rust-i18n, gettext).
- [x] Message externalization conventions and `.ftl` format documented.
- [x] Pluralization (Fluent selectors vs. `icu_plurals`) documented.
- [x] Locale-aware formatting cross-referenced (Section 20).
- [x] RTL / bidirectional support (`unicode-bidi`, `LocaleDirection`) documented.
- [x] Locale negotiation and fallback documented.
- [x] Translation workflow conventions documented.

### References

- Project Fluent — https://projectfluent.org/
- Fluent Rust docs — https://docs.rs/fluent
- ICU4X project — https://github.com/unicode-org/icu4x
- ICU4X 2.0 release — https://blog.unicode.org/2025/05/icu4x-20-released.html
- The `icu` meta-crate — https://docs.rs/icu
- rust-i18n — https://docs.rs/rust-i18n
- `unicode-bidi` — https://docs.rs/unicode-bidi
- BCP 47 — https://www.rfc-editor.org/rfc/rfc5646
