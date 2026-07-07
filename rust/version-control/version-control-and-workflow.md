---
title: Rust - Version Control and Workflow
status: draft
version: 0.0.1
created: 2026-07-06
updated: 2026-07-06
---

# Rust — Version Control and Workflow

> Rust projects overwhelmingly use **Git** hosted on GitHub (the rust-lang organization itself lives on GitHub). There is **no single mandated branching model or commit convention** for Rust projects — this document presents the recognized alternatives side by side, including how the upstream Rust project itself works, and never picks a winner. Boundary: VCS workflow here; CI/CD pipeline stages are in `rust/cicd/` (planned); build artifacts in `rust/build/`.

## 1. Git is universal; hosting is GitHub-dominant

- The `cargo`/`rustc`/`stdlib/clippy/rustfmt` repos are all under `github.com/rust-lang`.
- crates.io source code is also on GitHub. The vast majority of public Rust crates mirror this.
- Alternatives exist (GitLab, Codeberg, sourcehut) and are used by some projects; document the GitHub default without excluding them.

## 2. `.gitignore` — what to ignore

The canonical Rust `.gitignore` (and the `cargo new` default for binaries) ignores build output:

```gitignore
# Rust build artifacts
/target/

# Cargo lockfile — see §4 for the library/binary debate
# Cargo.lock    # uncomment for libraries following the classic convention

# Editor / OS
.idea/
.vscode/
*.swp
.DS_Store
```

