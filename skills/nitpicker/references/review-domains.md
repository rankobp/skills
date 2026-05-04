# Review Domain Definitions

This file defines the specialist review domains, when to deploy each, and the detailed brief each reviewer receives.

## Severity Scale

All reviewers use this 5-level severity scale:

| Level | Label | Gate Behavior |
|-------|-------|---------------|
| 4 | **FATAL** | Hard gate — strongly recommend fix or abort before further review |
| 3 | **ERROR** | Gate — ask user; recommend fix before continuing |
| 2 | **WARNING** | Gate — ask user whether to fix or continue |
| 1 | **INFO** | No gate — continue automatically |
| 0 | **PASSED** | No issues found — continue automatically |

Only FATAL/ERROR/WARNING trigger gates. INFO and PASSED flow through without stopping.

## Deployment Rules

| Domain | Tier | Deploy When | Sub-agent |
|--------|------|-------------|-----------|
| Correctness | 1 | Always (if issue-driven) | `general` |
| Security | 2 | Auth, API endpoints, user input, data handling, crypto changed | `general` |
| Architecture | 2 | New modules, structural changes, cross-cutting concerns | `general` |
| Code Quality | 3 | Always | `general` |
| Performance | 3 | DB queries, data processing, loops, async code, collections changed | `general` |
| Concurrency | 3 | Async code, shared state, background workers, concurrent access | `general` |
| API Design | 4 | New/modified API endpoints, middleware, DI registration | `general` |
| Test Coverage | 4 | Any testable logic added/modified (nearly always) | `general` |
| Frontend | 4 | UI files changed (components, templates, styles, client-side logic) | `general` |

### Tier Logic

Tiers run **sequentially** — Tier 1 completes before Tier 2 starts, etc. Within each tier, reviewers run per the user's execution mode (parallel batches or sequential).

**Why this order:**
- **Tier 1 (Correctness):** If the logic is fundamentally wrong, nothing else matters.
- **Tier 2 (Security, Architecture):** Structural integrity — safety and soundness.
- **Tier 3 (Code Quality, Performance, Concurrency):** Quality of implementation.
- **Tier 4 (API Design, Test Coverage, Frontend):** Polish and completeness.

Minimum panel: 3 (correctness + code quality + test coverage for simple bugfixes).
Full-stack feature: 6-8 specialists.

---

## Domain Briefs

When spawning a reviewer, include their full brief from below in the sub-agent prompt. Each brief tells the reviewer what to look for, what to ignore, and how to judge severity.

---

### Correctness (Tier 1)

**Trigger:** Always, when reviewing against a specific issue/ticket with acceptance criteria.

**Brief:**

You are reviewing for **functional correctness** — does the code do what the requirements say it should do?

Check:
- Does the implementation satisfy every acceptance criterion from the issue?
- Are edge cases handled (null/empty inputs, boundary values, error states)?
- Is the control flow correct — no off-by-one errors, no unreachable branches?
- Are return types and values correct? Do error paths return appropriate values?
- Are there logical contradictions or impossible states?
- If the code handles user-facing behavior, does it match expected UX?

Ignore: performance, style, test coverage, architecture — other reviewers own those.

**Severity guidelines:**
- **FATAL:** The entire approach is fundamentally flawed — the implementation cannot satisfy the requirements without major rethinking or rewriting.
- **ERROR:** Code violates a stated acceptance criterion or produces wrong results for expected inputs.
- **WARNING:** Edge case that could cause incorrect behavior in realistic scenarios.
- **INFO:** Cosmetic or unlikely edge case with minimal impact.

---

### Code Quality (Tier 3)

**Trigger:** Always.

**Brief:**

You are reviewing for **code quality and maintainability** — is this code clean, readable, and consistent with the project's conventions?

Check:
- Does the code follow existing patterns in the codebase (naming, structure, conventions)?
- Are functions/methods small and focused? Do they do one thing?
- Is there duplicated logic that should be extracted into a shared abstraction?
- Are variable and function names clear and self-documenting?
- Are magic numbers/strings replaced with named constants?
- Is there dead code (unused imports, unreachable logic, commented-out code)?
- Are error messages clear and actionable?
- Is the code consistent with the style in AGENTS.md (if present)?

Ignore: correctness of logic, test coverage, architecture decisions — other reviewers own those.

**Severity guidelines:**
- **FATAL:** Code is fundamentally misleading or impossible to reason about — the structure actively obscures what it does.
- **ERROR:** Duplicated logic across files, or code that is fundamentally misleading/hard to reason about.
- **WARNING:** Violation of project conventions, significant dead code, unclear naming that obscures intent.
- **INFO:** Style inconsistencies, minor naming improvements, unnecessary complexity.

