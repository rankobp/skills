---
name: nitpicker
description: >
  Multi-agent post-implementation code review with severity-gated progressive review. Deploys
  specialist sub-agents in tiered batches — each tier must pass a gate check before the next tier
  runs. Domains: correctness, security, performance, concurrency, code quality, API design,
  test coverage, frontend, architecture. The coordinator merges reports into a prioritized work plan,
  presents it for user approval, then dispatches fixes in parallel across specialist sub-agents.
  Use this skill after completing an implementation task — whether it's a feature, bugfix,
  refactoring, or any code change. Triggers on: "review the changes", "review my code",
  "check correctness", "code review", "nitpick this", "roast my code", "what's wrong with this",
  or after any implementation is completed. Also triggers when the user asks to verify, validate,
  quality-check, or critique completed work.
license: MIT
metadata:
  author: rankobp
  version: "0.4.0"
---

# Nitpicker

This skill orchestrates a multi-agent code review pipeline with **severity-gated progressive review**:

1. **Scope & Gather** — determine what to review and collect context
2. **Deploy Review Panel (Tiered)** — spawn specialist sub-agents in priority tiers, with gate checks between tiers
3. **Merge & Organize** — collect reports, deduplicate, build a structured work plan
4. **Approve** — present the plan to the user for sign-off
5. **Execute** — dispatch approved fixes in parallel via specialist sub-agents

The coordinator (you, the agent using this skill) never reviews code directly. You orchestrate.

## When to Use

- After completing any implementation task (feature, bugfix, refactoring)
- When the user asks to review, verify, validate, or quality-check code
- When the user says "nitpick this", "review my code", "roast my code", "what's wrong"
- Before committing or creating a PR for significant changes
- Do NOT use for trivial one-line changes where the issue is obvious

## Severity Scale

All reviewers use a 5-level severity scale. Only WARNING and above trigger gates:

| Level | Label | Gate Behavior |
|-------|-------|---------------|
| 4 | **FATAL** | Hard gate — strongly recommend fix or abort before further review |
| 3 | **ERROR** | Gate — ask user; recommend fix before continuing |
| 2 | **WARNING** | Gate — ask user whether to fix or continue |
| 1 | **INFO** | No gate — continue automatically |
| 0 | **PASSED** | No issues found — continue automatically |

## Review Tiers

Reviewers run in priority tiers. Tiers are sequential — each tier must complete (and pass gate checks) before the next tier starts. Within a tier, reviewers run per the user's execution mode (parallel batches or sequential).

| Tier | Domains | Rationale |
|------|---------|-----------|
| 1 | Correctness | If the logic is fundamentally wrong, nothing else matters |
| 2 | Security, Architecture | Structural integrity — safety and soundness |
| 3 | Code Quality, Performance, Concurrency | Quality of implementation |
| 4 | API Design, Test Coverage, Frontend | Polish and completeness |

Not all tiers have all domains. The deployment rules in `references/review-domains.md` determine which domains within each tier are relevant based on what changed.

## Execution Mode

Different users run on different setups — some have rock-solid connections, others hit drops with parallel sub-agents. **Ask the user once at the start of every review session** (right after Phase 1, before Phase 2):

> "How should I run the review panel?
> 1. **Parallel** — spawn 2-3 sub-agents per turn (faster, but may stall on unstable connections)
> 2. **Sequential** — one sub-agent at a time (slower, but rock-solid)"

Store the answer and respect it throughout the entire review lifecycle. If the user doesn't answer explicitly, **default to sequential** — it's the safe choice.

The execution mode affects ONLY how many sub-agents you spawn per turn within a tier. It does NOT change the review content, domains, or output format. Tiers are always sequential regardless of execution mode.

### Parallel Mode

Spawn 2-3 sub-agents per turn. Acknowledge progress between batches.

- Tier deployment: Spawn reviewers in batches of 2-3 per turn within each tier.
- Fix execution: Spawn fix agents 2-3 per turn per parallelization group.
- Verification: Single agent — no batching needed.

### Sequential Mode

Spawn exactly ONE sub-agent per turn. Wait for it to complete before spawning the next.

- Tier deployment: One reviewer per turn within each tier.
- Fix execution: One fix agent per turn.
- Verification: Single agent — same for both modes.

