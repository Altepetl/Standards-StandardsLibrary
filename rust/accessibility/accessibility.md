---
title: Rust - Accessibility
status: draft
version: 0.0.1
created: 2026-07-06
updated: 2026-07-06
---

# Rust — Accessibility (a11y)

> **Scope note — applicability is limited.** Rust's dominant use cases — systems programming, command-line tools, embedded firmware, web servers, WebAssembly runtime components, and infrastructure libraries — produce **no direct user interface** for which WCAG/ARIA accessibility standards apply. For these projects, this section is **not applicable**, and a project standard can mark it as such with no further content.
>
> This document covers the cases where accessibility **does** apply to Rust: GUI applications, terminal UIs (TUIs), and web frontends built in Rust.

## 1. When this section applies

| Project type | Applicable? | Why |
|---|---|---|
| Library / crate | ❌ No | No user interface. |
| CLI (plain stdout/stderr) | ⚠️ Limited | Applies only to terminal-output ergonomics (color contrast, screen-reader-friendly output). See §3. |
| Terminal UI (TUI) | ✅ Yes | ratatui / crossterm apps. See §4. |
| Desktop GUI | ✅ Yes | egui, iced, Slint, Tauri, Dioxus, Xilem. See §5. |
| Web frontend | ✅ Yes | Leptos, Dioxus, Yew. WCAG/ARIA apply at the DOM layer. See §6. |
| Embedded | ❌ No | No UI in the WCAG sense. |
| Server / API | ❌ No | Indirect; the accessibility obligation moves to whatever consumes the API. |

If your project is in the "❌ No" rows, write a one-line standard: *"Not applicable — Rust crate/server with no direct UI; accessibility is delegated to the consumer."* and stop here.

## 2. Authoritative standards (when applicable)