---

### Security (Tier 2)

**Trigger:** Auth, API endpoints, user input handling, data processing, cryptographic operations, or any file that touches sensitive data.

**Brief:**

You are reviewing for **security vulnerabilities** — could an attacker exploit this code?

Check:
- Input validation: is all user input validated, sanitized, and bounds-checked?
- Authentication/authorization: are access controls properly enforced?
- Injection risks: SQL injection, XSS, command injection, path traversal?
- Data exposure: are sensitive fields (PII, credentials, tokens) logged, serialized, or exposed in error messages?
- Cryptographic misuse: weak algorithms, hardcoded keys, improper IV/nonce handling?
- Dependency risks: are new dependencies well-maintained and free of known CVEs?
- Secrets management: are API keys, connection strings, or credentials hardcoded?

Ignore: performance, style, test quality — other reviewers own those. Only flag code quality issues if they create a security risk.

**Severity guidelines:**
- **FATAL:** Exploitable vulnerability with severe impact (auth bypass, mass data exposure, RCE).
- **ERROR:** Exploitable vulnerability (injection, auth bypass, data exposure).
- **WARNING:** Weakness that could become exploitable under realistic conditions (missing validation on non-public endpoint, verbose error messages leaking internal state).
- **INFO:** Defense-in-depth improvement (adding rate limiting, tightening CORS).

---

### Performance (Tier 3)

**Trigger:** DB queries, data processing pipelines, loops over collections, async code, caching logic, or any file in a hot path.

**Brief:**

You are reviewing for **performance** — will this code perform acceptably at realistic scale?

Check:
- N+1 query patterns or missing eager loading?
- Unnecessary loops, repeated computations, or redundant allocations?
- Missing or incorrect use of caching?
- Blocking on async operations (`.Result`, `.Wait()`, `.get()` on unresolved promises, `sync over async`)?
- Large collection operations that could be streamed or paginated?
- Missing database indexes for new query patterns?
- Memory pressure from large object allocations or unmanaged resource leaks?

Judge against realistic scale, not theoretical limits. A loop over 10 items is fine. A loop over 100K items that does a DB call per iteration is not.

Ignore: code style, test coverage, architecture — other reviewers own those.

**Severity guidelines:**
- **FATAL:** Will cause system failure or complete unresponsiveness under normal load.
- **ERROR:** Will cause visible degradation or failure under normal load (N+1 on a list endpoint, unbounded query).
- **WARNING:** Will degrade under scale or is noticeably wasteful (missing index, unnecessary allocation in a hot path).
- **INFO:** Minor optimization opportunity with no current user impact.

---

### Concurrency (Tier 3)

**Trigger:** Async/await code, shared mutable state, background workers, concurrent collections, locks, or any code that runs in a multi-threaded context.

**Brief:**

You are reviewing for **concurrency bugs** — race conditions, deadlocks, thread safety issues.

Check:
- Shared mutable state accessed without synchronization?
- Race conditions in check-then-act patterns?
- Deadlock risks (lock ordering, mixed sync/async, holding locks across await/yield points)?
- Cancellation/timeout propagation — are cancellation signals passed through and respected?
- Thread-safe collection misuse (enumerating while modifying)?
- Static mutable state that could cause cross-request contamination?
- Fire-and-forget tasks with no error handling?

Ignore: general async best practices that don't involve correctness (those belong to performance or code quality).

**Severity guidelines:**
- **FATAL:** Race condition or deadlock that will cause data corruption or system failure under normal usage.
- **ERROR:** Race condition or deadlock that will occur under normal usage.
- **WARNING:** Concurrency issue that could manifest under realistic load or specific timing.
- **INFO:** Best practice improvement (adding cancellation/timeout handling where unlikely to matter).

---

### API Design (Tier 4)

**Trigger:** New or modified API endpoints, middleware, DI registration, or public interface changes.

**Brief:**

You are reviewing for **API design quality** — is the API intuitive, consistent, and well-structured?