## Connection Stability

API providers drop connections when the model generates large tool call argument payloads in a single batch. The connection stability limit is about **tool call payload SIZE**, not about thinking too long. A single `task` spawn with a 2000-word prompt is fine. The danger is spawning 6+ sub-agents with 5000-word prompts each in a single turn.

**The most common failure mode is NOT connection drops — it's the coordinator stalling in Phase 2, reading "just one more file" before deploying reviewers. Resist this. The diff is enough. Deploy reviewers.**

**Hard limits — regardless of execution mode:**
- **Keep prompts lean** — don't include full file contents in every reviewer prompt. Partition by domain relevance. Give each reviewer only the files relevant to their specialty plus a summary of the full change
- **Break complex tasks into steps** — for multi-file changes, acknowledge progress between turns

**What this means for each phase:**
- Phase 1 (Scope): Fetch issue details in ONE call. Do NOT read issue comments separately — get them all in one go.
- Phase 2 (Gather): ONE turn max. Fetch the diff + file list. That's it. Do NOT read entity files, reference files, or upstream dependencies here.

**Between steps, briefly tell the user what's happening** — e.g., "Tier 1 reviewer deployed (Correctness), waiting..." or "Tier 2, batch 1/2 deployed, waiting..." This prevents the appearance of stalling.

## Phase 1: Determine Review Scope

Before deploying reviewers, figure out what this review is against:

1. **Check for a PR.** If the user mentions a PR number or URL, use `github_pull_request_read` with `get_diff` and `get_files` to get the changes directly — skip the git-based diff collection in Phase 2.
2. **Check if the issue context is clear.** Look at the conversation — was an issue number mentioned? Is there a clear description of what was implemented? Can you infer from the branch name?
3. **If you are NOT 100% clear on the requirements**, ask:
   > "I'm not sure which issue/ticket this is for. General code quality review, or can you provide an issue number so I can review against specific requirements?"
4. **If the user provides an issue number** (or you can identify it confidently), fetch details:
   - **GitHub**: Use `github_issue_read` with the issue number and `owner`/`repo` from AGENTS.md
   - **Other platforms**: Use whatever issue-tracking tools are available in the current environment. If no suitable tool exists, ask the user to paste the issue details.
5. **If general quality review**, skip requirements-checking and focus on quality domains.

Never guess at requirements. When in doubt, ask.

## Phase 2: Gather Context

Collect everything the review panel needs:

**Get the diff:**

If reviewing a PR, you already have the diff from Phase 1. Otherwise:

1. Determine base branch (`origin/master` or `origin/main` — check `git remote`).
2. `git fetch origin` to get latest base state.
3. `git diff origin/<base>...HEAD` — triple-dot gives only branch changes.
4. `git diff --name-only origin/<base>...HEAD` — list of changed files.
5. **If the diff is empty**, try these fallbacks in order:
   - `git diff HEAD~1...HEAD` (last commit only)
   - `git diff --cached` (staged changes)
   - `git diff` (unstaged changes)
   - If all are empty, tell the user: "I can't find any changes to review. Are there uncommitted changes, or should I compare against a specific commit?"
6. If the diff fails or looks wrong (stale branch), tell the user and offer to rebase first.

**Filter changed files:**

Remove files that reviewers should not waste time on:
- Lock files: `package-lock.json`, `yarn.lock`, `pnpm-lock.yaml`, `Gemfile.lock`, `poetry.lock`, etc.
- Generated/minified files: `*.min.js`, `*.min.css`, `*.generated.*`, `*.pb.go`
- Binary assets: images, fonts, compiled binaries, `.woff`, `.png`, `.jpg`
- Vendored dependencies: `vendor/`, `node_modules/`, `third_party/`

**Build the shared context package:**

```
## Review Context
- Branch: <name>
- Base: origin/<base>
- Issue: <number and title, or "General quality review">
- Requirements: <acceptance criteria summary>
- Changed files: <filtered list>
- Technology stack: <from AGENTS.md>

<full diff>
```

### Phase 2 Hard Stop

After collecting the diff and filtered file list, **you must proceed to Phase 3 in your very next turn.** Do NOT read additional entity files, reference files, or upstream dependencies. Reviewers review what changed, not the entire dependency graph. If a reviewer truly needs more context, they will note it in their report — you can address it then.

