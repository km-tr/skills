---
name: codex-exec-review
description: Runs parallel multi-agent code review via codex exec. Use when reviewing branch changes before merge, when pre-merge review is needed across type safety, security, architecture, React, and code quality perspectives, or when severity-classified automated review is wanted
---

# codex-exec-review

## Overview

Launches a single read-only `codex exec` session that performs 5 parallel reviews from distinct perspectives. Produces severity-classified findings with an APPROVE/WARNING/BLOCK verdict per perspective.

## When to Use

- Before merging a feature branch — catches cross-cutting concerns
- When multiple review perspectives are needed in one pass
- After significant refactoring or architectural changes
- Symptoms: "ready to merge", "need review", "pre-merge check"

**Not for:** trivial single-line changes, branches with no diff

## Workflow

1. Run the script below
2. Read each perspective's output — focus on CRITICAL and HIGH findings first
3. If verdict is **Block**: fix critical issues, re-run
4. If verdict is **Warning**: evaluate HIGH issues, fix or document exceptions
5. If all verdicts are **Approve**: safe to merge

## Script

```bash
#!/usr/bin/env bash

BASE_BRANCH=$(git symbolic-ref refs/remotes/origin/HEAD 2>/dev/null \
  | sed 's@^refs/remotes/origin/@@' || echo "main")
DIFF=$(git diff "${BASE_BRANCH}"...HEAD)

if [ -z "$DIFF" ]; then
  echo "No changes detected against ${BASE_BRANCH}."
  exit 0
fi

echo "=== Codex Exec Parallel Review ==="
echo "Base: ${BASE_BRANCH} | Model: gpt-5.4 | Sandbox: read-only"
echo ""

codex exec -m gpt-5.4 -s read-only -c model_reasoning_effort=high "$(cat <<EOF
You are a senior code review coordinator. Review the git diff below from 5 distinct perspectives IN PARALLEL. Produce a separate report for each perspective.

## Review Perspectives

1. **TypeScript Pro** — Type safety, strict mode, generics, utility types, type narrowing, module boundaries
2. **Code Quality Reviewer** — Readability, naming, maintainability, DRY, coding standards
3. **Security Reviewer** — Bugs, logic errors, edge cases, null/undefined, race conditions, input validation, OWASP top 10
4. **Architecture Reviewer** — Architectural patterns, separation of concerns, dependency management, scalability, coupling/cohesion, API design
5. **React Specialist** — Component patterns, hooks rules, re-render optimization, state management, memoization, accessibility

## Output Format

For each perspective, produce a titled section with findings organized by severity.

For each issue:

[SEVERITY] Issue Title
File: path/to/file.ts:lineNumber
Issue: Description of the problem and its impact.
Fix: Suggested resolution with code example.

Severity levels: CRITICAL, HIGH, MEDIUM, LOW

Conclude each perspective with:

## [Perspective Name] Review Summary

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

## Git diff to review:

${DIFF}
EOF
)"

echo ""
echo "=== Review complete ==="
```

## Review Output Format

Each perspective outputs findings organized by severity:

```
[SEVERITY] Issue Title
File: path/to/file.ts:lineNumber
Issue: Description of the problem and its impact.
Fix: Suggested resolution with code examples.

  code example showing BAD pattern
  code example showing GOOD pattern
```

Every perspective concludes with a summary table:

```
## [Perspective Name] Review Summary

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