The gitignore.io Rust template (https://www.toptal.com/developers/gitignore/api/rust) is essentially `/target/` plus IDE/OS files. The community consensus items:

| Path | Ignore? | Why |
|---|---|---|
| `/target/` | ✅ always | Build output; large, machine-generated. |
| `Cargo.lock` | ⚠️ debated | See §4. |
| `*.rs.bk` | ✅ | Leftover rustfmt backups. |

✅ Scope `/target/` with a leading slash so it only matches the repo root.

❌ Committing `/target/` (or `target/` without the leading slash matching nested ones inconsistently) — massive, noisy diffs.

## 3. The `Cargo.lock` commit debate (cross-ref dependencies)

See `rust/dependencies/dependency-management.md` §3 for the full treatment. Summary of the two recognized positions:

1. **Always commit (post-2023 Cargo team recommendation):** for reproducible CI and bisection.
2. **Classic split:** binaries commit it; libraries gitignore it because downstream consumers regenerate it.

Both are defensible; document your project's choice in the README.

## 4. Branching strategies — alternatives side by side

| Strategy | Description | Trade-offs |
|---|---|---|
| **Trunk-Based Development** | Everyone commits to `main`; short-lived feature branches; continuous integration. | Fast feedback, simpler; requires strong CI and small changes. Favored by many modern Rust shops. |
| **GitHub Flow** | `main` is always deployable; feature branches + PRs; deploy on merge. | Simple, pairs with crates.io publishing on tag. |
| **GitFlow** | `main` + `develop` + `release/*` + `hotfix/*` branches. | Heavier; suits projects with formal release management and long-lived releases. Less common in Rust. |
| **Release-train (upstream Rust's model)** | `master` continuously integrates; release branches cut per 6-week stable cycle. | Used by rust-lang/rust itself and large projects with a release train. |

The upstream Rust compiler uses a **bors-style merge queue** (formerly the `bors` bot, now `homu`/GitHub merge queue hybrids) so that PRs are tested on a merged state before landing on `master`, keeping `master` always green.

> No winner is picked here. Selection happens downstream in StandardBuilder.

## 5. Commit message conventions — alternatives side by side

Rust projects use several conventions; none is universal.

### 5.1 Conventional Commits
```
feat(auth): add refresh token rotation
fix(orders): prevent duplicate shipment on retry
docs(readme): document local setup
chore(deps): bump serde to 1.0.197
```
Increasingly common in Rust crates that ship a CHANGELOG and use tooling like `release-plz` or `cargo-release`.

### 5.2 Free-form / descriptive (upstream Rust style)
The rust-lang/rust repo uses descriptive, human-readable commit messages, often prefixed by a subsystem (e.g., `rustc: ...`, `cargo: ...`, `std: ...`) but **not** Conventional Commits format. Example: `Rollup of pull requests #12345`.

### 5.3 Imperative mood (Git default guidance)
`Add X`, `Fix Y`, `Refactor Z` — the generic Git best practice; many crates use this with no formal scheme.

| Convention | When you see it |
|---|---|
| Conventional Commits | Crates using automated changelogs (`release-plz`, `cargo-release`, semantic-release). |
| Subsystem-prefixed prose | Upstream rust-lang repos and large projects. |
| Plain imperative | Most small/medium crates. |

✅ Pick one per project and enforce it in CI / a `CONTRIBUTING.md`. Document the choice.

❌ Mixing `feat:`, `Fix:`, and `added stuff` in the same log.

## 6. Branch naming

Common patterns (no universal rule):
- `feature/<topic>` / `feat/<topic>`
- `fix/<bug>` / `bugfix/<topic>`
- `chore/<topic>`
- Issue-keyed: `1234-add-login`

✅ Lowercase, hyphen-separated, descriptive, optionally prefixed by type and/or issue number.

## 7. Pull Requests — size, checks, reviews

Recognized conventions (community, not mandated):
- **Size:** keep PRs small and reviewable; many Rust projects aim for PRs reviewable in < 30 min.
- **Required checks (typical):** `cargo fmt --check`, `cargo clippy -- -D warnings`, `cargo test` (often via `cargo nextest`), `cargo audit` / `cargo deny check`, and MSRV build.
- **Reviews (SLA and Process):** upstream Rust requires at least one (often two) reviewer sign-offs via GitHub PR review; `r+` comments were historically used via `bors`. Projects typically aim for an SLA of 24-48 business hours for initial review to avoid blocking contributors.
- **Reviewer Expectations:** Reviewers are expected to explicitly verify:
  - **`unsafe` blocks:** Must be accompanied by a `// SAFETY:` comment justifying the invariants. Reviewers must rigorously scrutinize these for soundness.
  - **Breaking changes:** Ensure public API changes are intended and version-bumped correctly.
  - **Performance and allocations:** Flag unnecessary allocations (e.g., `.clone()` in hot paths) or sub-optimal data structures.
  - **Tests and docs:** Verify new logic has test coverage and public APIs have docstrings.
- **Description:** the upstream Rust PR template asks for a description, motivation, and "does this break anything".

## 8. Tags and releases

- **Annotated tags** are conventional for releases: `git tag -a v1.2.3 -m "..."`.
- **SemVer tags** (`vMAJOR.MINOR.PATCH`) are the norm for crates that publish to crates.io, since the crates.io version must match.
- Release artifacts are attached to GitHub Releases for binaries.

## 9. `CHANGELOG.md` and release tooling

| Tool | Role |
|---|---|
| **cargo-release** | Automates version bumping, tagging, publishing to crates.io, and pushing. The most established release tool. |
| **release-plz** | Generates a CHANGELOG from Conventional Commits and publishes one crate per merged PR; popular for workspaces. |
| **cargo-smart-release** | Workspace-aware release management. |
| **Hand-written `CHANGELOG.md`** | Follows "Keep a Changelog" format (https://keepachangelog.com); used by many crates. |

✅ Use `cargo release` (or `release-plz`) to make versioning + publishing deterministic:
```bash
cargo release patch --execute    # bumps version, tags, publishes
```

❌ Manually editing `Cargo.toml` version + tagging + publishing in ad-hoc steps — error-prone.

## 10. crates.io publishing flow

The end-to-end release path:
1. Bump version in `Cargo.toml` (and workspace Cargo if needed).
2. Update `Cargo.lock` (run `cargo update -p <self>`).
3. Update `CHANGELOG.md`.
4. `cargo publish --dry-run` — verifies packaging.
5. `cargo publish` — uploads to crates.io (requires an API token via `cargo login`).
6. Tag the release: `git tag v1.2.3 && git push --tags`.
7. (For binaries) create a GitHub Release and attach build artifacts.

Notes:
- crates.io **does not allow republishing a version** — once `1.2.3` is published, it is permanent (you can yank it with `cargo yank`, which only prevents new resolves from picking it; existing lockfiles still work).
- `cargo yank` is the official "soft delete" for a broken version.

✅ `cargo publish --dry-run` before every release.

❌ Force-pushing a published version's tag after the fact — breaks downstream `cargo install --locked`.

## 11. Monorepo vs multi-repo

| Approach | Rust tooling | Trade-offs |
|---|---|---|
| **Cargo workspace (monorepo)** | Native `[workspace]`; shared `Cargo.lock` and `target/`. | Excellent for tightly-coupled crates; the dominant pattern in Rust. |
| **Multi-repo** | Each repo its own crate. | Easier independent versioning; loses workspace benefits. |
| **Cargo workspaces + path deps for local, version deps for publish** | `[dependencies] foo = { path = "../foo", version = "1.0" }` | Lets monorepos publish independent versions. |

## Sources

| Topic | Source |
|---|---|
| Upstream Rust workflow | https://rustc-dev-guide.rust-lang.org/contributing.html |
| rust-lang GitHub org | https://github.com/rust-lang |
| Cargo.lock guidance | https://blog.rust-lang.org/2023/08/29/committing-lockfiles/ |
| Conventional Commits | https://www.conventionalcommits.org |
| Keep a Changelog | https://keepachangelog.com |
| cargo-release | https://github.com/crate-ci/cargo-release |
| release-plz | https://github.com/release-plz/release-plz |
| cargo yank | https://doc.rust-lang.org/cargo/commands/cargo-yank.html |