The only exception: if the diff is for a modified file with less than 10 lines changed and the surrounding context is unclear, you may read that one file. But this should be rare.

**Large diff handling:**

If the diff exceeds ~500 lines, do not reduce the number of reviewers. Instead, partition the changed files by domain relevance and give each reviewer only the files relevant to their specialty, plus a summary of the full change. Include the filtered file list in the shared context so reviewers know what else changed.

## Phase 3: Deploy Review Panel (Tiered)

Read `references/review-domains.md` for the full domain definitions, deployment rules, and reviewer briefs. That file is the authoritative source for which domains exist, when to deploy each, and which tier they belong to.

### Panel Size

More reviewers means more parallel sub-agents, which costs time and tokens. Use judgment:

- **2-3 reviewers** — simple bugfix, small scope, few files changed. Tier 1 only: Correctness. Tier 3: Code Quality. Tier 4: Test Coverage. Skip Tiers 2 if no relevant changes.
- **4-5 reviewers** — typical feature, moderate scope
- **6-8 reviewers** — large feature, full-stack changes, cross-cutting concerns
- **Cap at 8** — beyond this, coordination overhead exceeds value. Combine related domains (e.g., correctness + code quality, performance + concurrency).

### Tiered Deployment

Deploy reviewers tier by tier. **Each tier must complete and pass its gate check before the next tier starts.**

**Per-tier workflow:**

1. **Deploy reviewers for the current tier** — spawn them per the execution mode (parallel batches of 2-3, or sequential one-at-a-time).
2. **Collect reports** from all reviewers in the tier.
3. **Run gate check** (see Phase 3.5).
4. **If gate passes** → proceed to next tier.
5. **If gate triggers** → handle per Phase 3.5, then re-check or continue.
6. **After all tiers complete** → proceed to Phase 4 (Merge).

### Spawning Reviewers

Read `agents/reviewer.md` for the full prompt template, then construct each reviewer's prompt by:
1. Pasting the domain brief from `references/review-domains.md`
2. Pasting the shared context package from Phase 2 (the diff you already have — do NOT re-read files)
3. Pasting only the file contents relevant to that domain (not all files for every reviewer — this keeps prompts lean)

If you have not yet read `references/review-domains.md` and `agents/reviewer.md`, read them NOW (one turn, both files). Then immediately proceed to spawning Tier 1 reviewers in the NEXT turn. Do NOT interleave any other file reads.

**Spawn in batches of 2-3 reviewers per turn** — this is critical for connection stability. Do NOT spawn all reviewers in one turn.

### Handling Reviewer Failures

If a reviewer sub-agent times out or fails:
- **Non-critical domain** (Frontend, Architecture, API Design): skip it, note the gap in the final report.
- **Core domain** (Code Quality, Security, Test Coverage, Correctness): retry once. If it fails again, tell the user which domain couldn't be reviewed and offer to rerun just that one.

## Phase 3.5: Gate Check

After each tier's reviewers complete, evaluate the findings and determine if a gate should trigger.

### Gate Rules

1. **Collect all findings** from the tier's reviewers.
2. **Find the highest severity** among all findings in the tier.
3. **Apply gate logic:**
   - **PASSED or INFO only** → gate does NOT trigger. Continue to next tier.
   - **WARNING** → gate triggers. Ask user.
   - **ERROR** → gate triggers. Strongly recommend fixing.
   - **FATAL** → gate triggers. Strongly recommend fixing or aborting.

### Gate Prompt

Present a summary of the tier's findings and ask the user:

**For WARNING:**
```
** Gate: <Tier> review found significant issues **

<Domain> findings:
- [prefix-W1] WARNING | file:line — description
- [prefix-W2] WARNING | file:line — description

Would you like to:
1. Fix these now and re-review
2. Continue reviewing other domains anyway
3. Abort review entirely
```

**For ERROR:**
```
** Gate: <Tier> review found critical issues **

<Domain> findings:
- [prefix-E1] ERROR | file:line — description
- [prefix-E2] ERROR | file:line — description

Recommended: fix now before continuing. Would you like to:
1. Fix these now and re-review (Recommended)
2. Continue reviewing other domains anyway
3. Abort review entirely
```

