---
name: address-reviews
description: 直前の会話に含まれるレビュー結果を精査し、妥当な指摘に対して修正を実施する。レビュアーの指摘がすべて正しいとは限らないため、各コメントを批判的に評価する。「レビュー対応して」「レビューコメント修正して」「指摘を直して」「レビュー指摘に対応して」などのリクエストで使用。
---

# address-reviews

## Overview

Evaluates review findings from the preceding conversation (severity-classified reviews, PR comments, free-form feedback — any format), critically assesses each one, fixes justified issues, and produces an updated summary with a new verdict.

## When to Use

- When review findings in the conversation need to be addressed
- When PR review comments need resolution
- Symptoms: "fix review issues", "address review findings", "resolve review comments", "レビュー対応して", "指摘を直して"

**Not for:** when no review findings exist, running the review itself (that is a separate skill)

## Scope

This skill **evaluates, addresses, and re-summarizes** review findings. It does not run reviews or merge branches.

## Workflow

### Step 1 — Collect Review Findings

Locate review findings in the preceding conversation. Accept any of these formats:

- **Severity-classified format**: `[SEVERITY] Title` + `File:` + `Issue:` + `Fix:`
- **PR review comments**: inline comments with file and line references
- **Free-form review**: bullet points or text-based feedback

For each finding, record the reviewer name (agent name, GitHub username, etc.) and the target file/line.

If no review output is found in the conversation, ask the user to provide it.

### Step 2 — Evaluate Each Finding Independently

For **each** extracted issue:

1. **Read the actual code** at the referenced file and line
2. Assess whether the issue is real:
   - Does the described problem actually exist in the code?
   - Is the suggested fix appropriate, or is there a better approach?
   - Could this be a false positive (e.g., the reviewer lacked project context)?
3. If severity is not provided, assign CRITICAL / HIGH / MEDIUM / LOW yourself
4. Classify your assessment:
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
| No review output in context | Ask user to provide review results |
| Fix introduces new issue | Verify surrounding code still works after each change |
| Findings without severity left untriaged | Assign severity yourself before triaging |
