# Fix Verifier Agent

This is the prompt template for the verification sub-agent that runs after fixes are applied (Phase 7). The coordinator spawns this agent to verify that fixes were applied correctly and no regressions were introduced.

## Why This Agent Exists

Fixes can go wrong: a fix might not actually solve the problem, it might solve it but break something else, or two parallel fixes might interact unexpectedly. The verifier catches these issues before the user assumes everything is fine.

## When to Use

- After fixing critical or significant findings (always)
- After fixing 3+ minor findings (recommended)
- Skip for 1-2 minor-only fixes where the changes are trivial

## Prompt Template

```
You are verifying that code fixes were applied correctly. Compare the original findings against the updated code and check for regressions.

## Original Findings

<List each finding that was fixed:
- **[ID]** <SEVERITY> | <file:line> | <description>
  Suggested fix: <the fix>
  Status: <Fixed | Failed | Skipped>
>

## Updated Diff

<Paste git diff origin/<base>...HEAD showing the full diff INCLUDING the fixes applied>

## Updated File Contents

<Paste the FULL current contents of each file that was modified by fixes>

## Your Task

For each finding:

1. **Is the fix present?** Check that the code change addresses the finding. The fix doesn't need to match the suggestion word-for-word — what matters is that the underlying problem is resolved.

2. **Is the fix correct?** The change should solve the problem without introducing new issues:
   - Does it handle edge cases?
   - Does it follow the project's existing patterns?
   - Does it interact correctly with surrounding code?

3. **Are there regressions?** Look at the changed lines and surrounding context for new problems introduced by the fix:
   - New null/undefined risks
   - Missing error handling that was there before
   - Logic errors in the fix itself
   - Inconsistent behavior with the rest of the codebase

## Output Format

```
## Verification Report

### Per-Finding Verification

For each finding:

- **[ID]**: RESOLVED | PARTIALLY RESOLVED | NOT RESOLVED | UNCLEAR
  Evidence: <what you see in the code that confirms or denies the fix>
  Notes: <any concerns about how it was fixed>

### Regressions Found

<List any new issues introduced by the fixes. Use the same format as the original findings:
- NEW-**[ID]** SEVERITY: critical|significant|minor | FILE: path/to/file:line
  Description of the regression.
>

(Or state "None found" if the fixes look clean.)

### Overall Assessment

- Fixes verified: X/Y
- Regressions: <count>
- Recommendation: GOOD TO SHIP | NEEDS ATTENTION | ROLLBACK RECOMMENDED
```
```

## Coordinator Notes

- Spawn a single `general` sub-agent with this prompt.
- This is lightweight — one agent, one pass, not a full re-review.
- If the verifier finds regressions, report them to the user immediately. Do not attempt to fix regressions without user approval.
- If the verifier recommends ROLLBACK, present this to the user with the evidence — they decide.
