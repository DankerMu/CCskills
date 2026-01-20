---
name: gh-pr-review
description: Review and merge GitHub PR. Performs code review via /review command, checks CI status. Pass → auto-merge; Fail → comment issues and directly fix. Returns standardized JSON result.
---

# GitHub PR Review & Merge

## Role in Architecture

See [gh-flow Architecture](../gh-flow/SKILL.md#architecture) for complete workflow diagram.

**This skill**: Claude Code performs review via /review command and directly fixes issues.

**Key Principle**:
- Code review by Claude Code (via /review command)
- Bug fixes by Claude Code directly
- Claude Code orchestrates and implements all changes

## Overview

Given a PR number, perform comprehensive code review and handle the result. Automatically merges on approval, or triggers fixes via gh-issue-implement on failure.

## When to Use

- User provides a PR number and requests review
- Called by gh-flow during review phase
- Automated PR review in CI/CD pipelines

## Usage

**Invoke via skill call:**
```
Call gh-pr-review skill with:
  - input: pr_number (e.g., 200)
  - output: JSON { status, merged_sha, issues }
```

## Parameters

| Parameter | Required | Default | Description |
|-----------|----------|---------|-------------|
| pr_number | Yes | - | GitHub PR number to review |
| merge_strategy | No | squash | Merge method: squash, merge, rebase |
| auto_fix | No | true | Auto-fix issues on failure |
| max_retries | No | 2 | Max fix attempts before marking BLOCKED |

## Workflow

**合并条件: CI 通过 + Review 通过，缺一不可**

1. **Fetch PR info** via `gh pr view <number>`
2. **Check CI status** via `gh pr checks <number>`
3. **Perform code review** via `/review` command on PR changes
4. **Merge decision** (必须同时满足):
   - CI pass + Review approved → Merge
   - CI fail 或 Review fail → 直接修复，不合并

## Code Review via /review

**Trigger review:**
Use the `/review` command to review all changes in the PR:
```bash
# Checkout PR branch
gh pr checkout <PR_NUMBER>

# Review changes
/review

# Review focuses on:
# - Code quality and maintainability
# - Security vulnerabilities
# - Performance implications
# - Test coverage
# - Best practices
```

The /review command will:
- Analyze all changed files
- Identify issues and improvements
- Provide actionable feedback
- Suggest specific fixes

## Return Format

**Success (merged):**
```json
{
  "pr_number": 200,
  "status": "MERGED",
  "merged_sha": "abc123def456",
  "issues": []
}
```

**Failure:**
```json
{
  "pr_number": 200,
  "status": "CHANGES_REQUESTED",
  "merged_sha": null,
  "issues": ["High cyclomatic complexity in auth.ts", "Missing unit tests"]
}
```

## Status Codes

| Status | Meaning | Action |
|--------|---------|--------|
| `MERGED` | Review passed, PR merged | None |
| `CI_FAILED` | CI checks failed | Call gh-issue-implement |
| `CHANGES_REQUESTED` | Code review failed | Call gh-issue-implement |
| `BLOCKED` | Conflicts or protection rules | Manual intervention |

**Integration**: See [gh-flow Status Codes](../gh-flow/SKILL.md#status-codes-parallel-mode) for orchestration mapping.

## Auto-Fix Flow

When `auto_fix=true` and issues are identified in review or CI fails:

```bash
# Retry loop (max_retries=2 by default)
for attempt in 1..max_retries:
  # Checkout PR branch
  gh pr checkout <PR_NUMBER>

  # Fix identified issues directly
  # - Address review comments
  # - Fix failing tests
  # - Improve code quality

  # Commit and push fixes
  git add .
  git commit -m "Fix review issues: <summary>"
  git push

  # Wait for CI cycle

  if CI pass and no review issues:
    merge PR
    return SUCCESS

# After max_retries exhausted
return BLOCKED with issues list
```

**Retry behavior**:
- Each retry creates new commits on same branch
- No exponential backoff (CI is async)
- After max_retries: mark PR as BLOCKED, require manual intervention

## Review Checklist

| Check | Required | Description |
|-------|----------|-------------|
| CI pass | ✅ | All required checks green |
| PR size | ⚠️ | Warn if > 500 lines changed |
| Issue linked | ✅ | PR references issue correctly |
| Code quality | ✅ | Meets project standards |
| Security | ✅ | No obvious vulnerabilities |
| Tests | ✅ | New code has test coverage |

## Error Handling

| Error | Resolution |
|-------|------------|
| PR not found | Return error JSON |
| No merge permission | Return BLOCKED |
| Merge conflict | Return BLOCKED with conflict info |
| CI pending | Wait or return BLOCKED |
| Review command fails | Retry or return BLOCKED |

## Related Skills

- [gh-flow](../gh-flow/SKILL.md) - Workflow orchestration
- [gh-issue-implement](../gh-issue-implement/SKILL.md) - Issue implementation
