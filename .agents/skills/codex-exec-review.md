---
name: codex-exec-review
description: Use when reviewing current branch changes before merge, when parallel multi-perspective code review is needed, or when wanting automated severity-based review via codex exec agents
---

# codex-exec-review

## Overview

Parallel code review using 5 specialized `codex exec` agents (gpt-5.4, high effort). Each agent reviews the current branch's diff from a distinct perspective and produces a severity-classified report.

## When to Use

- Before merging a feature branch
- When you want multi-perspective review (type safety, security, architecture, React, general quality)
- When manual review would miss cross-cutting concerns

**When NOT to use:**
- No changes on branch (script exits automatically)
- Single-file trivial changes where manual review suffices

## Quick Reference

| Agent | Focus |
|---|---|
| typescript-pro | Type safety, generics, strict mode, module boundaries |
| reviewer | Code quality, readability, naming, DRY, standards |
| code-reviewer | Bugs, edge cases, security (OWASP top 10), error handling |
| architect-reviewer | Architecture, coupling/cohesion, API design, scalability |
| react-specialist | Hooks, re-renders, memoization, state management, a11y |

## Implementation

Run the script below. It detects the base branch, generates the diff, and launches all 5 agents in parallel.

```bash
#!/usr/bin/env bash
set -euo pipefail

BASE_BRANCH=$(git symbolic-ref refs/remotes/origin/HEAD 2>/dev/null | sed 's@^refs/remotes/origin/@@' || echo "main")
DIFF=$(git diff "${BASE_BRANCH}"...HEAD)

if [ -z "$DIFF" ]; then
  echo "No changes detected against ${BASE_BRANCH}."
  exit 0
fi

echo "=== Codex Exec Parallel Review ==="
echo "Base: ${BASE_BRANCH} | Model: gpt-5.4 | Effort: high"
echo ""

DIFF_FILE=$(mktemp)
echo "$DIFF" > "$DIFF_FILE"
trap 'rm -f "$DIFF_FILE"' EXIT

REVIEW_INSTRUCTIONS=$(cat <<'REVIEW_END'
Review the git diff below. Organize findings by severity using this structure:

[SEVERITY] Issue Title
File: path/to/file.ts:lineNumber
Issue: Description of the problem and its impact.
Fix: Suggested resolution with code examples.

  code example showing BAD pattern
  code example showing GOOD pattern

Conclude with:

## Review Summary

| Severity | Count | Status |
|----------|-------|--------|
| CRITICAL | X     | status |
| HIGH     | X     | status |
| MEDIUM   | X     | status |
| LOW      | X     | status |

Verdict: [APPROVE/WARNING/BLOCK] — explanation

Approval Criteria:
- Approve: No critical or high-severity issues detected
- Warning: High-severity issues present; mergeable with caution
- Block: Critical issues found; changes must be fixed before merging
REVIEW_END
)

declare -A AGENTS=(
  [typescript-pro]="expert TypeScript reviewer. Focus: type safety, strict mode, generics, utility types, type narrowing, module boundaries"
  [reviewer]="senior code reviewer. Focus: code quality, readability, naming, maintainability, DRY, coding standards"
  [code-reviewer]="security-focused code reviewer. Focus: bugs, logic errors, edge cases, null/undefined, race conditions, input validation, OWASP top 10"
  [architect-reviewer]="software architecture reviewer. Focus: architectural patterns, separation of concerns, dependency management, scalability, coupling/cohesion, API design"
  [react-specialist]="React and frontend performance expert. Focus: component patterns, hooks rules, re-render optimization, state management, memoization, accessibility"
)

for agent in "${!AGENTS[@]}"; do
  codex exec -m gpt-5.4 --effort high "$(cat <<EOF
You are **${agent}**, a ${AGENTS[$agent]}.

${REVIEW_INSTRUCTIONS}

Git diff to review:
$(cat "$DIFF_FILE")
EOF
)" &
done

wait
echo ""
echo "=== All reviews complete ==="
```

## Review Output Format

Organize findings by severity. For each issue, use this structure:

```
[SEVERITY] Issue Title
File: path/to/file.ts:lineNumber
Issue: Description of the problem and its impact.
Fix: Suggested resolution with code examples.

  code example showing BAD pattern
  code example showing GOOD pattern
```

Conclude every review with a summary table:

```
## Review Summary

| Severity | Count | Status |
|----------|-------|--------|
| CRITICAL | X     | status |
| HIGH     | X     | status |
| MEDIUM   | X     | status |
| LOW      | X     | status |

Verdict: [APPROVE/WARNING/BLOCK] — explanation
```

## Approval Criteria

- **Approve**: No critical or high-severity issues detected
- **Warning**: High-severity issues present; mergeable with caution
- **Block**: Critical issues found; changes must be fixed before merging

## Common Mistakes

| Mistake | Fix |
|---|---|
| Running with no diff | Script handles this — exits with message |
| Base branch not found | Falls back to `main`; set `origin/HEAD` if using different default |
| Diff too large for prompt | Split review by directory or limit to staged changes |
| Missing `codex` CLI | Install OpenAI Codex CLI first: `npm i -g @openai/codex` |
