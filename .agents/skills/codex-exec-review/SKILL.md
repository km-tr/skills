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

## Scope

This skill **only reviews and reports** — it does not fix issues or merge branches. The output is the verdict itself.

## Workflow

1. Run the script below
2. Read each agent's output — focus on CRITICAL and HIGH findings first
3. Report the consolidated verdict to the user

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
You are a senior code review coordinator. Run the following 5 existing named agents to review the git diff below IN PARALLEL. Each agent produces its own independent report.

## Agents to run

1. **typescript-pro**
2. **reviewer**
3. **code-reviewer**
4. **architect-reviewer**
5. **react-specialist**

## Confidence Filter

Only report issues with >80% confidence. Consolidate similar findings. Skip stylistic preferences unless they violate project conventions. Skip issues in unchanged code unless CRITICAL.

## Review Output Format

Organize findings by severity. For each issue:

\`\`\`
[CRITICAL] Hardcoded API key in source
File: src/api/client.ts:42
Issue: API key "sk-abc..." exposed in source code. This will be committed to git history.
Fix: Move to environment variable and add to .gitignore/.env.example

  const apiKey = "sk-abc123";           // BAD
  const apiKey = process.env.API_KEY;   // GOOD
\`\`\`

### Summary Format

End every review with:

\`\`\`
## Review Summary

| Severity | Count | Status |
|----------|-------|--------|
| CRITICAL | 0     | pass   |
| HIGH     | 2     | warn   |
| MEDIUM   | 3     | info   |
| LOW      | 1     | note   |

Verdict: WARNING — 2 HIGH issues should be resolved before merge.
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

Organize findings by severity. For each issue:

```
[CRITICAL] Hardcoded API key in source
File: src/api/client.ts:42
Issue: API key "sk-abc..." exposed in source code. This will be committed to git history.
Fix: Move to environment variable and add to .gitignore/.env.example

  const apiKey = "sk-abc123";           // BAD
  const apiKey = process.env.API_KEY;   // GOOD
```

### Summary Format

End every review with:

```
## Review Summary

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
| Ignoring WARNING verdicts | Evaluate each HIGH issue — document exceptions or fix before merge |
