---
title: Programming Standard - Constituent Elements
status: draft
version: 0.2.0
created: 2026-07-05
updated: 2026-07-06
---

# Programming Standard - Constituent Elements

This document defines the **19 research areas** that any robust programming standard must cover. Its sole purpose is to serve as the **research backbone** for the `/standard_research` command — it guides *what* to investigate for a given language, not *what the rules should be*.

> **This document does not define standards.** Every section describes research objectives, not recommendations. The language-specific documents produced by `/standard_research` are where the actual standards live.

> **This file never leaves this repository.** It is not copied into language directories. It is exclusively the guide used by `/standard_research`.

The section order does not imply priority. Each section becomes one independent file in the language directory.

---

## Table of Contents

1. [Scope and Purpose](#1-scope-and-purpose)
2. [Naming Conventions](#2-naming-conventions)
3. [Formatting and Structure Style](#3-formatting-and-structure-style)
4. [Project Architecture and Directory Layout](#4-project-architecture-and-directory-layout)
5. [Language Rules and Best Practices](#5-language-rules-and-best-practices)
6. [Documentation and Comments](#6-documentation-and-comments)
7. [Error Handling, Exceptions, and Logging](#7-error-handling-exceptions-and-logging)
8. [API Design and Data Contracts](#8-api-design-and-data-contracts)
9. [Security and Privacy](#9-security-and-privacy)
10. [Dependency Management](#10-dependency-management)
11. [Testing and Quality Validation](#11-testing-and-quality-validation)
12. [Version Control and Workflow](#12-version-control-and-workflow)
13. [Automation Tools (Linters and Formatters)](#13-automation-tools-linters-and-formatters)
14. [Build, Packaging, and Containerization](#14-build-packaging-and-containerization)
15. [Performance and Resource Efficiency](#15-performance-and-resource-efficiency)
16. [Concurrency and Thread Safety](#16-concurrency-and-thread-safety)
17. [Governance and Standard Updates](#17-governance-and-standard-updates)
18. [Ecosystem Conventions](#18-ecosystem-conventions)
19. [Development Toolchain](#19-development-toolchain)

---

## 1. Scope and Purpose

Defines the "what", "why", and "for whom" of a standard. Every standard produced from this library should document its own scope so consumers know exactly what it covers.

### Research Objectives

Research and document:

- What the official language or ecosystem documentation says about the intended use cases of the language.
- Whether there is an official or widely accepted definition of scope for standards in this ecosystem (e.g., PSR for PHP, PEP for Python).
- How existing community style guides define their own applicability (which layers, which project types, which lifecycle phases).
- Whether the language has a governance body that publishes scope definitions (e.g., TC39 for JavaScript, JCP for Java).
- How different communities (enterprise, open-source, embedded, academia) interpret and limit the scope of their standards.

### Illustrative concepts *(research these, do not assume they apply)*

- Primary objective of the standard: unify criteria to reduce technical debt and facilitate teamwork.
- Applicability dimensions: language version, frameworks, architectural layers, project types (library vs. application vs. script).
- Lifecycle applicability: new code only, refactoring, legacy maintenance.
- External reference standards (e.g., MISRA, CERT, PSR-12, Google Style Guides).

### Research Completion Checklist

- [ ] Official language documentation reviewed for stated scope and intended use cases.
- [ ] Existing community-recognized style guides identified and their scope sections documented.
- [ ] Governance body and its publications identified (if any).
- [ ] Applicability dimensions (layers, project types, lifecycle phases) documented for this language's ecosystem.

---

## 2. Naming Conventions

Covers the rules by which identifiers — classes, functions, variables, files, and other named elements — are written in the language.

### Research Objectives

Research and document:

- Official language specification rules for identifiers (legal characters, length limits, reserved words).
- Official style guide naming recommendations (if one exists).
- Community-adopted conventions for each identifier category: classes/types, functions/methods, variables/parameters, constants/enumerations, interfaces/traits/protocols, files and modules, private/internal members.
- Whether the language is case-sensitive and how that affects conventions.
- Naming conventions for booleans, predicates, and event handlers.
- Conventions for prefixes, suffixes, and namespaces.
- Cases where the community is split between multiple recognized conventions — document all of them with their trade-offs.

### Illustrative concepts *(research these, do not assume they apply)*

- `PascalCase` for classes and types (e.g., `UserService`).
- `camelCase` or `snake_case` for functions and variables.
- `UPPERCASE_WITH_UNDERSCORES` for constants.
- `I`-prefix or `-able`-suffix for interfaces.
- `_`-prefix for private members.
- `is`/`has`/`can`/`should` prefix for booleans.

**Example (good vs. bad — illustrative only; research what applies to the target language):**

```
// Meaningless name vs. intent-revealing name
int d;                   →   int daysSinceLastLogin;
boolean flag;            →   boolean isEmailVerified;
public class usr_svc { } →   public class UserService { }
```

### Research Completion Checklist

- [ ] Official language spec rules for identifiers documented.
- [ ] Official or widely adopted style guide naming rules documented for each identifier category.
- [ ] Community-split conventions documented side by side with trade-offs.
- [ ] Boolean and predicate naming conventions documented.
- [ ] File and module naming conventions documented.

---

## 3. Formatting and Structure Style

Covers the visual presentation of code: indentation, line length, brace placement, spacing, and the internal ordering of code elements within a file.

### Research Objectives

Research and document:

- Official style guide recommendations for indentation (tabs vs. spaces, width).
- Maximum line length recommendations and whether they differ by context (code vs. comments vs. strings).
- Brace placement styles recognized in the ecosystem (e.g., K&R, Allman, Stroustrup) and which is conventional.
- Spacing rules around operators, control structures, and function calls.
- Import/include/use statement ordering conventions.
- Canonical internal ordering of elements within a file or class (fields, constructors, methods, etc.).
- Whether the language has an official formatter that enforces these rules automatically.
- Cases where the community is split — document all recognized approaches.

### Illustrative concepts *(research these, do not assume they apply)*

- Indentation: spaces vs. tabs, 2 vs. 4 width.
- Maximum line length: 80, 100, or 120 characters.
- Brace styles: K&R (same line) vs. Allman (new line).
- Import grouping: standard library → external → internal, alphabetically within groups.
- File internal order: static fields → instance fields → constructors → public methods → private methods.

**Example (brace styles — illustrative only):**

```
// K&R style              // Allman style
if (isValid) {            if (isValid)
    process();            {
}                             process();
                          }
```

### Research Completion Checklist

- [ ] Official style guide formatting rules documented (or noted as absent).
- [ ] Official formatter identified (if one exists) and its default behavior documented.
- [ ] Indentation, line length, and brace conventions documented with recognized alternatives if split.
- [ ] Import/include ordering conventions documented.
- [ ] Internal file/class ordering conventions documented.

---

## 4. Project Architecture and Directory Layout

Covers the structure of the repository itself — where each piece of code lives and why — as distinct from the internal structure of a single file (which is Section 3).

### Research Objectives

Research and document:

- The canonical project layout defined or recommended by the language's official documentation or tooling.
- The directory conventions of the dominant build tools or frameworks in the ecosystem (e.g., Maven layout for Java, src-layout for Python, `cmd/internal/pkg` for Go, `src/dist` for Node.js).
- Widely used architectural patterns in this ecosystem and the directory structures that accompany them. Do not assume any single architecture (such as Clean Architecture, Hexagonal, or MVC) is the default — document what the ecosystem actually uses.
- Framework-specific project layouts for the most widely adopted frameworks (e.g., Spring Boot, Django, Laravel, Rails, NestJS, Phoenix, ASP.NET).
- Where to place: source code, tests, generated files, configuration, scripts, documentation, assets, database migrations.
- Package/module organization strategies (by feature, by layer, by domain, etc.) and their trade-offs.
- What should never be committed (build outputs, caches, secrets, generated code) and standard ignore rules.
- Community-recognized monorepo patterns if applicable.

### Illustrative concepts *(research these, do not assume they apply)*

- Canonical root folders: `src/`, `tests/`, `docs/`, `scripts/`, `config/`.
- Ecosystem-specific idioms: `cmd/`, `internal/`, `pkg/` in Go; Maven's `src/main/java` and `src/test/java`; Python src-layout.
- Organization strategy: by feature (`orders/`, `users/`) vs. by layer (`controllers/`, `services/`, `repositories/`) — document trade-offs, do not prescribe.
- Common architectural patterns (MVC, layered, hexagonal, feature-based, etc.) as ecosystem examples — research which ones are idiomatic, not which one is correct.

**Example (illustrative layout for one ecosystem — research the equivalent for the target language):**

```
myservice/            ← research what the ecosystem names this
├── src/              ← or cmd/, lib/, app/ — depends on language
├── tests/            ← or spec/, test/ — research the convention
├── docs/
├── scripts/
└── <build-file>      ← pom.xml, go.mod, package.json, etc.
```

### Research Completion Checklist

- [ ] Official or tooling-defined canonical layout documented.
- [ ] At least the two most popular frameworks' project layouts documented.
- [ ] Module/package organization strategies documented with trade-offs.
- [ ] Test placement conventions documented (colocated vs. mirrored tree).
- [ ] Standard ignore rules documented.
- [ ] No single architectural style presented as default — alternatives covered.

---

## 5. Language Rules and Best Practices

Covers language-specific rules about how to use the language itself: which features to use, which to avoid, and what design principles are idiomatic.

### Research Objectives

Research and document:

- The language's typing model and official or community recommendations regarding type usage (strong/weak, static/dynamic, type inference, generics, type hints).
- Language features that are officially deprecated, discouraged, or known to be dangerous — and why.
- Official or community-recognized limits on complexity (function length, nesting depth, cyclomatic complexity).
- Design principles that are idiomatic to this language and ecosystem (these vary by language — do not assume SOLID, DRY, or KISS apply universally or in the same form).
- Immutability conventions and how the language supports or encourages them.
- Memory management model and associated best practices (relevant in languages like C, C++, Rust).
- Idiomatic error handling model (exceptions, return values, result types, option types — research what is conventional).
- Design patterns that are widely used and recognized in this ecosystem, and those that are considered anti-patterns.
- Community-recognized code smells specific to this language.

### Illustrative concepts *(research these, do not assume they apply)*

- Prohibited constructs: `goto`, `eval`, `global` — depends on the language.
- Complexity limits: max lines per function, nesting depth — research what the community recognizes.
- Design principles: DRY, KISS, SOLID — research whether and how they manifest in this language.
- Guard clauses, early returns — research whether they are idiomatic.

**Example (illustrative — verify idiom for the target language):**

```
// Reducing nesting with early returns — illustrative pattern
// Research whether this is idiomatic in the target language

// Deep nesting approach:
function process(order) {
  if (order) { if (order.isPaid) { if (order.items > 0) { ship(order); } } }
}

// Early return approach:
function process(order) {
  if (!order) return;
  if (!order.isPaid) return;
  if (order.items === 0) return;
  ship(order);
}
```

### Research Completion Checklist

- [ ] Language typing model and type usage recommendations documented.
- [ ] Officially deprecated or discouraged language features listed with rationale.
- [ ] Community-recognized complexity limits documented.
- [ ] Idiomatic design principles for this language documented (not assumed from other languages).
- [ ] Memory management best practices documented (if applicable).
- [ ] Common design patterns and anti-patterns specific to this ecosystem documented.

---

## 6. Documentation and Comments

Covers how to document code through inline comments, block comments, and structured documentation formats.

### Research Objectives

Research and document:

- The official or community-standard documentation format for this language (e.g., JSDoc, Javadoc, Docstrings/Sphinx, Rustdoc, Godoc, XML docs for C#).
- Required and optional tags in the documentation format (e.g., `@param`, `@return`, `@throws`, `@example`).
- Community conventions for when to write inline comments and what they should express.
- Conventions for block comments and their appropriate use cases.
- Conventions for marker comments (`TODO`, `FIXME`, `HACK`, `NOTE`) including whether they should reference tickets, authors, or dates.
- Whether the ecosystem has tooling that generates documentation from code comments (identify it).
- README conventions for modules, packages, and components.
- Whether the ecosystem favors "self-documenting code" over comments and how that manifests.

### Illustrative concepts *(research these, do not assume they apply)*

- DocBlock template structure with mandatory tags.
- Inline comments: explain *why*, not *what*.
- TODO markers with author and ticket reference.
- Per-module README requirement.

**Example (DocBlock template — research the equivalent format for the target language):**

```
/**
 * Brief description of what this does.
 *
 * @param paramName   description; constraints (e.g., must be >= 0)
 * @return            description of the return value
 * @throws ExcType    when this exception is thrown
 */
```

**Example (comment intent — illustrative):**

```
// ❌ Repeats the code:    counter += 1;  // increment counter
// ✅ Explains the why:    counter += 1;  // API is 1-indexed; offset before sending
```

### Research Completion Checklist

- [ ] Official or community-standard documentation format identified and its required tags documented.
- [ ] Documentation generation tooling identified.
- [ ] Inline and block comment conventions documented.
- [ ] Marker comment conventions documented.
- [ ] README conventions for modules/packages documented.

---

## 7. Error Handling, Exceptions, and Logging

Covers how the language and ecosystem handle errors, anomalous situations, and observability through logging, metrics, and tracing.

### Research Objectives

**Error handling:**

- The language's fundamental error model (exceptions, error return values, result/option types, panic/recover — research what is native and idiomatic).
- Official or community recommendations on exception hierarchy and custom exception design.
- Conventions for fail-fast vs. resilience/retry patterns and when each is appropriate.
- Recommendations for input validation before processing.
- Whether swallowing errors silently is explicitly discouraged, and how.

**Logging:**

- The standard logging library or framework for this language/ecosystem.
- Recognized log level semantics (`DEBUG`, `INFO`, `WARN`, `ERROR`, and any others) and what belongs at each level.
- Structured logging conventions (JSON logs, key-value pairs) vs. plaintext.
- What must never appear in logs (passwords, tokens, PII) and how the ecosystem handles log sanitization.

**Observability (beyond logging):**

- Structured logging standards and tooling in this ecosystem.
- Metrics conventions (counters, gauges, histograms) and the dominant library or protocol (e.g., Prometheus, StatsD, OpenTelemetry).
- Distributed tracing conventions and the dominant standard (e.g., OpenTelemetry, Jaeger, Zipkin).
- Correlation ID / request ID conventions.
- Health check, readiness probe, and liveness probe conventions (especially for services).
- Whether the ecosystem has an official or de facto telemetry standard.

### Illustrative concepts *(research these, do not assume they apply)*

- Empty catch block prohibition.
- Centralized global error handler for APIs.
- Log level table: DEBUG for diagnostics, INFO for business events, WARN for recoverable anomalies, ERROR for failures.

**Example (log level semantics — illustrative; verify for the target ecosystem):**

| Level   | Purpose                          | Example                                  |
|---------|----------------------------------|------------------------------------------|
| `DEBUG` | Diagnostic detail, dev/test only | `"Cache miss for key user:42"`           |
| `INFO`  | Normal business events           | `"Order 1234 created"`                   |
| `WARN`  | Recoverable anomalies            | `"Retry 2/3 calling payment API"`        |
| `ERROR` | Failures requiring attention     | `"Payment API unreachable after 3 retries"` |

### Research Completion Checklist

- [ ] Language's native error model documented (exceptions, result types, error values, etc.).
- [ ] Standard logging library/framework identified and log level semantics documented.
- [ ] Structured logging conventions documented.
- [ ] PII and secret sanitization in logs documented.
- [ ] Dominant metrics and tracing standards/libraries for this ecosystem identified.
- [ ] Correlation ID and health check conventions documented (for service-oriented use cases).

---

## 8. API Design and Data Contracts

Covers how data and capabilities are exposed externally. This section addresses the *external contract*, as distinct from Section 7 which covers *internal* error handling. Primarily relevant when the language is used to build web services or libraries consumed by other systems.

### Research Objectives

Research and document:

- Official or community-recognized REST API design conventions for this ecosystem (resource naming, HTTP verb usage, URL structure).
- JSON payload field naming conventions (`camelCase`, `snake_case`, `PascalCase`) — note that this may differ from the internal language convention.
- API versioning strategies recognized in this ecosystem (URI versioning, header versioning, media type versioning) and their trade-offs.
- Standard HTTP status code semantics as applied in this ecosystem.
- Community-standard error response body format (e.g., RFC 7807 Problem Details, or ecosystem-specific formats).
- Pagination, filtering, and sorting conventions.
- Contract-first design tools and formats used in this ecosystem (OpenAPI/Swagger, Protobuf, GraphQL, AsyncAPI).
- Backward compatibility rules and deprecation conventions.
- Idempotency conventions for safe retries.
- If the language is not typically used for web services, document what contract-based communication looks like (e.g., library APIs, RPC, message queues).

### Illustrative concepts *(research these, do not assume they apply)*

- URL naming: plural nouns, kebab-case, no verbs in paths.
- Versioning in URI: `/api/v1/...`.
- RFC 7807 error body with `type`, `title`, `status`, `detail`, `instance`, `correlationId`.
- Backward compatibility: additive changes allowed within a version; renames/removals require a new version.

**Example (error body format — illustrative; research the convention for the target ecosystem):**

```json
{
  "type": "https://api.example.com/errors/insufficient-funds",
  "title": "Insufficient funds",
  "status": 422,
  "detail": "Balance is 12.50, transfer requires 100.00",
  "correlationId": "b7ad6b7169203331"
}
```

### Research Completion Checklist

- [ ] URL/resource naming conventions documented.
- [ ] Payload field casing convention documented (and any divergence from internal language convention noted).
- [ ] API versioning strategies documented with trade-offs.
- [ ] Standard error response format for this ecosystem documented.
- [ ] Contract-first tooling identified (OpenAPI, Protobuf, etc.).
- [ ] Backward compatibility and deprecation conventions documented.

---

## 9. Security and Privacy

Covers practices to prevent vulnerabilities and protect sensitive data. This section should be researched with reference to OWASP, CERT, and any language- or ecosystem-specific security guidance.

### Research Objectives

Research and document:

- The OWASP Top 10 risks as they specifically manifest in this language/ecosystem, and the idiomatic mitigation for each.
- Code injection prevention techniques idiomatic to this ecosystem (parameterized queries, ORM usage, output encoding).
- Secrets management conventions: what the ecosystem recommends for managing credentials, tokens, and API keys.
- Authentication and authorization patterns recognized in this ecosystem.
- Data encryption conventions (at rest and in transit) as recommended for this ecosystem.
- File upload validation conventions.
- PII handling and privacy conventions (logging, storage, transmission).
- Security-focused static analysis tools available for this language.
- Whether the language or ecosystem has official security advisories or a CVE tracking process.

### Illustrative concepts *(research these, do not assume they apply)*

- Parameterized queries vs. string concatenation for SQL.
- Environment variables or secret managers for credentials.
- Never logging passwords, tokens, or PII.

**Example (SQL injection — illustrative; research the idiomatic approach for the target language):**

```
// ❌ String concatenation:  "SELECT * FROM users WHERE email = '" + email + "'"
// ✅ Parameterized query:   cursor.execute("SELECT ... WHERE email = %s", (email,))
```

### Research Completion Checklist

- [ ] OWASP Top 10 mitigations documented as they apply to this language/ecosystem.
- [ ] Injection prevention techniques documented with idiomatic examples.
- [ ] Secrets management conventions documented.
- [ ] Authentication and authorization patterns documented.
- [ ] PII handling and log sanitization conventions documented.
- [ ] Security-focused static analysis tools identified for this language.

---

## 10. Dependency Management

Covers how external libraries and packages are introduced, versioned, maintained, and secured. *Boundary with Section 14:* this section focuses on external library lifecycle (what to use and how to manage it); Section 14 focuses on producing the build artifact (how to compile and package).

### Research Objectives

Research and document:

- The primary package manager(s) for this language and their conventions (covered in depth in Section 18 — cross-reference).
- Version pinning conventions: exact pinning vs. semantic versioning ranges — and which the ecosystem recommends.
- Lockfile conventions: which file, whether it is committed, and its role.
- Trusted/official package repositories for this ecosystem and how to verify a package's provenance.
- Dependency auditing tools and conventions (`npm audit`, `cargo audit`, OWASP Dependency-Check, etc.).
- Dependency update cadence conventions (how often to review and update).
- How to evaluate a new dependency before adding it (license, maintenance activity, download volume, security history).
- Transitive dependency management conventions.
- How the ecosystem handles vendoring (copying dependencies into the repository).

### Illustrative concepts *(research these, do not assume they apply)*

- SemVer range policy: exact pin for critical runtime deps, ranges for utilities.
- Lockfile: always committed for applications, sometimes gitignored for libraries.
- Audit tools integrated into CI pipeline.

### Research Completion Checklist

- [ ] Primary package manager(s) identified and their version pinning conventions documented.
- [ ] Lockfile role and commit conventions documented.
- [ ] Trusted repositories identified.
- [ ] Dependency auditing tools documented and their CI integration described.
- [ ] Criteria for evaluating new dependencies documented.
- [ ] Vendoring conventions documented (if applicable).

---

## 11. Testing and Quality Validation

Covers how to verify that code behaves correctly: unit, integration, and other testing conventions.

### Research Objectives

Research and document:

- The primary testing framework(s) for this language and their conventions.
- Test organization: where tests live relative to the code they test (Section 4 cross-reference), how test files are named.
- Test structure conventions (e.g., Arrange-Act-Assert, Given-When-Then, describe/it).
- Test naming conventions.
- Code coverage conventions: what to measure (lines, branches, paths), minimum thresholds recognized by the community.
- Unit test conventions: isolation approaches, mocking/stubbing libraries and idioms.
- Integration test conventions: what to test together, how to handle external dependencies (real vs. test doubles).
- Contract testing conventions (if applicable in this ecosystem).
- Property-based testing availability and conventions.
- Static analysis tools and their role in quality validation (cross-reference Section 13).
- Community conventions for test independence (avoiding shared mutable state between tests).

### Illustrative concepts *(research these, do not assume they apply)*

- AAA pattern: Arrange, Act, Assert.
- Test method naming: `whenX_givenY_thenZ` or `should_doX_when_Y`.
- Coverage minimum: > 80% lines and branches.

**Example (AAA pattern — illustrative; research the idiomatic structure for the target language):**

```
// Arrange: set up inputs and context
// Act:     call the unit under test
// Assert:  verify the outcome
```

### Research Completion Checklist

- [ ] Primary testing framework(s) identified and their conventions documented.
- [ ] Test file naming and location conventions documented.
- [ ] Test structure convention documented (AAA, GWT, or other).
- [ ] Coverage tooling and community-recognized thresholds documented.
- [ ] Mocking/stubbing approach documented.
- [ ] Integration test scope and conventions documented.
- [ ] Test independence conventions documented.

---

## 12. Version Control and Workflow

Covers conventions for using version control systems (primarily Git) in this ecosystem.

### Research Objectives

Research and document:

- Branching strategies recognized in this ecosystem and their trade-offs (GitFlow, GitHub Flow, Trunk-Based Development, others).
- Branch naming conventions.
- Commit message conventions recognized in this ecosystem (Conventional Commits, Angular, or other).
- Pull Request / Merge Request conventions: size, required checks, review requirements.
- Code review conventions: what reviewers are expected to verify, response time expectations.
- Tagging and release conventions.
- Monorepo vs. multi-repo conventions in this ecosystem.
- Any ecosystem-specific version control tooling or extensions.

### Illustrative concepts *(research these, do not assume they apply)*

- Conventional Commits: `feat:`, `fix:`, `docs:`, `chore:`.
- Branch naming: `feature/TICKET-description`, `hotfix/description`.
- Atomic commits: small, single-purpose changes.

**Example (Conventional Commits format — illustrative; research whether this is conventional in the target ecosystem):**

```
feat(auth): add refresh token rotation
fix(orders): prevent duplicate shipment on retry
docs(readme): document local setup
chore(deps): bump dependency version
```

### Research Completion Checklist

- [ ] Recognized branching strategies documented with trade-offs (do not prescribe one).
- [ ] Commit message convention documented (Conventional Commits or ecosystem alternative).
- [ ] Branch naming conventions documented.
- [ ] PR/MR conventions documented (size, required checks, approvals).
- [ ] Release and tagging conventions documented.

---

## 13. Automation Tools (Linters and Formatters)

Covers the tooling used to automatically enforce standards without human intervention. *Boundary with Section 14:* this section covers quality enforcement tools; Section 14 covers build and artifact production.

### Research Objectives

Research and document:

- The canonical linter(s) for this language: their names, what they check, and their configuration format.
- The canonical formatter(s) for this language and whether formatting is part of the language toolchain (e.g., `gofmt`, `rustfmt` are official).
- Static Application Security Testing (SAST) tools available for this language.
- Code quality platforms used in this ecosystem (SonarQube, CodeClimate, Codacy, etc.).
- Pre-commit hook tooling conventions (Husky, pre-commit, lefthook, etc.).
- How linter and formatter configurations are shared across a team (config files in the repository).
- CI/CD integration conventions: at what stage quality gates run and what constitutes a failure.
- Whether the ecosystem has an official quality/style enforcement tool bundled with the language.

### Illustrative concepts *(research these, do not assume they apply)*

- Linters: ESLint, Pylint, Rubocop, Checkstyle, Clippy.
- Formatters: Prettier, Black, gofmt, rustfmt.
- CI gate: pipeline fails on lint or format violations.
- Shared config file committed to the repository.

### Research Completion Checklist

- [ ] Canonical linter(s) identified with their configuration format documented.
- [ ] Canonical formatter(s) identified; note if officially bundled with the language.
- [ ] SAST tools available for this language identified.
- [ ] Pre-commit hook tooling conventions documented.
- [ ] CI/CD quality gate integration conventions documented.

---

## 14. Build, Packaging, and Containerization

Covers how the final deliverable artifact is produced and distributed. *Boundary with Section 10:* Section 10 covers managing which external libraries to use; this section covers producing the build artifact from the source code.

### Research Objectives

Research and document:

- The build system(s) canonical to this language (e.g., Maven/Gradle for Java, Cargo for Rust, the Go toolchain, MSBuild for C#).
- Standard build commands and conventions (the "single entry point" principle — one command to build).
- Compiler or transpiler flag conventions per environment (development, CI, release).
- Reproducible build conventions: lockfiles, toolchain pinning, deterministic output.
- Artifact naming and versioning conventions.
- Environment configuration conventions (12-factor, `.env` files, platform-specific approaches).
- Container image conventions for this language/ecosystem: approved base images, multi-stage build patterns, non-root user conventions, image vulnerability scanning.
- Artifact signing and software bill of materials (SBOM) conventions.
- Distribution conventions: artifact registries, package publishing.

### Illustrative concepts *(research these, do not assume they apply)*

- Single build entry point: `make build`, `./gradlew build`, `cargo build`.
- 12-factor: one artifact, configuration from environment, never per-environment builds.
- Multi-stage Docker builds with minimal runtime image and non-root user.
- Toolchain version pinned in a lockfile or toolchain file.

**Example (multi-stage container build — illustrative; research the idiomatic base image for the target language):**

```dockerfile
# Build stage: full toolchain
FROM <language-official-image>:<version> AS build
# ... build steps ...

# Runtime stage: minimal image, non-root user
FROM <minimal-runtime-image>
COPY --from=build /artifact /artifact
USER nonroot
ENTRYPOINT ["/artifact"]
```

### Research Completion Checklist

- [ ] Canonical build system(s) identified and standard build commands documented.
- [ ] Compiler/transpiler flag conventions per environment documented.
- [ ] Reproducible build approach documented.
- [ ] Artifact naming and versioning conventions documented.
- [ ] Environment configuration conventions documented.
- [ ] Official or community-standard container base images identified.
- [ ] SBOM and artifact signing conventions documented (if applicable).

---

## 15. Performance and Resource Efficiency

Covers practices to prevent avoidable performance problems.

### Research Objectives

Research and document:

- Algorithmic complexity conventions: are there community-recognized guidelines about when to use which data structure or algorithm?
- Database access patterns that are considered anti-patterns in this ecosystem (e.g., N+1 queries) and their idiomatic solutions.
- Resource lifecycle management conventions (connections, file handles, streams, sockets) — how the language and ecosystem manage them.
- Caching conventions: what layers own caching, which libraries are standard, invalidation patterns.
- Profiling and benchmarking tooling available for this language.
- Memory management and garbage collection tuning guidance (if applicable).
- Community conventions around premature optimization: when to optimize and how to justify it.
- Performance testing conventions (load tests, benchmarks) if they exist in this ecosystem.

### Illustrative concepts *(research these, do not assume they apply)*

- N+1 query pattern and solutions (batch fetching, joins, eager loading).
- Resource lifecycle: `try-with-resources`, `using`, context managers, RAII.
- Profiling tools standard to this language.

**Example (resource lifecycle — illustrative; research the idiomatic approach for the target language):**

```
// Automatic resource release patterns vary by language:
// Java:   try (Connection c = ...) { }
// Python: with open(...) as f:
// C#:     using (var c = ...) { }
// Rust:   ownership/drop (automatic)
// Research what is idiomatic for the target language
```

### Research Completion Checklist

- [ ] Community-recognized algorithmic complexity guidelines documented.
- [ ] Common performance anti-patterns for this ecosystem documented with idiomatic solutions.
- [ ] Resource lifecycle management conventions documented.
- [ ] Profiling and benchmarking tooling identified.
- [ ] Caching layer conventions documented (if applicable).
- [ ] Memory/GC tuning guidance documented (if applicable).

---

## 16. Concurrency and Thread Safety

Covers code that runs in parallel or asynchronously. Relevance varies significantly by language (critical in Java, C#, Go, Rust; different model in Python, JavaScript, Elixir).

### Research Objectives

Research and document:

- The concurrency model of this language (threads, async/await, coroutines, actors, CSP channels, green threads, event loop — research what is native).
- Community conventions for shared mutable state (minimize it, use immutability, message passing, etc.).
- Synchronization primitives available in this language and when each is appropriate (locks, mutexes, semaphores, atomics, channels).
- Thread-safe data structures available in the standard library or ecosystem.
- Async/await conventions if applicable: what blocking calls are prohibited in async contexts, cancellation handling.
- Deadlock prevention conventions.
- How classes or functions are expected to declare their concurrency/thread-safety guarantees in documentation.
- The dominant concurrency libraries or frameworks in this ecosystem.

### Illustrative concepts *(research these, do not assume they apply)*

- Thread-safe collections vs. plain collections.
- Blocking calls inside async contexts.
- Lock-ordering rules for deadlock prevention.
- Actor model or channel-based communication vs. shared memory.

**Example (concurrency model varies widely — illustrative):**

```
// The right model depends entirely on the target language:
// Java:       threads + synchronized, CompletableFuture, virtual threads
// JavaScript: single-threaded event loop, async/await, no shared memory
// Go:         goroutines + channels (CSP)
// Rust:       ownership prevents data races at compile time
// Python:     GIL limits true parallelism; asyncio for I/O concurrency
// Research the idiomatic model for the target language
```

### Research Completion Checklist

- [ ] Language's native concurrency model documented.
- [ ] Shared mutable state conventions documented.
- [ ] Available synchronization primitives and their use cases documented.
- [ ] Async/await or equivalent conventions documented (if applicable).
- [ ] Thread-safety documentation conventions documented.
- [ ] Dominant concurrency libraries identified.

---

## 17. Governance and Standard Updates

Covers how a standard produced from this library is maintained and evolved over time.

### Research Objectives

Research and document:

- How official language specifications and style guides are maintained and updated (governance model, versioning, change process).
- How the language community handles breaking changes, deprecations, and migration paths (e.g., Python 2→3, Java LTS, Rust editions).
- Patterns for deprecating rules in a standard: how to mark a rule as deprecated, communication, and enforcement phase-out.
- Backward compatibility policies recognized in this ecosystem.
- Migration path documentation conventions.
- Change impact assessment practices.
- How standards bodies or community groups announce and discuss changes (RFCs, PEPs, JEPs, GitHub Discussions, etc.).
- Historical tracking of removed or changed rules — conventions for maintaining a changelog.

### Illustrative concepts *(research these, do not assume they apply)*

- Standard ownership (architect, lead, working group).
- RFC or proposal-based change process.
- Quarterly or semi-annual review cadence.
- Rule exception request process.
- Versioning the standard itself (with a changelog).
- Deprecation lifecycle: warn → soft-remove → hard-remove, with migration guidance at each stage.

**Example (rule exception request — illustrative template):**

```markdown
## Rule Exception Request
- Rule: <section.rule reference>
- File(s): <paths>
- Justification: <why this specific case warrants an exception>
- Mitigation: <what compensates for the exception>
- Approved by: <owner> — Date: <yyyy-mm-dd> — Expires: <date or "permanent">
```

**Example (deprecation lifecycle — illustrative):**

```
Phase 1 – Warn:        rule still enforced; tooling emits a deprecation warning
Phase 2 – Soft remove: rule removed from the standard; tooling warning remains
Phase 3 – Hard remove: tooling warning removed; rule considered historical
```

### Research Completion Checklist

- [ ] Language/ecosystem governance model documented (how official specs evolve).
- [ ] Community change process documented (RFCs, PEPs, JEPs, or equivalent).
- [ ] Breaking change and deprecation conventions documented.
- [ ] Migration path documentation conventions documented.
- [ ] Changelog conventions documented.

---

## 18. Ecosystem Conventions

Covers conventions that are specific to the language's ecosystem and do not fit naturally into the sections above. These are often as important as language syntax rules but are frequently overlooked in generic standards.

### Research Objectives

Research and document:

- Package manager conventions specific to this language: naming, publishing, metadata (e.g., `package.json`, `Cargo.toml`, `pyproject.toml`, `composer.json`, `pom.xml`).
- The language's own standards/specification process and its outputs (e.g., Python PEPs including PEP 8 and PEP 257, PHP PSRs, Java JSRs/JEPs, Rust RFCs, TC39 proposals for JavaScript).
- Module system and namespace conventions (how modules are named, structured, and published to a registry).
- Package naming conventions for published libraries.
- Workspace and monorepo tooling conventions specific to this ecosystem.
- Cross-language interop conventions if applicable (e.g., JVM interop, WASM, FFI).
- Community resources: the canonical Q&A site, official forum, major conference, primary community governance channel.
- Versioning conventions specific to this ecosystem (beyond SemVer if applicable).

### Illustrative concepts *(examples of ecosystem conventions by language — do not copy, research the target)*

- **Python:** PEP 8 (style), PEP 257 (docstrings), PEP 517/518 (build), `pyproject.toml`, `src` layout, virtual environments.
- **Go:** Module path conventions (e.g., `github.com/org/repo`), `go.mod`/`go.sum`, workspace mode.
- **Rust:** Crate naming (lowercase, underscores), `Cargo.toml` conventions, `crates.io` publishing.
- **Node.js/JavaScript:** `package.json` field conventions, npm/pnpm/yarn workspace conventions, ESM vs. CommonJS module system.
- **PHP:** PSR standards (PSR-1, PSR-2, PSR-4 autoloading, PSR-7, PSR-12), Composer conventions.
- **Java:** Package naming (reverse domain, e.g., `com.example.service`), Maven Central publishing, JCP/JEP process.

### Research Completion Checklist

- [ ] Package manager metadata file conventions documented.
- [ ] Language's own specification/standards process documented (PEPs, PSRs, JEPs, RFCs, etc.).
- [ ] Module and package naming conventions documented.
- [ ] Workspace/monorepo tooling conventions documented (if applicable).
- [ ] Published package conventions documented (registry, naming, metadata).
- [ ] Key community resources identified (official forum, primary registry, governance channel).

---

## 19. Development Toolchain

Covers the standard tools used day-to-day to develop in this language: the compiler or interpreter, the runtime, the SDK, and supporting tools. This is distinct from Sections 13 (quality enforcement) and 14 (build/packaging).

### Research Objectives

Research and document:

- The compiler, interpreter, or transpiler: official name, version management, and how to install it.
- The runtime environment: what is needed to run the language (JVM, Node.js, CPython, .NET CLR, native binary, etc.).
- The official SDK or standard development kit and what it includes.
- The official or community-standard version manager for this language (e.g., `nvm`, `pyenv`, `sdkman`, `rustup`, `gvm`, `asdf`).
- The official REPL or interactive development environment (if one exists).
- Debugging tools: official debugger, IDE integration, remote debugging conventions.
- Official or community-standard language server (LSP) for IDE support.
- Recommended IDE or editor support: extensions, plugins, official recommendations.
- Official documentation tools (if not covered in Section 6).
- Any official CLI tools bundled with the language for scaffolding, formatting, testing, etc.

### Illustrative concepts *(examples — research for the target language)*

- Java: JDK version management (sdkman, Jabba), JVM flags, official debugger (JDWP), Language Server (Eclipse JDT).
- Python: `pyenv` for version management, `venv`/`virtualenv`, `pdb` debugger, Pylsp/Pyright.
- Go: single official toolchain (`go` command covers build, test, format, doc), `gopls` language server.
- Rust: `rustup` for toolchain management, `cargo` (also the build tool), `rust-analyzer` language server.
- Node.js: `nvm`/`fnm`, `node` + `npm`/`pnpm`, Chrome DevTools protocol for debugging.

### Research Completion Checklist

- [ ] Compiler/interpreter/runtime documented with version management tooling.
- [ ] Official SDK contents documented.
- [ ] Standard version manager identified and its conventions documented.
- [ ] Debugging tooling documented (local and remote).
- [ ] Official or community-standard language server identified.
- [ ] Recommended IDE/editor support and plugins documented.
- [ ] Official CLI tools bundled with the language documented.

---

> **Final note:** This document is a research guide, not a standard. Every bullet point and example above is a topic to investigate for the target language — not a rule to adopt. The language-specific documents produced by `/standard_research` are where the actual researched standards live.