**For FATAL:**
```
** Gate: <Tier> review found fundamental issues **

<Domain> findings:
- [prefix-F1] FATAL | file:line — description

These issues likely make further review meaningless until addressed. Would you like to:
1. Fix these now and re-review (Recommended)
2. Continue reviewing other domains anyway
3. Abort review entirely
```

### User Picks "Fix Now"

1. **Spawn fix agents** for the specific findings that triggered the gate, using `agents/fix-executor.md`.
2. **After fixes complete**, spawn a `quick-reviewer` agent using `agents/quick-reviewer.md` — scoped to the original domain and specific findings only (NOT a full domain pass).
3. **Evaluate quick-reviewer result:**
   - **PASSED** → gate clears. Proceed to next tier.
   - **GATE** (still has issues) → increment re-review counter. See loop protection below.

### User Picks "Continue Anyway"

- Note the findings. They will be included in the final merged report in Phase 4.
- Proceed to the next tier.
- Do NOT re-review these findings unless the user asks for it later.

### User Picks "Abort"

- Stop the review immediately.
- Present whatever findings were collected so far as a partial report.

### Re-review Loop Protection

To prevent infinite fix→re-review cycles:

- **Max 2 re-review cycles per tier.**
- After the 2nd cycle, if issues persist, tell the user:
  > "After 2 fix attempts, issues still remain in <Domain>. Would you like to:
  > 1. Continue reviewing other domains anyway (findings will be in the final report)
  > 2. Abort review entirely"
- Do NOT attempt a 3rd fix cycle.

### Cross-Domain Note

If a fix introduces issues in a domain that belongs to a **different tier**, ignore them here — they'll be caught when that tier runs. The quick-reviewer only checks the original domain.

## Phase 4: Merge Reports & Organize Work Plan

After all tiers have completed (passed gates or user chose to continue), merge all findings from all tiers into a single work plan.

### Merge Findings

1. **Collect** all findings from all tiers into a single list. Each finding already has a domain-prefixed ID (e.g., `sec-E1`, `perf-W1`).
2. **Deduplicate** — when multiple reviewers flag the same or overlapping issues:
   - **Same code location, same concern**: Merge into one finding. Use the ID from the domain with highest expertise (security > code quality, performance > code quality). Combine descriptions.
   - **Same code location, different concerns**: Merge into one finding led by the higher-expertise domain, with separate sub-items for each concern.
   - **Same root cause, different locations**: Group as one finding with multiple affected files.
   - **Severity rule**: When merging, use the highest severity among the duplicates.
3. **Cross-reference** — identify when seemingly separate findings share a root cause. Group them as a single work item with multiple facets.

### Build the Work Plan

Organize merged findings into a structured plan:

```
## Review Summary
- Reviewers deployed: <list domains by tier>
- Total findings: X fatal, Y error, Z warning, W info

## Work Plan

### Priority 1 — Fatal (must fix, blocks other work)
- [ ] **[ID]** <Domain> | File:Line | Description
  Fix: <suggested fix>
  Depends on: <none | other IDs>

### Priority 2 — Error (must fix)
- [ ] ...

### Priority 3 — Warning (should fix)
- [ ] ...

### Priority 4 — Info (nice to have)
- [ ] ...

### Parallelization Groups
- **Group A** (independent, can run simultaneously): [ID1, ID3, ID5]
- **Group B** (depends on Group A): [ID2, ID4]
- **Group C** (depends on Group B): [ID6]
```

The parallelization analysis is key: findings that touch different files with no logical dependency can run in parallel. Findings that modify the same file or depend on a structural change must be sequenced. **Never assign two fixes to the same file in the same parallelization group** — conflicting edits will cause failures.

### Verdict

Based on the findings:
- **APPROVE** — no fatal or error issues
- **REQUEST CHANGES** — fatal or error issues found
- **REJECT** — fundamental problems that require rethinking the approach

## Phase 5: Present Plan & Get User Approval

Present the merged work plan to the user. Show:

1. **Which tiers ran** and which reviewers were deployed, with a one-line summary of each domain's verdict
2. **The full findings list** with severity, location, and suggested fix
3. **The parallelization groups** — explain which fixes can run simultaneously
4. **The overall verdict**

