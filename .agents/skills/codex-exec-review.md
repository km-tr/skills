---
name: codex-exec-review
description: Runs parallel multi-agent code review via codex exec. Use when reviewing branch changes before merge, when pre-merge review is needed across type safety, security, architecture, React, and code quality perspectives, or when severity-classified automated review is wanted
---

# codex-exec-review

## Overview

Launches a single read-only `codex exec` session that creates 5 named review agents (typescript-pro, reviewer, code-reviewer, architect-reviewer, react-specialist) to review the current branch's diff in parallel. Each agent produces a severity-classified report with an APPROVE/WARNING/BLOCK verdict.

## When to Use

- Before merging a feature branch — catches cross-cutting concerns
- When multiple review perspectives are needed in one pass
- After significant refactoring or architectural changes
- Symptoms: "ready to merge", "need review", "pre-merge check"

**Not for:** trivial single-line changes, branches with no diff

## Workflow

1. Run the script below
2. Read each agent's output — focus on CRITICAL and HIGH findings first
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
You are a senior code review coordinator. Create the following 5 named agents and have them review the git diff below IN PARALLEL. Each agent produces its own independent report.

## Agents

1. **typescript-pro** — Expert TypeScript reviewer. Focus: type safety, strict mode, generics, utility types, type narrowing, module boundaries
2. **reviewer** — Senior code reviewer. Focus: code quality, readability, naming, maintainability, DRY, coding standards, large functions (>50 lines), large files (>800 lines), deep nesting (>4 levels), dead code
3. **code-reviewer** — Security-focused code reviewer. Focus: hardcoded credentials, SQL injection, XSS, path traversal, CSRF, auth bypasses, insecure dependencies, exposed secrets in logs, unvalidated input, missing rate limiting, N+1 queries
4. **architect-reviewer** — Software architecture reviewer. Focus: architectural patterns, separation of concerns, dependency management, scalability, coupling/cohesion, API design
5. **react-specialist** — React and frontend performance expert. Focus: missing dependency arrays, state updates in render, missing keys, prop drilling, unnecessary re-renders, client/server boundary, missing loading/error states, stale closures, memoization

## Confidence Filter

Only report issues with >80% confidence. Consolidate similar findings. Skip stylistic preferences unless they violate project conventions. Skip issues in unchanged code unless CRITICAL.

## Output Format

Each agent produces a titled section. For each issue found:

\`\`\`
[SEVERITY] Issue Title
File: path/to/file.ts:lineNumber
Issue: Description of the problem and its impact.
Fix: Suggested resolution with code example.

  code example // BAD
  code example // GOOD
\`\`\`

Severity levels: CRITICAL, HIGH, MEDIUM, LOW

Each agent concludes with:

\`\`\`
## [agent-name] Review Summary

| Severity | Count | Status |
|----------|-------|--------|
| CRITICAL | 0     | pass   |
| HIGH     | 2     | warn   |
| MEDIUM   | 3     | info   |
| LOW      | 1     | note   |

Verdict: [APPROVE/WARNING/BLOCK] — explanation
\`\`\`

## Approval Criteria

- **Approve**: No CRITICAL or HIGH issues
- **Warning**: HIGH issues only (can merge with caution)
- **Block**: CRITICAL issues found — must fix before merge

## Git diff to review:

${DIFF}
EOF
)"

echo ""
echo "=== Review complete ==="
```

## Review Output Format

Each agent outputs findings organized by severity:

```
[CRITICAL] Hardcoded API key in source
File: src/api/client.ts:42
Issue: API key "sk-abc..." exposed in source code. This will be committed to git history.
Fix: Move to environment variable and add to .gitignore/.env.example

  const apiKey = "sk-abc123";           // BAD
  const apiKey = process.env.API_KEY;   // GOOD
```

Each agent concludes with a summary table:

```
## [agent-name] Review Summary

| Severity | Count | Status |
|----------|-------|--------|
| CRITICAL | 0     | pass   |
| HIGH     | 2     | warn   |
| MEDIUM   | 3     | info   |
| LOW      | 1     | note   |

Verdict: WARNING — 2 HIGH issues should be resolved before merge.
```

## Approval Criteria

- **Approve**: No CRITICAL or HIGH issues
- **Warning**: HIGH issues only (can merge with caution)
- **Block**: CRITICAL issues found — must fix before merge

## Common Mistakes

| Mistake | Fix |
|---|---|
| Base branch not found | Falls back to `main`. Set `origin/HEAD` if using different default |
| Diff too large for prompt | Split review by directory or use `git diff --stat` to limit scope |
| Missing `codex` CLI | Install first: `npm i -g @openai/codex` |
| Ignoring WARNING verdicts | Evaluate each HIGH issue — document exceptions or fix before merge |
