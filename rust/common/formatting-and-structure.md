---
title: Rust - Formatting and Structure Style
status: draft
version: 0.0.1
created: 2026-07-06
updated: 2026-07-07
---

# Rust — Formatting and Structure Style

Rust has an unusually strong consensus on formatting because the official team ships **`rustfmt`**, an opinionated formatter that implements the **Rust Style Guide**. The community norm is: *run `rustfmt`, accept its output, spend review cycles on substance instead of style.* This section catalogs the de facto rules, the configurable knobs, and the stable-vs-nightly split.

## Canonical sources

- **The Rust Style Guide** — `rust-lang.github.io/style-guide/`. The prose specification of how Rust code should look.
- **rustfmt** — `rust-lang.github.io/rustfmt/`. The tool that implements the Style Guide.
- **rustfmt configuration reference** — `rust-lang.github.io/rustfmt/` (each option has a visual example).

## The de facto rules (enforced by `cargo fmt`)

Unless a project deliberately overrides them, `rustfmt` produces:

- **Indentation:** 4 spaces, never tabs.
- **Line width:** 100 characters by default (`max_width = 100`).
- **Braces:** opening brace on the same line for items, functions, control flow (`fn`, `if`, `match`, `impl`), K&R style.
- **Trailing commas:** in multi-line constructs, `rustfmt` *adds* a trailing comma after the last item; in single-line constructs it removes it.
- **`match` arms:** `=>` followed by an expression on the same line when short, or a block; multi-line arm bodies are indented one level.
- **Imports:** reordered alphabetically within each group by default (`reorder_imports = true`).
- **`use` statements:** grouped; nested paths collapsed with braces.

✅ **Do** — let `rustfmt` own formatting:

```rust
pub fn parse_config(path: &Path) -> Result<Config, ConfigError> {
    let contents = std::fs::read_to_string(path)
        .map_err(|e| ConfigError::Read(path.to_path_buf(), e))?;
    let config: Config = toml::from_str(&contents)?;
    Ok(config)
}
```

❌ **Don't** — hand-format against `rustfmt`:

```rust
pub fn parse_config( path : &Path )
                      -> Result< Config, ConfigError >
{
let contents=std::fs::read_to_string(path)
.map_err(|e| ConfigError::Read(path.to_path_buf(),e))?;
let config:Config=toml::from_str(&contents)?;
Ok(config)
}
```

Running `cargo fmt` on the second snippet yields the first.

## `rustfmt.toml` — the configuration file

A `rustfmt.toml` (or `.rustfmt.toml`) at the crate/workspace root overrides defaults. **Stable options** work on any toolchain:

```toml
# Stable rustfmt.toml
max_width = 100            # default 100
hard_tabs = false          # default false (spaces)
tab_spaces = 4             # default 4
edition = "2021"
newline_style = "Unix"
use_field_init_shorthand = true
use_try_shorthand = true
```

### `use_field_init_shorthand` and `use_try_shorthand`

These two are commonly enabled and reduce noise:

```rust
// Field init shorthand ON (recommended)
struct Point { x: i32, y: i32 }
let x = 0;
let p = Point { x, y: 1 };        // ✅ shorthand where name == field

// Field init shorthand OFF
let p = Point { x: x, y: 1 };     // verbose
```

```rust
// try shorthand ON (recommended) — ? instead of try!()
let n: i32 = s.parse()?;          // ✅

// try shorthand OFF
let n: i32 = try!(s.parse());     // legacy macro form
```

## Stable vs. nightly `rustfmt`

