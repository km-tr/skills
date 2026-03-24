---
name: address-reviews
description: Evaluates and addresses code review issues from codex-exec-review output. Use when review findings need to be addressed, when codex-exec-review returned WARNING or BLOCK verdict, when automated review issues need triage and resolution. Trigger words — "fix review issues", "address review findings", "resolve review comments", "レビュー対応して"
---

# address-reviews

## Overview

Takes severity-classified review output from codex-exec-review (or any review using the same `[SEVERITY] Title` format), critically evaluates each finding against the actual code, fixes justified issues, and produces an updated summary with the new verdict.

## When to Use

- After running codex-exec-review and receiving WARNING or BLOCK verdict
- When review output contains CRITICAL or HIGH issues that need resolution
- When you want automated triage + fix of review findings
- Symptoms: "fix review issues", "address review findings", "resolve review comments", "レビュー対応して"

**Not for:** reviews with APPROVE verdict (nothing to fix), non-severity-classified review formats

## Scope

This skill **evaluates, addresses, and re-summarizes** review findings. It does not re-run the review from scratch or merge branches.

## Workflow

### Step 1 — Collect Review Findings

Locate the codex-exec-review output in the conversation. Extract every issue block matching this pattern:

```
[SEVERITY] Issue Title
File: path/file.ts:line
Issue: Description of problem
Fix: Suggested solution
```

Also note the original verdict (APPROVE / WARNING / BLOCK), severity counts, and which reviewer agent reported each issue (e.g., typescript-pro, reviewer, code-reviewer, architect-reviewer, react-specialist).

If no review output is found in conversation context, ask the user to provide it or run codex-exec-review first.

### Step 2 — Evaluate Each Finding Independently

For **each** extracted issue:

1. **Read the actual code** at the referenced file and line
2. Assess whether the issue is real:
   - Does the described problem actually exist in the code?
   - Is the severity classification accurate?
   - Is the suggested fix appropriate, or is there a better approach?
   - Could this be a false positive (e.g., the reviewer lacked project context)?
3. Classify your assessment:
   - **FIX** — issue is valid and should be addressed
   - **SKIP** — issue is invalid, a false positive, or not worth fixing (must provide reason)
   - **DOWNGRADE** — issue is real but severity is overstated (state corrected severity)

**Critical principle: Do not blindly accept all review feedback.** Reviewers can be wrong. Evaluate each issue on its own merits against the actual code.

### Step 3 — Apply Fixes

Process in priority order: CRITICAL → HIGH → MEDIUM → LOW.

For each issue classified as FIX:

1. Open the referenced file
2. Apply the fix (use the reviewer's suggestion as a starting point, but improve it if you see a better approach)
3. Verify the fix does not introduce new issues or break surrounding code
4. Note exactly what was changed

For DOWNGRADE issues: only fix if the corrected severity is still CRITICAL or HIGH.

### Step 4 — Produce Summary

Output the results in this exact format:

```
## Address-Reviews Summary

### Addressed Issues

#### [CRITICAL] Issue Title (reviewer-name)
- **File**: path/file.ts:line
- **Problem**: brief description of what was wrong
- **Fix applied**: brief description of what was changed

#### [HIGH] Issue Title (reviewer-name)
- **File**: path/file.ts:line
- **Problem**: brief description
- **Fix applied**: brief description

### Not Addressed

#### [MEDIUM] Issue Title (reviewer-name)
- **File**: path/file.ts:line
- **Reason**: explanation of why this was skipped (false positive / severity downgraded / stylistic preference / etc.)

#### [LOW] Issue Title (reviewer-name)
- **File**: path/file.ts:line
- **Reason**: explanation

### Updated Review Summary

| Severity | Original | Fixed | Remaining |
|----------|----------|-------|-----------|
| CRITICAL | N        | N     | N         |
| HIGH     | N        | N     | N         |
| MEDIUM   | N        | N     | N         |
| LOW      | N        | N     | N         |

Verdict: APPROVE/WARNING/BLOCK — explanation based on remaining issues
```

## Approval Criteria

Re-evaluate the verdict based on **remaining** (unfixed) issues only:

- **Approve**: No remaining CRITICAL or HIGH issues
- **Warning**: Remaining HIGH issues only (can merge with caution)
- **Block**: Remaining CRITICAL issues — must fix before merge

If all CRITICAL and HIGH issues were fixed or validly downgraded/skipped, the verdict should change to APPROVE.

## Common Mistakes

| Mistake | Fix |
|---|---|
| Blindly fixing every reported issue | Evaluate each issue independently; skip false positives with justification |
| Not reading actual code before fixing | Always read the referenced file and line before deciding |
| Changing verdict without re-counting | Re-count remaining issues by severity after all fixes |
| Fixing MEDIUM/LOW when CRITICAL exists | Prioritize CRITICAL → HIGH → MEDIUM → LOW |
| Applying reviewer's fix verbatim when a better fix exists | Use the suggestion as a starting point; improve when appropriate |
| No review output in context | Ask user to provide it or run codex-exec-review first |
| Fix introduces new issue | Verify surrounding code still works after each change |
