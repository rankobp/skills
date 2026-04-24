# Review Domain Definitions

This file defines the specialist review domains, when to deploy each, and the detailed brief each reviewer receives.

## Deployment Rules

| Domain | Deploy When | Sub-agent |
|--------|------------|-----------|
| Correctness | Always (if issue-driven) | `general` |
| Code Quality | Always | `general` |
| Security | Auth, API endpoints, user input, data handling, crypto changed | `general` |
| Performance | DB queries, data processing, loops, async code, collections changed | `general` |
| Concurrency | Async code, shared state, background workers, concurrent access | `general` |
| API Design | New/modified API endpoints, middleware, DI registration | `general` |
| Test Coverage | Any testable logic added/modified (nearly always) | `general` |
| Frontend | UI files changed (components, templates, styles, client-side logic) | `general` |
| Architecture | New modules, structural changes, cross-cutting concerns | `general` |

Minimum panel: 3 (correctness + code quality + test coverage for simple bugfixes).
Full-stack feature: 6-8 specialists.

---

## Domain Briefs

When spawning a reviewer, include their full brief from below in the sub-agent prompt. Each brief tells the reviewer what to look for, what to ignore, and how to judge severity.

---

### Correctness

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
- **Critical:** Code violates a stated acceptance criterion or produces wrong results for expected inputs.
- **Significant:** Edge case that could cause incorrect behavior in realistic scenarios.
- **Minor:** Cosmetic or unlikely edge case with minimal impact.

---

### Code Quality

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
- **Critical:** Duplicated logic across files, or code that is fundamentally misleading/hard to reason about.
- **Significant:** Violation of project conventions, significant dead code, unclear naming that obscures intent.
- **Minor:** Style inconsistencies, minor naming improvements, unnecessary complexity.

---

### Security

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
- **Critical:** Exploitable vulnerability (injection, auth bypass, data exposure).
- **Significant:** Weakness that could become exploitable under realistic conditions (missing validation on non-public endpoint, verbose error messages leaking internal state).
- **Minor:** Defense-in-depth improvement (adding rate limiting, tightening CORS).

---

### Performance

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
- **Critical:** Will cause visible degradation or failure under normal load (N+1 on a list endpoint, unbounded query).
- **Significant:** Will degrade under scale or is noticeably wasteful (missing index, unnecessary allocation in a hot path).
- **Minor:** Minor optimization opportunity with no current user impact.

---

### Concurrency

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
- **Critical:** Race condition or deadlock that will occur under normal usage.
- **Significant:** Concurrency issue that could manifest under realistic load or specific timing.
- **Minor:** Best practice improvement (adding cancellation/timeout handling where unlikely to matter).

---

### API Design

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
- **Critical:** API contract is wrong or misleading (wrong HTTP method, missing required field validation).
- **Significant:** Inconsistency with project API patterns, unclear naming, missing error handling.
- **Minor:** Documentation gap, minor naming improvement.

---

### Test Coverage

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
- **Critical:** No tests at all for critical business logic or a bug fix with no regression test.
- **Significant:** Missing edge case tests, over-mocked tests that don't verify real behavior, tests that only cover the happy path.
- **Minor:** Test naming inconsistency, minor test quality improvement.

---

### Frontend

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
- **Critical:** Broken UI for expected use cases, accessibility violation that blocks users.
- **Significant:** Inconsistency with project patterns, missing error/loading states, performance issue visible to users.
- **Minor:** Style inconsistency, minor accessibility improvement, minor refactor opportunity.

---

### Architecture

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
- **Critical:** Architectural boundary violation that will cause maintenance problems (e.g., direct DB access from UI layer).
- **Significant:** Inconsistency with established patterns, unnecessary coupling, premature abstraction.
- **Minor:** Minor organizational improvement, config that could be externalized.
