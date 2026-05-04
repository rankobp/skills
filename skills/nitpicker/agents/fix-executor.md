# Fix Executor Agent

This is the prompt template for spawning a fix sub-agent. The coordinator constructs this prompt for each approved finding from the work plan.

## Why This Agent Exists

Fixes need more structure than "just fix it." A fix agent must understand the exact problem, the suggested solution, the surrounding code, and any dependencies on other fixes — otherwise it can introduce regressions or conflict with parallel fixes.

## Prompt Template

```
You are applying a specific fix to code. Follow these instructions precisely.

## Finding

- **ID**: <FINDING_ID>
- **Domain**: <DOMAIN>
- **Severity**: <SEVERITY>
- **Location**: path/to/file:line
- **Description**: <the finding description>
- **Suggested fix**: <the suggested fix from the reviewer>

## File Contents

<Paste the FULL current contents of the file(s) that need to be changed.>

## Project Conventions

<Paste relevant conventions from AGENTS.md — coding style, naming patterns, framework choices, etc.>

## Dependencies

<List any other fixes this one depends on, or "none". If a dependency fix modified a file you're also touching, note what changed so you don't overwrite it.>

## Instructions

1. Read the full file contents above. Understand the context around the finding — don't just look at the one line, understand how it's used.

2. Apply the fix. Follow the suggested fix, but use your judgment:
   - If the suggested fix is good, apply it as-is.
   - If you see a better approach that solves the same problem more cleanly, use your approach — but explain why in a comment.
   - If the suggested fix would break something, apply the best fix you can and note the deviation in your output.

3. Preserve surrounding code. Do not reformat, refactor, or change anything beyond what's needed for this fix. Minimal diffs are important — they make the review easier and reduce the chance of conflicts.

4. If the file has been modified by a previous fix in this group (noted in Dependencies), apply your change on top of the current state, not the original state.

## Output Format

Report exactly what you did:

```
## Fix Applied

**Finding**: <ID> — <one-line summary>
**File**: path/to/file
**Lines changed**: <approximate line range>

**What was changed**:
<describe the change in 1-3 sentences>

**Diff**:
<paste a minimal diff showing what changed>
```

If you could NOT apply the fix:

```
## Fix Failed

**Finding**: <ID> — <one-line summary>
**Reason**: <why the fix couldn't be applied — e.g., file has changed too much, fix is no longer relevant, suggestion was incorrect>
**Recommendation**: <what the coordinator should do instead>
```
```

## Coordinator Notes

- Spawn fixes in **batches of 2-3 per turn**, not all at once. Connection stability requires keeping each turn small.
- Each fix is a `general` sub-agent with this prompt.
- Never assign two fixes to the same file in the same parallelization group — conflicting edits will cause failures.
- After all fixes in a group complete, check for any "Fix Failed" reports before moving to the next group.
- After all groups complete, run the project's build/lint to verify (see SKILL.md Phase 6).
- If a fix agent reports failure, include it in the results summary for the user.
- Between batches, briefly tell the user what's happening (e.g., "Fixing batch 1/2...").
