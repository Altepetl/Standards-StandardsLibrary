---
title: Programming Standard - Constituent Elements
status: draft
version: 0.1.0
created: 2026-07-05
updated: 2026-07-05
---

# Programming Standard - Constituent Elements

This document describes the fundamental elements that any robust programming standard must include. Its purpose is to serve as a base template for defining the rules, best practices, and processes that will ensure code quality, security, and maintainability across any software project.

It is the **research backbone** of this repository: when building the standards for a specific language, each section below should be researched for that language and materialized as a **separate document** (one file per section), following the repository's structure rules.

> **Note:** This file never leaves this repository. It is not copied into language directories or client projects — it is exclusively the guide used by the `/standard_research` command to elaborate a standard.

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

---

## 1. Scope and Purpose

Defines the "what", "why", and "for whom" of the standard.

- **Primary objective:** Unify criteria to reduce technical debt and facilitate teamwork.
- **Applicability:** Specify which languages, frameworks, modules, or architectural layers it applies to (e.g., back-end, front-end, scripts).
- **Lifecycle:** Indicate whether it applies only to new code, refactoring efforts, or also to legacy code maintenance.
- **External references:** Mention any base standards or guides being followed (e.g., MISRA, CERT, PSR-12, Google Style Guides).

**Example (scope declaration):**

```markdown
## Scope
- Applies to: all back-end services written in Java 17+ using Spring Boot.
- Does NOT apply to: legacy modules in `legacy/` (frozen, maintenance only).
- New code: mandatory. Refactored code: mandatory. Untouched legacy code: exempt.
- Based on: Google Java Style Guide + OWASP Secure Coding Practices.
```

---

## 2. Naming Conventions

Consistent naming is key to readability.

- **Classes and Types:** Use `PascalCase` (e.g., `UserService`).
- **Methods and Functions:** Use `camelCase` (e.g., `fetchData`).
- **Variables and Parameters:** Use `camelCase` or `snake_case` (define based on the language; e.g., `totalPrice`).
- **Constants and Enums:** Use `UPPERCASE_WITH_UNDERSCORES` (e.g., `MAX_RETRIES`).
- **Interfaces:** Prefix `I` (e.g., `IRepository`) or suffix `able` (e.g., `Cloneable`) according to language standards.
- **Files:** Rules for naming files (e.g., `class.extension`, `component.tsx`).
- **Private members:** Use an underscore `_` prefix (e.g., `_internalState`) if the language allows it.
- **Booleans:** Prefix with `is`, `has`, `can`, or `should` (e.g., `isActive`, `hasPermission`).
- **Semantic names:** Names must reveal intent; avoid abbreviations and single letters outside short loop indexes.

**Example (good vs. bad):**

```java
// ❌ Bad
int d;                        // meaningless
public class usr_svc { }      // wrong case, abbreviated
boolean flag;                 // what flag?

// ✅ Good
int daysSinceLastLogin;
public class UserService { }
boolean isEmailVerified;
```

---

## 3. Formatting and Structure Style

Aesthetic rules and code ordering.

- **Indentation:** Define the use of **Spaces** or **Tabs**, and the exact number (e.g., 2 or 4 spaces).
- **Maximum line length:** Character limit per line (e.g., 80, 100, or 120).
- **Braces:** Specify the style (e.g., **Allman** - brace on a new line, or **K&R** - brace on the same line).
- **Spacing:** Rules about spaces after `if`, `for`, `while`, and around binary operators.
- **Import/include order:** Group by type (native libraries, external, internal) and in alphabetical order.
- **File structure:** Define the internal order of a class (e.g., static attributes, instance attributes, constructor, public methods, private methods).

**Example (brace styles):**

```c
// K&R style (brace on the same line)
if (isValid) {
    process(data);
}

// Allman style (brace on a new line)
if (isValid)
{
    process(data);
}
```

**Example (import ordering):**

```python
# 1. Standard library
import os
import sys

# 2. Third-party packages
import requests

# 3. Internal modules
from app.services import user_service
```

---

