# Quick Reviewer Agent

This is a lightweight re-review agent used after a gate fix is applied. It is scoped to a single domain and checks only whether the specific findings were resolved, not a full domain pass.

## Why This Agent Exists

When a gate triggers (WARNING/ERROR/FATAL found) and the user chooses to fix before continuing, we need to verify the fix resolved the issue without running a full domain review again. This agent checks only the specific findings that triggered the gate — fast, focused, and cheap.

## When to Use

- After a gate fix is applied (user chose "Fix now and re-review")
- Scoped to the domain that triggered the gate
- Only checks the specific findings that were fixed
- Does NOT replace the full fix verifier (Phase 7)

## Prompt Template

```
You are verifying that specific issues were fixed in the <DOMAIN> domain. This is a focused re-review — do NOT do a full domain pass. Only check the findings listed below.

<DOMAIN BRIEF — paste ONLY the severity guidelines from this domain's brief in references/review-domains.md>

## Original Findings to Verify

<List each finding that was fixed:
- **[ID]** SEVERITY | file:line | description
  Suggested fix: <the fix that was applied>
>

## Updated File Contents

<Paste the FULL current contents of each file that was modified by the fix>

## Updated Diff

<Paste the diff showing only the fix changes>

## Your Task

For each finding above:

1. **Is the specific problem resolved?** Check that the code change addresses the root cause of the finding. The fix doesn't need to match the suggestion word-for-word.

2. **Any regressions in this domain?** Look at the changed lines and surrounding context ONLY for new issues in YOUR domain. Do not look for issues outside your domain.

3. **Assign a severity** to any new issues using the same scale: FATAL | ERROR | WARNING | INFO.

## Output Format

```
## Quick Re-Review — <Domain>

### Per-Finding Verification

For each finding:

- **[ID]**: RESOLVED | PARTIALLY RESOLVED | NOT RESOLVED
  Evidence: <what you see in the code that confirms or denies the fix>
  Notes: <any concerns about how it was fixed, or "Clean fix">

### New Issues in This Domain

<List any NEW issues introduced by the fix, using the same format:
- **[ID]** SEVERITY: FATAL|ERROR|WARNING|INFO | FILE: path/to/file:line
  Description.
>

(Or state "None found" if no new issues.)

### Domain Status

PASSED | GATE
- If all findings are RESOLVED and no new FATAL/ERROR/WARNING issues: PASSED
- If any finding is NOT RESOLVED or new WARNING+ issues found: GATE
```
```

## Coordinator Notes

- Spawn a single `general` sub-agent with this prompt.
- This is lightweight — one agent, one domain, specific findings only.
- If the result is **PASSED**: proceed to the next tier.
- If the result is **GATE**: the same gate fires again with the new/remaining findings. This counts toward the re-review loop limit (max 2 cycles per tier).
- Do NOT use this agent for the final verification pass (Phase 7) — that uses `fix-verifier.md`.
- If the quick reviewer finds new issues in a DIFFERENT domain, ignore them — they'll be caught when that domain's tier runs.
