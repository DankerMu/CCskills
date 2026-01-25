---
name: gh-pr-review
description: |
  Review and merge GitHub PR. Uses local codeagent for review, checks CI, auto-merges on pass or calls gh-issue-implement on fail.

  Input: pr_number → Output: JSON { status, score, issues }

  Part of gh-flow workflow. See gh-flow for architecture.
---

# GitHub PR Review & Merge

## Workflow

```
1. gh pr view <N>              → Get PR info + diff
2. gh pr checks <N>            → Check CI status
3. Call codeagent skill:       → Local review
     backend=codex
     task: "Review PR #<N>" + diff content
4. Decision:
   - CI pass + Score≥4 → gh pr merge
   - Otherwise → gh-issue-implement to fix
```

**Review 完成后**: `gh pr comment` post 结果，`gh pr review --approve/--request-changes` 提交决定。

## Parameters

| Parameter | Default | Description |
|-----------|---------|-------------|
| pr_number | required | PR number to review |
| merge_strategy | squash | squash/merge/rebase |
| auto_fix | true | Auto-fix on failure |
| max_retries | 2 | Fix attempts before BLOCKED |

## Review Standards

### Scoring

| Score | Label | Action |
|-------|-------|--------|
| 5 | LGTM 👍 | Merge |
| 4 | Nitpicks 🤓 | Merge |
| 3 | Needs Work 🔧 | Fix → gh-issue-implement |
| 2 | Needs a Lot of Work 🚨 | Fix |
| 1 | Abandon ❌ | BLOCKED, manual intervention |

### Categories & Severity

| Category | Focus |
|----------|-------|
| Correctness 🎯 | Bugs, logic errors, null access |
| Quality ✨ | Structure, DRY, naming |
| Testing 🧪 | Missing tests, edge cases |
| Security 🔒 | Injection, secrets, XSS |

| Severity | Merge Impact |
|----------|--------------|
| 🔴 HIGH | Blocks merge |
| 🟡 MEDIUM | Blocks if >2 |
| 🔵 LOW | Does not block |

### Comment Format

```
🔴 HIGH | Security | `file.py:42`
> ```python
> problematic_code()
> ```
Issue description.
```

## Return Format

```json
{
  "pr_number": 200,
  "status": "MERGED|CHANGES_REQUESTED|CI_FAILED|BLOCKED",
  "score": 5,
  "label": "LGTM",
  "merged_sha": "abc123|null",
  "issues": [{"severity": "HIGH", "category": "Security", "file": "auth.ts", "issue": "..."}]
}
```

## Error → Status

| Error | Status |
|-------|--------|
| CI failed | CI_FAILED → fix |
| Review failed | CHANGES_REQUESTED → fix |
| Conflict/no permission | BLOCKED → manual |