## 4. Project Architecture and Directory Layout

Conventions about the architecture of the repository itself — where each piece of code lives. (Section 3 covers the internal structure of a file; this section covers the structure of the project.)

- **Canonical root layout:** Define the standard top-level folders and their purpose (e.g., `src/`, `tests/`, `docs/`, `scripts/`, `config/`).
- **Ecosystem idioms:** Follow the recognized layout of the language's ecosystem (e.g., `cmd/`, `internal/`, `pkg/` in Go; src-layout in Python; Maven/Gradle layout in Java; `src/` + `dist/` in Node.js).
- **Layer boundaries:** Define where each architectural layer lives (domain, application, infrastructure, presentation) and the allowed **dependency direction** between them (e.g., domain must not import infrastructure).
- **Package organization strategy:** Choose and enforce one approach — *by feature* (`orders/`, `users/`) or *by layer* (`controllers/`, `services/`, `repositories/`) — and prohibit mixing them.
- **Test placement:** Decide between a mirrored test tree (`tests/` reflecting `src/`) or colocated tests (`file.test.ts` next to `file.ts`).
- **Generated artifacts:** Build outputs, caches, and generated code must never be committed; define the standard ignore rules (`.gitignore` baseline).
- **Configuration and assets:** Standard location for configuration files, static assets, migrations, and infrastructure definitions.

**Example (canonical layout, Go):**

```text
myservice/
├── cmd/myservice/main.go      # entry points only, no business logic
├── internal/                  # private application code
│   ├── domain/                # entities and business rules (imports nothing below)
│   ├── application/           # use cases
│   └── infrastructure/        # DB, HTTP clients, adapters
├── pkg/                       # public reusable libraries (only if truly reusable)
├── configs/
├── scripts/
└── go.mod
```

**Example (dependency direction rule):**

```text
✅ Allowed:    infrastructure → application → domain
❌ Forbidden:  domain → infrastructure (the core must not know the adapters)
```

**Example (by-feature vs. by-layer — pick ONE):**

```text
By feature (recommended for large domains):     By layer:
src/orders/{controller,service,repository}      src/controllers/orders...
src/users/{controller,service,repository}       src/services/orders...
```

---

## 5. Language Rules and Best Practices

Restrictions and recommendations on language usage.