| Standard | Scope |
|---|---|
| **[WCAG 2.2](https://www.w3.org/TR/WCAG22/)** (A / AA / AAA) | Web content; applies to Rust-compiled-to-WASM frontends. |
| **[WAI-ARIA 1.2](https://www.w3.org/TR/wai-aria-1.2/)** | Roles, states, properties for rich web apps; applies at the DOM layer. |
| **Platform accessibility APIs** | Microsoft UI Automation, macOS Accessibility API (AXUIElement), Linux AT-SPI, iOS UIAccessibility, Android Accessibility Framework. Native Rust GUI toolkits must bridge to these. |
| **[WCAG2.1 for non-web UIs](https://www.w3.org/TR/wcag2ict/)** (WCAG2ICT) | How to apply WCAG principles to non-web software, including native desktop apps. |

## 3. AccessKit — the cross-framework a11y abstraction

[**AccessKit**](https://accesskit.dev/) is the canonical Rust accessibility project. It is a cross-framework abstraction that bridges a Rust UI toolkit's widget tree to the host operating system's accessibility API:

- **Windows:** UI Automation
- **macOS:** Accessibility API
- **Linux / Unix:** AT-SPI
- **Web / WASM:** browser DOM (via a virtual accessibility tree exposed to the page)

AccessKit is integrated into:

| Toolkit | Status |
|---|---|
| **egui** | First-class. egui was the first pure-Rust GUI toolkit to ship native accessibility via AccessKit. |
| **iced** | AccessKit adapter available. |
| **Slint** | Has its own accessibility story plus AccessKit integration; especially strong on embedded. |
| **Tauri** | Web content rendered by the platform webview; AccessKit is not needed for the HTML layer (the browser handles a11y). |
| **Dioxus (native)** | AccessKit integration where native rendering is used. |
| **Xilem (Linebender)** | AccessKit integration in development. |

The AccessKit philosophy is that **developers should not need to be accessibility experts** — the framework exposes a simple tree of nodes (window, button, text, etc.) with roles, labels, and states, and the platform adapter does the rest. Quote from the project: *"most developers who might use AccessKit are not experts in accessibility."*

### Rust example — exposing a node with AccessKit

```rust
use accesskit::{Node, Role, TreeUpdate};

// Build a minimal accessible tree: a window with a labeled button
let mut update = TreeUpdate {
    nodes: vec![
        Node {
            role: Role::Window,
            label: Some("Settings".into()),
            ..Default::default()
        },
        Node {
            role: Role::Button,
            label: Some("Save changes".into()),
            ..Default::default()
        },
    ],
    // … tree, focus, etc.
    ..Default::default()
};
// Hand `update` to the toolkit's AccessKit adapter.
```

> The exact adapter API depends on the toolkit (egui, iced, Slint). Always prefer the toolkit's high-level accessibility APIs over hand-rolling a tree.

## 4. Terminal UIs (ratatui, crossterm)

[`ratatui`](https://crates.io/crates/ratatui) is the dominant TUI framework. Terminal accessibility considerations:

- **Screen readers** read the visible text in the terminal in reading order; ratatui apps that redraw the whole screen can confuse screen readers because the redraw is not announced. Prefer applications that emit content sequentially when possible, or test explicitly with NVDA / VoiceOver / Orca.
- **Color contrast:** ratatui lets you pick any foreground/background pair; follow WCAG 2.2 AA contrast ratios (≥ 4.5:1 for normal text, ≥ 3:1 for large text) when designing your palette.
- **No mouse required:** ensure every action is reachable via keyboard — this is usually free in a TUI but verify.
- **Reduced motion / no animation:** offer a `--no-animation` flag; some users are sensitive to flicker.
- **Detect a TTY:** when stdout is not a TTY (e.g., piped to a screen reader or a script), disable colors and box-drawing characters; emit plain text.

```rust
// Detect TTY and degrade gracefully
use std::io::IsTerminal;

if std::io::stdout().is_terminal() {
    // render ratatui UI
} else {
    // emit plain text
}
```

## 5. Native GUI frameworks

| Framework | Architecture | a11y story |
|---|---|---|
| **egui** | Immediate mode | AccessKit integrated; first pure-Rust toolkit with native a11y. |
| **iced** | Elm-inspired, retained | AccessKit adapter; accessibility is an explicit design goal. |
| **Slint** | Declarative `.slint` markup, native rendering | Built-in accessibility properties (`accessible-role`, etc.) + AccessKit; strong on embedded/desktop. |
| **Tauri** | Native webview per platform | The HTML/CSS/JS layer is the UI; a11y is the browser's job. Rust code does not interact with a11y directly. |
| **Dioxus** | React-like, multi-renderer | Web target inherits DOM a11y; native targets use AccessKit where available. |
| **Xilem** | Linebender, in active development | AccessKit integration planned. |
| **Druid** | Original Linebender toolkit, now superseded | Has AccessKit integration but is **not recommended for new projects** — use Xilem instead. |

### Guiding principles for native Rust UIs

- Prefer **semantic roles** over hand-drawn visuals. A button rendered as a styled `Rect` must still be exposed to the platform as `Role::Button`, with a label.
- Every interactive element must be **keyboard-reachable** and have a visible focus indicator.
- Provide an **accessible name** for every control (button label, icon tooltip, `aria-label` equivalent).
- Respect the system's **reduced-motion** / **high-contrast** / **dark mode** settings (each toolkit exposes these differently).
- Test with a screen reader on each platform you ship to: NVDA (Windows), VoiceOver (macOS/iOS), TalkBack (Android), Orca (Linux).

## 6. Web frontends in Rust

For Rust-to-WASM frameworks (Leptos, Dioxus, Yew, Sycamore), **accessibility lives at the DOM layer**. The Rust source generates HTML elements, and the same rules apply as in JavaScript frontends:

```rust
// Leptos example — semantic, keyboard-accessible, labeled control
view! {
    <button type="submit" aria-label="Save changes">
        "Save"
    </button>
}
```

Conventions:

- Use **semantic HTML** (`<button>`, `<nav>`, `<main>`, `<dialog>`) rather than rebuilding native controls from `<div>`.
- Provide `alt` for images, `label`/`aria-label` for inputs, and `aria-live` regions for dynamic updates.
- Ensure logical tab order and visible focus styles.
- Run an automated audit (`axe-core`, integrated into your test runner via `wasm-bindgen-test` or a headless browser) in CI, and pair it with manual screen-reader testing.

```rust
// ❌ Non-semantic, keyboard-inaccessible
view! { <div on:click=save>"Save"</div> }

// ✅ Semantic, keyboard-operable, labeled
view! { <button type="submit" aria-label="Save changes">"Save"</button> }
```

## 7. Automated accessibility testing tooling (Rust-aware)

| Tool | Use |
|---|---|
| **axe-core** (via headless Chrome from Rust) | Automated WCAG audits of WASM-rendered pages; integrate into integration tests. |
| **accesskit** consumer-side checks | Walk the accessible tree in tests and assert role/label/focusability. |
| **`wasm-bindgen-test`** + a headless browser | Drive a rendered WASM UI and assert semantic properties. |
| Manual screen readers | NVDA, VoiceOver, TalkBack, Orca — never skip; automated tools catch only ~30–40% of issues. |

## 8. Internationalization and RTL (cross-reference Section 21)

Accessibility and i18n overlap: an RTL UI mislabeled as LTR is an accessibility failure. When shipping a Rust UI:

- Set `dir="rtl"` (web) or the equivalent toolkit direction property based on the active locale.
- Use ICU4X's `LocaleDirection` (cross-reference Section 21) to drive layout direction.
- Test your UI in an RTL locale (Arabic, Hebrew) before shipping.

## Completion checklist

- [x] Applicability scope documented — explicit "not applicable" guidance for non-UI Rust projects.
- [x] Authoritative standards (WCAG, WAI-ARIA, platform a11y APIs, WCAG2ICT) documented.
- [x] AccessKit (cross-framework a11y abstraction) documented with framework integrations.
- [x] Terminal UI (ratatui) accessibility considerations documented.
- [x] Native GUI frameworks (egui, iced, Slint, Tauri, Dioxus, Xilem) documented.
- [x] Web frontend frameworks (Leptos, Dioxus, Yew) — WCAG/ARIA at the DOM layer documented.
- [x] Automated accessibility tooling documented.
- [x] i18n/RTL cross-reference documented.

### References

- AccessKit — https://accesskit.dev/
- WCAG 2.2 — https://www.w3.org/TR/WCAG22/
- WAI-ARIA 1.2 — https://www.w3.org/TR/wai-aria-1.2/
- WCAG2ICT — https://www.w3.org/TR/wcag2ict/
- ratatui — https://ratatui.rs/
- egui accessibility issue — https://github.com/emilk/egui/issues/167
- A 2025 Survey of Rust GUI Libraries — https://www.boringcactus.com/2025/04/13/2025-survey-of-rust-gui-libraries.html
