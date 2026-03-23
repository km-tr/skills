---
name: codex-exec-review
description: Runs parallel multi-agent code review via codex exec. Use when reviewing branch changes before merge, when pre-merge review is needed across type safety, security, architecture, React, and code quality perspectives, or when severity-classified automated review is wanted
---

# codex-exec-review

## Overview

Launches 5 specialized review agents in parallel via `codex exec` (gpt-5.4, high effort). Each agent reviews the current branch's diff from a distinct perspective and produces a severity-classified report with an APPROVE/WARNING/BLOCK verdict.

## When to Use

- Before merging a feature branch — catches cross-cutting concerns
- When multiple review perspectives are needed in one pass
- After significant refactoring or architectural changes
- Symptoms: "ready to merge", "need review", "pre-merge check"

**Not for:** trivial single-line changes, branches with no diff

## Agents

| Agent | Specialty |
|---|---|
| typescript-pro | Type safety, generics, strict mode, module boundaries |
| reviewer | Code quality, readability, naming, DRY, standards |
| code-reviewer | Bugs, edge cases, security (OWASP top 10), error handling |
| architect-reviewer | Architecture, coupling/cohesion, API design, scalability |
| react-specialist | Hooks, re-renders, memoization, state management, a11y |

## Workflow

1. Run the script below
2. Read each agent's output — focus on CRITICAL and HIGH findings first
3. If verdict is **Block**: fix critical issues, re-run
4. If verdict is **Warning**: evaluate HIGH issues, fix or document exceptions
5. If all verdicts are **Approve**: safe to merge

## Script

```bash
#!/usr/bin/env bash
set -euo pipefail

BASE_BRANCH=$(git symbolic-ref refs/remotes/origin/HEAD 2>/dev/null \
  | sed 's@^refs/remotes/origin/@@' || echo "main")
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
Review the git diff below. For each issue found, use this format:

[SEVERITY] Issue Title
File: path/to/file.ts:lineNumber
Issue: Description of the problem and its impact.
Fix: Suggested resolution with code example.

Severity levels: CRITICAL, HIGH, MEDIUM, LOW

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

Each agent outputs findings organized by severity:

```
[SEVERITY] Issue Title
File: path/to/file.ts:lineNumber
Issue: Description of the problem and its impact.
Fix: Suggested resolution with code examples.

  code example showing BAD pattern
  code example showing GOOD pattern
```

Every review concludes with a summary table:

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
| Base branch not found | Falls back to `main`. Set `origin/HEAD` if using different default |
| Diff too large for prompt | Split review by directory or use `git diff --stat` to limit scope |
| Missing `codex` CLI | Install first: `npm i -g @openai/codex` |
| Ignoring WARNING verdicts | Evaluate each HIGH issue — document exceptions or fix before merge |