- **Typing:** Specify whether strong/strict typing is required and when type inference is allowed.
- **Prohibited features:** List outdated, dangerous, or confusing constructs (e.g., `goto`, `eval`, `global` in PHP).
- **Memory management:** Guidelines to avoid leaks (in languages like C/C++).
- **Maximum complexity:** Limits on lines per function/method (e.g., max 30 lines), cyclomatic complexity (e.g., max 10), or nesting depth (e.g., max 4 levels of `if/else`).
- **Design principles:** Mandate the application of **DRY** (Don't Repeat Yourself), **KISS** (Keep It Simple, Stupid), and **SOLID** principles where possible.
- **Use of patterns:** Recommend specific design patterns for common problems (e.g., Factory, Repository, Singleton).
- **Immutability:** Prefer immutable data structures and `final`/`const`/`readonly` declarations where the language supports them.

**Example (reducing nesting with guard clauses):**

```javascript
// ❌ Bad: deep nesting
function processOrder(order) {
  if (order) {
    if (order.isPaid) {
      if (order.items.length > 0) {
        ship(order);
      }
    }
  }
}

// ✅ Good: guard clauses (fail fast)
function processOrder(order) {
  if (!order) return;
  if (!order.isPaid) return;
  if (order.items.length === 0) return;
  ship(order);
}
```

---

## 6. Documentation and Comments

Guidelines for writing useful documentation that complements self-documenting code.

- **DocBlocks / PHPDoc / JSDoc:** Mandatory template for classes, methods, and functions (including `@param`, `@return`, `@throws`).
- **Inline comments:** When to use them (explain the *why*, not the *what*).
- **Block comments:** Use to explain complex algorithms or critical business logic.
- **Markers:** Establish rules for `TODO` and `FIXME` (e.g., include the author's name and date, or a ticket reference).
- **README:** Require each module or component to have an entry file describing its purpose.

**Example (DocBlock template):**

```java
/**
 * Calculates the final price applying taxes and discounts.
 *
 * @param basePrice   the price before adjustments; must be >= 0
 * @param discountPct discount percentage between 0 and 100
 * @return the final price, never negative
 * @throws IllegalArgumentException if basePrice is negative
 */
public BigDecimal calculateFinalPrice(BigDecimal basePrice, int discountPct) { ... }
```

**Example (comment the *why*, not the *what*):**

```python
# ❌ Bad: repeats what the code says
counter += 1  # increment counter

# ✅ Good: explains the reason
# The external API is 1-indexed, so we offset before sending.
counter += 1
```

**Example (TODO marker rule):**

```java
// TODO(jperez, 2026-07-05, JIRA-1234): remove this fallback once API v2 is live
```

---

## 7. Error Handling, Exceptions, and Logging

Policies for managing anomalous situations.

- **Exception hierarchy:** Define custom exceptions (e.g., `DomainException`, `InfrastructureException`).
- **Recovery policy:** Decide between *Fail-Fast* or *Resilience* (retries, circuit breakers).
- **Input validation:** Mandate validation of data coming from users, APIs, or databases before processing.
- **Logging:** Establish log levels (`DEBUG`, `INFO`, `WARN`, `ERROR`) and what information should be logged at each level (e.g., request traces, response times).
- **Global error handling:** Implement a centralized handler to catch unhandled exceptions and return standardized responses (for APIs).
- **Never swallow exceptions:** Prohibit empty `catch` blocks; every caught exception must be handled, logged, or rethrown.
- **Observability:** Complement logs with metrics and distributed tracing when the architecture requires it (e.g., correlation IDs across services).

**Example (empty catch prohibition):**

```java
// ❌ Bad: the error disappears silently
try {
    repository.save(user);
} catch (Exception e) { }

// ✅ Good: contextualize and rethrow as a domain exception
try {
    repository.save(user);
} catch (SQLException e) {
    log.error("Failed to persist user {}", user.getId(), e);
    throw new InfrastructureException("User persistence failed", e);
}
```

**Example (log level usage):**

| Level | Use for | Example |
|-------|---------|---------|
| `DEBUG` | Diagnostic detail, disabled in production | `"Cache miss for key user:42"` |
| `INFO`  | Normal business events | `"Order 1234 created"` |
| `WARN`  | Recoverable anomalies | `"Retry 2/3 calling payment API"` |
| `ERROR` | Failures requiring attention | `"Payment API unreachable after 3 retries"` |

---

## 8. API Design and Data Contracts

Rules about how data is exposed to the outside world. (Section 7 covers *internal* error handling; this section covers the *external* contract.) Essential when the language is used to build web services.

- **Resource naming:** Use plural nouns and kebab-case in URLs; no verbs in paths (the HTTP method is the verb).
- **Payload field casing:** Choose **one** convention for JSON/response bodies (`camelCase` or `snake_case`) and enforce it across all services, regardless of the internal language convention.
- **Versioning strategy:** Define how APIs are versioned (URI: `/api/v1/...`, or headers) and the deprecation policy for old versions.
- **HTTP status codes:** Standardize which codes are returned for each situation (e.g., `400` validation, `401` unauthenticated, `403` unauthorized, `404` not found, `409` conflict, `422` semantic error, `500` unexpected).
- **Standard error body:** Define a single error response format for all services (e.g., RFC 7807 `application/problem+json`), including a machine-readable error code and a correlation ID.
- **Pagination, filtering, and sorting:** Fixed conventions (e.g., `?page=`, `?size=`, `?sort=field,desc`) and a mandatory maximum page size.
- **Contract-first:** Require the contract (OpenAPI, Protobuf, GraphQL schema) to be the source of truth, versioned in the repository.
- **Backward compatibility:** Only additive changes within a version; removing or renaming a field requires a new version.
- **Idempotency:** Require idempotency keys for operations that clients may retry (e.g., payments).

**Example (endpoint naming):**

```text
❌ Bad:  POST /api/createUser          GET /api/get_user_orders?id=42
✅ Good: POST /api/v1/users            GET /api/v1/users/42/orders
```

**Example (standard error body, RFC 7807 style):**

```json
{
  "type": "https://api.example.com/errors/insufficient-funds",
  "title": "Insufficient funds",
  "status": 422,
  "detail": "Account 42 balance is 12.50, transfer requires 100.00",
  "instance": "/api/v1/transfers/9083",
  "correlationId": "b7ad6b7169203331"
}
```

**Example (backward compatibility):**

```text
✅ Allowed in v1:  adding the optional field "nickname" to the User response
❌ Requires v2:    renaming "email" to "emailAddress", or making "phone" mandatory
```

---

## 9. Security and Privacy

Guidelines to prevent vulnerabilities.

- **Code injection:** Use parameterized queries (SQL) and output escaping (XSS).
- **Secrets management:** Prohibit hardcoding passwords, tokens, or API keys; use environment variables or secret managers (e.g., Vault).
- **Authentication and Authorization:** Define how access controls should be implemented (JWT, OAuth, roles).
- **Encryption:** Standards for encrypting sensitive data at rest and in transit.
- **File validation:** Restrictions on type, size, and content for file uploads.
- **Sensitive data in logs:** Prohibit logging passwords, tokens, or personally identifiable information (PII); mask when unavoidable.

**Example (SQL injection prevention):**

```python
# ❌ Bad: string concatenation, vulnerable to injection
cursor.execute("SELECT * FROM users WHERE email = '" + email + "'")

# ✅ Good: parameterized query
cursor.execute("SELECT * FROM users WHERE email = %s", (email,))
```

**Example (secrets management):**

```javascript
// ❌ Bad: hardcoded secret
const apiKey = "sk-live-9f8a7b6c5d";

// ✅ Good: environment variable
const apiKey = process.env.PAYMENT_API_KEY;
```

---

## 10. Dependency Management

Control over external libraries and packages.

- **Versioning:** Pin exact versions or allow specific semantic versioning (SemVer) ranges.
- **Updates:** Establish a frequency for reviewing and updating outdated or vulnerable dependencies.
- **Official repositories:** Only use trusted sources (e.g., npmjs, Maven Central, Packagist).
- **Auditing:** Mandate vulnerability scanning on dependencies (e.g., `npm audit`, `OWASP Dependency Check`).
- **Justification for new dependencies:** Require an evaluation (license, maintenance activity, size) before adding a new library.

**Example (SemVer range policy):**

```json
{
  "dependencies": {
    "express": "4.19.2",     // ✅ exact pin for critical runtime deps
    "lodash": "^4.17.21"     // ✅ minor/patch range allowed for utilities
  }
}
```

---

## 11. Testing and Quality Validation

Criteria to ensure the code works as expected.

- **Code coverage:** Set a minimum percentage for line and branch coverage (e.g., > 80%).
- **Unit tests:** Mandatory structure (AAA Pattern: *Arrange, Act, Assert*).
- **Integration tests:** Define which modules or external services must be tested together.
- **Test naming:** Rules for naming test methods (e.g., `whenDoingX_GivenConditionY_ThenResultZ`).
- **Static analysis:** Pass all rules from quality tools before merging.
- **Test independence:** Each test must be runnable in isolation and in any order (no shared mutable state).

**Example (AAA pattern and naming):**

```java
@Test
void whenApplyingDiscount_givenPercentageAbove100_thenThrowsException() {
    // Arrange
    PriceCalculator calculator = new PriceCalculator();

    // Act & Assert
    assertThrows(IllegalArgumentException.class,
        () -> calculator.calculateFinalPrice(BigDecimal.TEN, 150));
}
```

---

## 12. Version Control and Workflow

Rules regarding the use of Git (or similar).

- **Branching strategy:** Choose between GitFlow, GitHub Flow, or Trunk-Based Development.
- **Branch naming:** Define a convention (e.g., `feature/JIRA-123-user-login`, `hotfix/payment-timeout`).
- **Commit messages:** Adopt a standard like **Conventional Commits** (e.g., `feat:`, `fix:`, `docs:`, `chore:`).
- **Pull Requests (PRs) / Merge Requests (MRs):** Minimum requirements to open a PR (e.g., passing CI, at least 1 approval).
- **Code review checklist:** Define what reviewers must verify (correctness, tests, security, adherence to this standard) and expected response time.
- **Commit frequency:** Encourage atomic and frequent commits.

**Example (Conventional Commits):**

```text
feat(auth): add refresh token rotation
fix(orders): prevent duplicate shipment on retry
docs(readme): document local setup with Docker
chore(deps): bump spring-boot to 3.3.1
```

**Example (minimal review checklist):**

```markdown
- [ ] The change does what the ticket describes
- [ ] New/changed logic is covered by tests
- [ ] No secrets, credentials, or PII introduced
- [ ] Naming and formatting follow the standard
- [ ] No commented-out or dead code left behind
```

---

## 13. Automation Tools (Linters and Formatters)

Mandatory software to enforce the standard automatically.

- **Linters:** Language-specific tools (e.g., ESLint, Pylint, Rubocop, Checkstyle).
- **Formatters:** Use of tools like Prettier, Black, or gofmt to apply formatting automatically.
- **Static Application Security Testing (SAST) tools:** Integrate SonarQube, CodeQL, or similar to detect code smells and vulnerabilities.
- **CI/CD integration:** Define that the pipeline must fail if automated validations are not passed.
- **Pre-commit hooks:** Run formatters and fast linters locally before each commit (e.g., Husky, pre-commit).
- **Shared configuration:** Linter/formatter config files must live in the repository so every developer and CI job uses identical rules.

**Example (pipeline quality gate):**

```yaml
# ci.yml (excerpt)
quality:
  steps:
    - run: npm run lint          # fails the build on lint errors
    - run: npm run test -- --coverage
    - run: npm audit --audit-level=high
```

---

## 14. Build, Packaging, and Containerization

Rules about how the final artifact is compiled, packaged, and shipped. (Section 13 covers quality gates in CI; this section covers producing the deliverable itself.)

- **Reproducible builds:** Mandate lockfiles and pinned toolchain versions (compiler, SDK, runtime) so any machine produces the same artifact.
- **Compiler/build flags:** Define the standard flags per environment (e.g., warnings-as-errors in CI, optimization level for release, debug symbols policy).
- **Single build entry point:** One standardized command to build the project (e.g., `make build`, `./gradlew build`, `npm run build`) — no tribal knowledge of manual steps.
- **Artifact naming and versioning:** Convention for artifact names (e.g., `service-name-1.4.2+gitsha.jar`) and where they are published (artifact registry).
- **Environment configuration:** Follow 12-factor principles — configuration comes from the environment; one build serves all environments (never compile per environment); `.env` files are never committed.
- **Containerization:** Define the approved base images for the language (and their update policy), mandate multi-stage builds, run as non-root user, and require image vulnerability scanning.
- **Artifact integrity:** Sign artifacts/images and generate an SBOM (software bill of materials) when the criticality of the system requires it.

**Example (compiler flags policy):**

```text
CI / release builds:   -Wall -Wextra -Werror  (all warnings are errors)
Local development:     -Wall -Wextra          (warnings visible, not blocking)
```

**Example (multi-stage, non-root Dockerfile):**

```dockerfile
# ✅ Build stage: full toolchain
FROM golang:1.24 AS build
WORKDIR /app
COPY . .
RUN CGO_ENABLED=0 go build -o /service ./cmd/myservice

# ✅ Runtime stage: minimal image, non-root
FROM gcr.io/distroless/static:nonroot
COPY --from=build /service /service
USER nonroot
ENTRYPOINT ["/service"]
```

**Example (environment configuration):**

```text
❌ Bad:  building myapp-prod.jar and myapp-qa.jar with different embedded configs
✅ Good: one myapp-1.4.2.jar; DATABASE_URL and LOG_LEVEL injected by the environment
```

---

## 15. Performance and Resource Efficiency

Rules to prevent avoidable performance problems from reaching production.

- **Algorithmic complexity:** Flag obviously inefficient patterns (e.g., nested loops over large collections when a lookup structure would do).
- **Database access:** Prohibit N+1 query patterns; require pagination for unbounded result sets; define indexing expectations.
- **Resource lifecycle:** Mandate explicit release of connections, files, and streams (e.g., `try-with-resources`, `using`, context managers).
- **Caching policy:** Define when caching is expected, which layers own it, and invalidation rules.
- **Premature optimization:** Prohibit micro-optimizations that harm readability without measured evidence; require profiling data to justify complex optimizations.
- **Budgets:** Optionally set measurable budgets (e.g., p95 latency, bundle size for front-end, memory ceilings).

**Example (resource lifecycle):**

```java
// ❌ Bad: connection may leak if an exception occurs
Connection conn = dataSource.getConnection();
runQuery(conn);
conn.close();

// ✅ Good: automatic release
try (Connection conn = dataSource.getConnection()) {
    runQuery(conn);
}
```

**Example (N+1 query):**

```text
❌ Bad:  1 query for orders + 1 query per order for its customer (N+1)
✅ Good: 1 query joining orders with customers, or a batched IN (...) fetch
```

---

## 16. Concurrency and Thread Safety

Rules for code that runs in parallel or asynchronously (critical in languages such as Java, C#, Go, Rust, and in async runtimes like Node.js or Python asyncio).

- **Shared state:** Minimize shared mutable state; prefer immutability and message passing.
- **Synchronization primitives:** Define which mechanisms are approved (locks, channels, atomics) and which are discouraged.
- **Thread-safe collections:** Mandate concurrent-safe structures when state must be shared (e.g., `ConcurrentHashMap`).
- **Async conventions:** Rules for `async/await` usage, prohibition of blocking calls inside async contexts, and cancellation handling.
- **Deadlock prevention:** Establish lock-ordering rules and timeouts.
- **Documentation:** Require classes/functions to declare their thread-safety guarantees.

**Example (shared mutable state):**

```java
// ❌ Bad: race condition on a plain HashMap accessed by multiple threads
Map<String, Integer> counters = new HashMap<>();

// ✅ Good: thread-safe structure with atomic update
Map<String, Integer> counters = new ConcurrentHashMap<>();
counters.merge(key, 1, Integer::sum);
```

**Example (blocking inside async):**

```javascript
// ❌ Bad: synchronous file read blocks the event loop
const data = fs.readFileSync(path);

// ✅ Good: non-blocking equivalent
const data = await fs.promises.readFile(path);
```

---

## 17. Governance and Standard Updates

How the standard is kept alive over time.

- **Standard owner:** Designate a responsible person (architect, lead developer) to resolve questions and approve changes.
- **Modification process:** Establish how new rules are proposed and discussed (e.g., via RFCs or issues in the repository).
- **Review frequency:** Define a schedule for reviewing and updating the document (e.g., quarterly or semi-annually).
- **Exceptions:** Document the process for requesting a justified exception to a specific rule.
- **Versioning of the standard:** Version the standard itself (e.g., SemVer) and keep a changelog so teams know what changed and when.

**Example (exception request template):**

```markdown
## Rule Exception Request
- Rule: 5.3 "Max 30 lines per function"
- File(s): src/parser/tokenizer.c
- Justification: the state machine is clearer as a single 85-line switch.
- Mitigation: extensive block comments + 100% branch coverage.
- Approved by: <standard owner> — Date: <yyyy-mm-dd> — Expires: <yyyy-mm-dd or "permanent">
```

---

> **Final note:** This document is a template. It is recommended to adjust the depth of each section to the size of the team and the criticality of the system being developed. The successful implementation of a standard depends as much on its clarity as on the automation of its enforcement.