Then ask:
> "Here's the review report. <N> fatal, <M> error, <K> warning, <L> info findings.
> The work is organized into <X> parallel groups. Would you like to:
> 1. Fix everything as planned
> 2. Fix only specific items (tell me which)
> 3. Skip fixes — you'll handle them yourself
> 4. Adjust the plan first"

Wait for the user's response before proceeding to execution. Do not start fixing anything without explicit approval.

## Phase 6: Execute Approved Fixes

Once the user approves (or adjusts) the work plan:

### Fix Prompt Construction

For each approved finding, spawn a `general` sub-agent using the prompt template from `agents/fix-executor.md`. Read that file for the full template, then fill in:
- The exact finding (description, file, line, suggested fix)
- The relevant file contents
- Project conventions from AGENTS.md
- Any dependencies on other fixes

The domain expertise comes from the finding description and suggested fix, not from the agent type — all fixes use `general` sub-agents.

### Parallel Execution (Batched)

1. Start with the first parallelization group — spawn up to 2-3 fix agents simultaneously (connection stability limit). Briefly acknowledge: "Fixing batch: [ID1, ID2]..."
2. Wait for the batch to complete. If a fix fails, log the error and continue — do not block the batch.
3. Spawn the next batch of fixes from the same group (or move to the next group if this one is done).
4. Repeat until all approved fixes are done.
5. After all fixes, detect the project's build system and run verification:
   - Check AGENTS.md for explicit build/lint commands first.
   - `package.json` present → `npm run build` (or `pnpm build`, `yarn build`)
   - `Cargo.toml` present → `cargo build`
   - `*.csproj` or `*.sln` present → `dotnet build`
   - `go.mod` present → `go build ./...`
   - `Makefile` present → `make`
6. If the build fails after fixes, report the failure to the user with the error output. Do not attempt to auto-fix build failures — the user decides whether to roll back or fix manually.

### Present Results

After execution, summarize:
- What was fixed (list each finding with ✓)
- What was skipped (if user chose partial fix)
- What failed (if any fixes errored)
- Build/lint results
- Any issues encountered during fixes

## Phase 7: Verify Fixes

If the fixes were significant (fatal or error findings), run a verification pass:

1. `git diff origin/<base>...HEAD` to get the updated diff (including fixes).
2. Spawn a single `general` sub-agent using the prompt template from `agents/fix-verifier.md`. Read that file for the full template, then fill in the original findings list, the updated diff, and the updated file contents.
3. Report results to the user. If the verifier finds regressions, present them immediately — do not attempt to fix without user approval.

Skip this step for 1-2 info-only fixes where the changes are trivial, or if the user wants to move on.

## Reference Files

- `references/review-domains.md` — Full domain definitions, tier assignments, deployment rules, and detailed reviewer briefs. Read this when building the reviewer panel prompts.
- `agents/reviewer.md` — Prompt template for spawning reviewer sub-agents. Read this when constructing reviewer prompts in Phase 3.
- `agents/quick-reviewer.md` — Prompt template for lightweight re-review after gate fixes. Read this when re-checking fixes in Phase 3.5.
- `agents/fix-executor.md` — Prompt template for spawning fix sub-agents. Read this when constructing fix prompts in Phase 6.
- `agents/fix-verifier.md` — Prompt template for the post-fix verification agent. Read this when running Phase 7.
- `evals/evals.json` — Test scenarios for verifying the skill works in your environment.

## Important Notes

- Issue-driven reviews always produce better results than generic quality passes.
- Never guess requirements — ask when uncertain.
- The coordinator does not review code. You orchestrate specialists.
- Every finding needs a file:line reference and a suggested fix.
- Compare against existing codebase patterns, not abstract ideals.
- Performance concerns should reference realistic scale, not premature optimization.
- Deduplication matters — three reviewers flagging the same line should produce one merged finding, not three separate fixes.
- Filter out lock files, generated files, and vendored code before giving files to reviewers.
- Tiers gate on WARNING and above. INFO and PASSED flow through automatically.
- Re-reviews after gate fixes are scoped to the original domain only — not a full re-pass.
- Max 2 re-review cycles per tier to prevent infinite loops.
