---
name: nitpicker
description: >
  Multi-agent post-implementation code review. Deploys a panel of specialist sub-agents in parallel —
  each focused on one domain (correctness, security, performance, concurrency, code quality, API design,
  test coverage, frontend, architecture). The coordinator merges their reports into a prioritized work plan,
  presents it for user approval, then dispatches fixes in parallel across specialist sub-agents.
  Use this skill after completing an implementation task — whether it's a feature, bugfix,
  refactoring, or any code change. Triggers on: "review the changes", "review my code",
  "check correctness", "code review", "nitpick this", "roast my code", "what's wrong with this",
  or after any implementation is completed. Also triggers when the user asks to verify, validate,
  quality-check, or critique completed work.
license: MIT
metadata:
  author: rankobp
  version: "0.1.0"
---

# Nitpicker

This skill orchestrates a multi-agent code review pipeline:

1. **Scope & Gather** — determine what to review and collect context
2. **Deploy Review Panel** — spawn specialist sub-agents in parallel, each reviewing one domain
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

## Connection Stability

API providers drop connections when the model generates large tool call argument payloads in a single batch. Since nitpicker orchestrates many sub-agents, this is the primary failure mode — the connection dies mid-generation and the entire review stalls.

**Hard limits — never exceed these:**
- **Max 2-3 tool calls per response** — fewer calls = smaller payload = shorter silence gap
- **Max 2-3 sub-agents spawned per turn** — batch reviewers and fix executors in groups of 2-3, acknowledge progress between batches, then continue
- **Keep prompts lean** — don't include full file contents in every reviewer prompt. Partition by domain relevance. Give each reviewer only the files relevant to their specialty plus a summary of the full change
- **Break complex tasks into steps** — for multi-file changes, do 2-3 files per turn, acknowledge progress, then continue. Do NOT batch 6+ sub-agents in one response

**What this means for each phase:**
- Phase 2 (Gather): 2-3 file reads per turn. Read diff + reference files in one turn (2-3 calls max).
- Phase 3 (Deploy): Spawn reviewers in batches of 2-3 per turn. Acknowledge which reviewers were launched, wait for results, then spawn the next batch.
- Phase 6 (Execute): Same batching — spawn fix agents 2-3 per turn per parallelization group.
- Phase 7 (Verify): Single agent — no batching needed.

**Between batches, briefly tell the user what's happening** — e.g., "Reviewers 1-3 deployed, waiting for results..." This prevents the appearance of stalling.

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

**Fetch issue details (if issue-driven):**

- Read the issue body for acceptance criteria and the comments for scope clarifications.
- Read ALL filtered changed files — the reviewers need full file contents, not just diff hunks.

**Build the shared context package** that every reviewer sub-agent receives:

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

**Large diff handling:**

If the diff exceeds ~500 lines, do not reduce the number of reviewers. Instead, partition the changed files by domain relevance and give each reviewer only the files relevant to their specialty, plus a summary of the full change. Include the filtered file list in the shared context so reviewers know what else changed.

## Phase 3: Deploy Review Panel

Read `references/review-domains.md` for the full domain definitions, deployment rules, and reviewer briefs. That file is the authoritative source for which domains exist and when to deploy each.

### Panel Size

More reviewers means more parallel sub-agents, which costs time and tokens. Use judgment:

- **2-3 reviewers** — simple bugfix, small scope, few files changed. Non-issue-driven: Code Quality + Test Coverage. Issue-driven: add Correctness.
- **4-5 reviewers** — typical feature, moderate scope
- **6-8 reviewers** — large feature, full-stack changes, cross-cutting concerns
- **Cap at 8** — beyond this, coordination overhead exceeds value. Combine related domains (e.g., correctness + code quality, performance + concurrency).

### Spawning Reviewers (Batched)

Read `agents/reviewer.md` for the full prompt template, then construct each reviewer's prompt by:
1. Pasting the domain brief from `references/review-domains.md`
2. Pasting the shared context package from Phase 2
3. Pasting only the file contents relevant to that domain (not all files for every reviewer — this keeps prompts lean)

**Spawn in batches of 2-3 reviewers per turn** — this is critical for connection stability. Do NOT spawn all reviewers in one turn. The batch workflow:

1. Determine the full panel and note the list for the user.
2. Spawn the first batch of 2-3 reviewers as parallel sub-agents.
3. Briefly acknowledge: "Reviewers batch 1 deployed (Correctness, Security, Performance), waiting for results..."
4. Wait for results, then spawn the next batch of 2-3.
5. Repeat until all reviewers are done.

This keeps each turn small enough to avoid connection drops while still getting parallelism within each batch.

### Handling Reviewer Failures

If a reviewer sub-agent times out or fails:
- **Non-critical domain** (Frontend, Architecture, API Design): skip it, note the gap in the final report.
- **Core domain** (Code Quality, Security, Test Coverage, Correctness): retry once. If it fails again, tell the user which domain couldn't be reviewed and offer to rerun just that one.

## Phase 4: Merge Reports & Organize Work Plan

As each reviewer completes, collect their report. Once ALL reviewers have reported (or been skipped per failure rules):

### Merge Findings

1. **Collect** all findings into a single list. Each finding already has a domain-prefixed ID (e.g., `sec-C1`, `perf-S1`).
2. **Deduplicate** — when multiple reviewers flag the same or overlapping issues:
   - **Same code location, same concern**: Merge into one finding. Use the ID from the domain with highest expertise (security > code quality, performance > code quality). Combine descriptions.
   - **Same code location, different concerns**: Merge into one finding led by the higher-expertise domain, with separate sub-items for each concern. Example: security flags SQL injection AND code quality flags string concatenation on the same line — one finding led by security, with the code quality concern as a secondary aspect.
   - **Same root cause, different locations**: Group as one finding with multiple affected files. Example: missing validation on three endpoints = one finding with three file references.
   - **Severity rule**: When merging, use the highest severity among the duplicates.
3. **Cross-reference** — identify when seemingly separate findings share a root cause. Group them as a single work item with multiple facets. This prevents the fix phase from doing redundant work.

### Build the Work Plan

Organize merged findings into a structured plan:

```
## Review Summary
- Reviewers deployed: <list domains>
- Total findings: X critical, Y significant, Z minor

## Work Plan

### Priority 1 — Critical (must fix)
- [ ] **[ID]** <Domain> | File:Line | Description
  Fix: <suggested fix>
  Depends on: <none | other IDs>

### Priority 2 — Significant (should fix)
- [ ] ...

### Priority 3 — Minor (nice to have)
- [ ] ...

### Parallelization Groups
- **Group A** (independent, can run simultaneously): [ID1, ID3, ID5]
- **Group B** (depends on Group A): [ID2, ID4]
- **Group C** (depends on Group B): [ID6]
```

The parallelization analysis is key: findings that touch different files with no logical dependency can run in parallel. Findings that modify the same file or depend on a structural change must be sequenced. **Never assign two fixes to the same file in the same parallelization group** — conflicting edits will cause failures.

### Verdict

Based on the findings:
- **APPROVE** — no critical or significant issues
- **REQUEST CHANGES** — critical or significant issues found
- **REJECT** — fundamental problems that require rethinking the approach

## Phase 5: Present Plan & Get User Approval

Present the merged work plan to the user. Show:

1. **Which reviewers were deployed** and a one-line summary of each domain's verdict
2. **The full findings list** with severity, location, and suggested fix
3. **The parallelization groups** — explain which fixes can run simultaneously
4. **The overall verdict**

Then ask:
> "Here's the review report. <N> critical, <M> significant, <K> minor findings.
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

If the fixes were significant (critical or significant findings), run a verification pass:

1. `git diff origin/<base>...HEAD` to get the updated diff (including fixes).
2. Spawn a single `general` sub-agent using the prompt template from `agents/fix-verifier.md`. Read that file for the full template, then fill in the original findings list, the updated diff, and the updated file contents.
3. Report results to the user. If the verifier finds regressions, present them immediately — do not attempt to fix without user approval.

Skip this step for 1-2 minor-only fixes where the changes are trivial, or if the user wants to move on.

## Reference Files

- `references/review-domains.md` — Full domain definitions, deployment rules, and detailed reviewer briefs. Read this when building the reviewer panel prompts.
- `agents/reviewer.md` — Prompt template for spawning reviewer sub-agents. Read this when constructing reviewer prompts in Phase 3.
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