The single most important caveat: **several popular options are unstable and only work on the nightly toolchain.** On stable, `cargo fmt` will reject an unstable config entry with *"unstable features are only available in nightly channel."* (rust-lang/rustfmt tracking issue #4991.)

Notable **nightly-only** options:

| Option | Effect | Notes |
|---|---|---|
| `imports_granularity` | Merge `use` statements at `Crate` / `Module` / `Item` / `One` level | Replaces the older `merge_imports`. |
| `group_imports` | Group imports as `StdExternalCrate` (std, then external, then local) | Replaced/renamed; check version. |
| `imports_layout` | How items within a `use` are laid out | Nightly. |
| `reorder_impl_items` | Reorder items within `impl` blocks | Nightly. |
| `condense_wildcard_suffixes` | Collapse trailing `..` in match arms | Nightly. |
| `format_code_in_doc_comments` | Run `rustfmt` inside doctests | Nightly. |

To enable nightly options:

```toml
# rustfmt.toml
unstable_features = true
edition = "2021"
imports_granularity = "Crate"
group_imports = "StdExternalCrate"
```

And run with the nightly toolchain:

```sh
rustup component add rustfmt --toolchain nightly
cargo +nightly fmt
```

> ⚠️ **Document the choice.** A project that relies on nightly `rustfmt` should pin the toolchain in CI (e.g. `rust-toolchain.toml`) and document it in the README, so contributors are not surprised by *"unstable option"* errors on stable. Some teams deliberately stay on stable-only options for portability; others accept the nightly dependency for nicer import grouping. Both are recognized positions.

### Import grouping — the two recognized styles

Because `group_imports` is nightly-only, two community conventions coexist:

**A. `StdExternalCrate` grouping** (nightly `rustfmt` or hand-edited):

```rust
use std::collections::HashMap;
use std::path::Path;

use anyhow::Context;
use serde::Deserialize;

use crate::config::Config;
use crate::error::AppError;
```

**B. Single alphabetical block** (default stable `rustfmt`):

```rust
use anyhow::Context;
use crate::config::Config;
use crate::error::AppError;
use serde::Deserialize;
use std::collections::HashMap;
use std::path::Path;
```

Both are acceptable; the project's `rustfmt.toml` and `rust-toolchain.toml` should make the choice explicit.

## Internal ordering of elements

While `rustfmt` handles whitespace and line-wrapping, it does not reorder the logical elements of a file or an `impl` block (other than optionally sorting `use` statements). The community convention for ordering items top-to-bottom is as follows:

### File-level ordering

1. **`#![...]`** inner attributes (e.g., `#![allow(dead_code)]`, `#![warn(missing_docs)]`).
2. **`mod` declarations** for child modules (`mod foo;`).
3. **`use` statements** (imports).
4. **`const` and `static` declarations**.
5. **`type` aliases**.
6. **`struct`, `enum`, and `union` definitions**.
7. **`trait` definitions**.
8. **`impl` blocks** (first inherent `impl T`, then trait implementations `impl Trait for T`).
9. **Free `fn` functions**.
10. **`mod tests`** (the inline test module, usually at the very bottom).

### `impl` block ordering

Within an inherent `impl` block for a type, group elements in this order:

1. **Associated `const` and `type` definitions**.
2. **Constructors** (`new`, `with_capacity`, `from_*`).
3. **Public methods** (`pub fn`).
4. **Private methods** (`fn` helper methods).

```rust
// File-level and impl-level ordering example
#![warn(missing_docs)]

use std::collections::HashMap;

const DEFAULT_CAPACITY: usize = 16;

pub struct Cache {
    data: HashMap<String, String>,
}

impl Cache {
    // 1. Associated constants
    pub const MAX_SIZE: usize = 1024;

    // 2. Constructors
    pub fn new() -> Self {
        Self {
            data: HashMap::with_capacity(DEFAULT_CAPACITY),
        }
    }

    // 3. Public methods
    pub fn get(&self, key: &str) -> Option<&String> {
        self.data.get(key)
    }

    // 4. Private methods
    fn clean_up(&mut self) {
        // ...
    }
}

// Tests at the bottom of the file
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_cache_new() {
        let cache = Cache::new();
        assert!(cache.data.capacity() >= 16);
    }
}
```

## Match arm formatting

```rust
// ✅ rustfmt default
match status {
    HttpStatus::Ok => write!(f, "200 OK"),
    HttpStatus::NotFound => write!(f, "404 Not Found"),
    HttpStatus::ServerError(code) => {
        log_error(code);
        write!(f, "500 Server Error")
    }
}
```

- Short arms stay on one line with `=>`.
- Block arms wrap, opening brace on the `=>` line.
- The wildcard `_` arm goes last.

## Where braces and spacing go (Style Guide)

- **Space before `{`** on functions, control flow, `impl`.
- **No space** between function name and `(`.
- **Space after** `,` and `;` (not before).
- **Space around** binary operators (`a + b`, not `a+b`).
- **No space** around `.` and `::` (method/field access, path).
- **No space** around unary `!`, `-`, `*`, `&` (when used as a prefix operator).
- **Comma after** the last field in a multi-line struct/array literal (rustfmt adds it).

```rust
let xs = [
    1,
    2,
    3,           // trailing comma added by rustfmt
];

let p = Point {
    x: 1,
    y: 2,
};
```

## Tooling

- **`cargo fmt`** — formats the whole workspace in place.
- **`cargo fmt -- --check`** — exits non-zero if any file would change; the form used in CI.
- **Editor integration** — `rust-analyzer` runs `rustfmt` on save; format-on-save is the community default.

A typical CI step:

```sh
cargo fmt --all -- --check
```

## Summary checklist

- [ ] `rustfmt.toml` (or `.rustfmt.toml`) present at the workspace root.
- [ ] `edition` set explicitly in `rustfmt.toml`.
- [ ] `cargo fmt --all -- --check` runs in CI.
- [ ] If nightly options are used, `rust-toolchain.toml` pins nightly and the README says so.
- [ ] No hand-formatted blocks that fight `rustfmt`.

## References

- The Rust Style Guide — `rust-lang.github.io/style-guide/`.
- rustfmt — `rust-lang.github.io/rustfmt/`.
- rustfmt configuration reference — `rust-lang.github.io/rustfmt/`.
- Tracking issue for unstable import options — `github.com/rust-lang/rustfmt/issues/4991`.
- rustfmt `Configurations.md` — `github.com/rust-lang/rustfmt/blob/main/Configurations.md`.