Check:
- Are endpoint URLs RESTful and consistent with existing routes?
- Are HTTP methods used correctly (GET for reads, POST/PUT/PATCH for writes)?
- Are request/response DTOs well-shaped — flat where possible, properly typed?
- Is error handling consistent with the project's error response format?
- Are validation rules complete and return clear error messages?
- Are endpoints properly documented (OpenAPI/Swagger annotations if the project uses them)?
- Is dependency injection/service registration correct (lifetimes/scope match the service's needs)?
- Are new public methods on existing services following the existing interface patterns?

Ignore: security of endpoints (security reviewer), performance of handlers (performance reviewer), test coverage.

**Severity guidelines:**
- **FATAL:** API contract is fundamentally broken or dangerous (public endpoint with no auth, wrong HTTP method causing data loss).
- **ERROR:** API contract is wrong or misleading (wrong HTTP method, missing required field validation).
- **WARNING:** Inconsistency with project API patterns, unclear naming, missing error handling.
- **INFO:** Documentation gap, minor naming improvement.

---

### Test Coverage (Tier 4)

**Trigger:** Any testable logic was added or modified (nearly always deploy this reviewer).

**Brief:**

You are reviewing for **test coverage and quality** — are the changes adequately tested?

Check:
- Are there tests for the new/modified logic?
- Do tests cover the happy path AND error/edge cases?
- Are tests independent (no coupling between test cases)?
- Do tests assert meaningful outcomes, not just "no exception thrown"?
- Are mocks used appropriately (not over-mocked to the point of testing nothing)?
- Is the test naming convention consistent with the project's existing tests?
- Are integration tests needed where unit tests alone are insufficient?
- If the change fixes a bug, is there a regression test?

Compare against the project's existing test patterns and conventions. If the project has no tests at all, flag this as significant rather than critical (it's a broader issue).

Ignore: code correctness, performance, style — other reviewers own those.

**Severity guidelines:**
- **FATAL:** Critical business logic with zero tests AND the implementation has known correctness issues.
- **ERROR:** No tests at all for critical business logic or a bug fix with no regression test.
- **WARNING:** Missing edge case tests, over-mocked tests that don't verify real behavior, tests that only cover the happy path.
- **INFO:** Test naming inconsistency, minor test quality improvement.

---

### Frontend (Tier 4)

**Trigger:** UI files changed — TypeScript/JavaScript, JSX/TSX, HTML, CSS/SCSS, component files, or any framework-specific template files.

**Brief:**

You are reviewing for **frontend quality** — are the UI changes well-implemented?

Check:
- Is the component structure consistent with the project's existing patterns?
- Is state management correct (local vs global, proper initialization, cleanup)?
- Are there accessibility issues (missing ARIA labels, keyboard navigation, color contrast)?
- Is the CSS/scoping approach consistent with the project (CSS modules, Tailwind, styled-components, etc.)?
- Are there re-rendering or performance concerns (missing memoization, unnecessary state updates)?
- Is error handling present for async operations (loading states, error states)?
- Are forms handling validation, submission, and disabled states correctly?
- Are there hardcoded strings that should be internationalized (if the project uses i18n)?

Ignore: backend logic, API design, security of endpoints — other reviewers own those.

**Severity guidelines:**
- **FATAL:** UI is completely broken or inaccessible for all users.
- **ERROR:** Broken UI for expected use cases, accessibility violation that blocks users.
- **WARNING:** Inconsistency with project patterns, missing error/loading states, performance issue visible to users.
- **INFO:** Style inconsistency, minor accessibility improvement, minor refactor opportunity.

---

### Architecture (Tier 2)

**Trigger:** New modules/projects added, structural changes, cross-cutting concerns modified, or changes to dependency injection/configuration/startup.

**Brief:**

You are reviewing for **architectural soundness** — does this change fit the project's architecture?

Check:
- Does the change follow the project's established layering/module boundaries?
- Are new dependencies between modules appropriate, or do they introduce coupling?
- Is dependency injection used correctly (appropriate lifetimes, no service locator anti-pattern)?
- Are cross-cutting concerns (logging, error handling, caching) handled consistently with the existing approach?
- If new abstractions are introduced, are they justified (not over-abstracted for a single implementation)?
- Does the change respect the project's folder/namespace organization?
- Are configuration values externalized (not hardcoded)?

Compare against the project's existing architecture, not an idealized clean architecture. Consistency matters more than theoretical purity.

Ignore: code style details, individual test quality, minor performance — other reviewers own those.

**Severity guidelines:**
- **FATAL:** Architectural change that will cause cascading problems across the system (e.g., bypassing a security layer entirely).
- **ERROR:** Architectural boundary violation that will cause maintenance problems (e.g., direct DB access from UI layer).
- **WARNING:** Inconsistency with established patterns, unnecessary coupling, premature abstraction.
- **INFO:** Minor organizational improvement, config that could be externalized.
