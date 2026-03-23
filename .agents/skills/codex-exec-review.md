# codex-exec-review

Run parallel code reviews on the current branch's changes using 5 specialized agents via `codex exec`.

## Usage

This skill detects the base branch, generates a git diff, and launches 5 `codex exec` processes in parallel with model `gpt-5.4` and effort `high`. Each agent reviews the changes from its own specialty perspective.

### Agents

| Agent | Focus Area |
|---|---|
| typescript-pro | TypeScript type safety, patterns, and best practices |
| reviewer | General code quality, readability, and maintainability |
| code-reviewer | Bugs, logic errors, edge cases, and security vulnerabilities |
| architect-reviewer | Architecture, design patterns, and scalability |
| react-specialist | React patterns, hooks, performance, and rendering optimization |

## Script

```bash
#!/usr/bin/env bash
set -euo pipefail

# Detect base branch
BASE_BRANCH=$(git symbolic-ref refs/remotes/origin/HEAD 2>/dev/null | sed 's@^refs/remotes/origin/@@' || echo "main")
DIFF=$(git diff "${BASE_BRANCH}"...HEAD)

if [ -z "$DIFF" ]; then
  echo "No changes detected against ${BASE_BRANCH}. Nothing to review."
  exit 0
fi

echo "=== Codex Exec Parallel Review ==="
echo "Base branch: ${BASE_BRANCH}"
echo "Model: gpt-5.4 | Effort: high"
echo "Launching 5 review agents in parallel..."
echo ""

DIFF_FILE=$(mktemp)
echo "$DIFF" > "$DIFF_FILE"
trap 'rm -f "$DIFF_FILE"' EXIT

REVIEW_PROMPT_SUFFIX=$(cat <<'PROMPT_END'

Review the git diff provided below. Organize findings by severity. For each issue, use this structure:

```
[SEVERITY] Issue Title
File: path/to/file.ts:lineNumber
Issue: Description of the problem and its impact.
Fix: Suggested resolution with code examples.

  code example showing BAD pattern
  code example showing GOOD pattern
```

Conclude your review with a summary table:

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

PROMPT_END
)

# --- typescript-pro ---
codex exec -m gpt-5.4 --effort high "$(cat <<EOF
You are **typescript-pro**, an expert TypeScript reviewer.
Focus on: type safety, strict mode compliance, generic usage, utility types, type narrowing, module boundaries, and TypeScript best practices.

${REVIEW_PROMPT_SUFFIX}

Git diff to review:
$(cat "$DIFF_FILE")
EOF
)" &

# --- reviewer ---
codex exec -m gpt-5.4 --effort high "$(cat <<EOF
You are **reviewer**, a senior code reviewer.
Focus on: overall code quality, readability, naming conventions, maintainability, DRY principle, documentation, and adherence to coding standards.

${REVIEW_PROMPT_SUFFIX}

Git diff to review:
$(cat "$DIFF_FILE")
EOF
)" &

# --- code-reviewer ---
codex exec -m gpt-5.4 --effort high "$(cat <<EOF
You are **code-reviewer**, a security-focused code reviewer.
Focus on: bugs, logic errors, edge cases, null/undefined handling, race conditions, error handling, input validation, and security vulnerabilities (OWASP top 10).

${REVIEW_PROMPT_SUFFIX}

Git diff to review:
$(cat "$DIFF_FILE")
EOF
)" &

# --- architect-reviewer ---
codex exec -m gpt-5.4 --effort high "$(cat <<EOF
You are **architect-reviewer**, a software architecture reviewer.
Focus on: architectural patterns, separation of concerns, dependency management, scalability, coupling/cohesion, API design, and system-level implications of the changes.

${REVIEW_PROMPT_SUFFIX}

Git diff to review:
$(cat "$DIFF_FILE")
EOF
)" &

# --- react-specialist ---
codex exec -m gpt-5.4 --effort high "$(cat <<EOF
You are **react-specialist**, a React and frontend performance expert.
Focus on: React component patterns, hooks usage (rules of hooks, custom hooks), re-render optimization, state management, memoization (useMemo/useCallback/React.memo), accessibility, and React best practices.

${REVIEW_PROMPT_SUFFIX}

Git diff to review:
$(cat "$DIFF_FILE")
EOF
)" &

echo "Waiting for all agents to complete..."
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
