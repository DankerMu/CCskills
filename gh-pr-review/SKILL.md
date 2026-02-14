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
1. gh pr checks <N>            → Check CI status
2. Get PR metadata + diff      → Head SHA is required for marker gate
3. Call codeagent skill:       → Local review (可能需要数小时；不要因为耗时而中断/kill)
     backend=codex
     task: "Review PR #<N>" + diff content
4. Post review comment         → Include required marker (if gate enabled)
5. Verify merge gates          → `gh-pr-review` status check becomes success
6. Decision:
   - CI pass + Score≥4 + no MEDIUM/HIGH issues + gate pass → gh pr merge
   - Otherwise → gh-issue-implement to fix
```

### Long-running `codeagent-wrapper` (IMPORTANT)

PR diff 很大时，`codeagent-wrapper` 可能运行数小时。**不要因为运行时间过长而 SIGINT / kill 掉进程**（除非用户明确要求取消，或你有充分证据表明进程已卡死）。

- `timeout`/等待时间建议按任务复杂度动态设置：简单 **30m（1800000ms）** / 中等 **1h（3600000ms）** / 复杂 **2h（7200000ms）**（不确定先用 1h）。
- 避免使用 shell 的 `timeout ...` 这类“到点自动 kill”的包装器；优先使用执行环境/工具层的 timeout 配置（例如 `CODEX_TIMEOUT`），并在需要时延长等待（不要 kill）。
- 如果 `codeagent-wrapper` 输出了 `session_id`，请记录；意外中断时可用 `codeagent-wrapper resume ...` 恢复（以 `codeagent-wrapper --help` 为准）。

**Review 完成后**:
- Always: `gh pr comment` post 结果（并在仓库启用 gate 时包含 marker）
- Only if branch protection requires it: `gh pr review --approve/--request-changes`

## Parameters

| Parameter | Default | Description |
|-----------|---------|-------------|
| pr_number | required | PR number to review |
| merge_strategy | squash | squash/merge/rebase |
| auto_fix | true | Auto-fix on failure |
| max_retries | 2 | Fix attempts before BLOCKED |

## Merge gate: `gh-pr-review` comment marker

Some repos enforce a required status check `gh-pr-review` that is set by a GitHub Actions “comment marker” gate.

### Marker format (single line)

```text
<!-- gh-pr-review: sha=<SHA> decision=PASS|FAIL score=<1-5> -->
```

Rules:
- `sha` MUST match the PR head commit SHA (7-char short SHA prefix is accepted).
- `decision=PASS` AND `score>=4` → gate sets status check `gh-pr-review=success`.
- Otherwise → `gh-pr-review=failure` (merge blocked).

### Get head SHA (REST; avoids GraphQL flakiness)

```bash
gh api repos/<OWNER>/<REPO>/pulls/<PR_NUMBER> --jq .head.sha
```

Fetch the head SHA **right before** posting the review comment to avoid races (new commits require a new marker).

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
| 🟡 MEDIUM | Blocks merge (must fix) |
| 🔵 LOW | Does not block |

**Rule:** Any 🟡 MEDIUM or 🔴 HIGH issues must be fixed before merging. If you find any, set `decision=FAIL` and use a score ≤ 3 so the workflow routes to `$gh-issue-implement`.

### Comment Format

```
🔴 HIGH | Security | `file.py:42`
> ```python
> problematic_code()
> ```
Issue description.
```

### Gate marker (append at the end when enabled)

```text
<!-- gh-pr-review: sha=<HEAD_SHA> decision=PASS|FAIL score=<1-5> -->
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
