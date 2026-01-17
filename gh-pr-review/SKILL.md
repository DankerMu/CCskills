---
name: gh-pr-review
description: Review and merge GitHub PR. Performs code review via codeagent (codex backend), checks CI status. Pass → auto-merge; Fail → comment issues and call gh-issue-implement for fixes. Returns standardized JSON result.
---

# GitHub PR Review & Merge

## Role in Architecture

```
Claude Code (主控)
    │
    └── gh-pr-review (本 skill)
            │
            ├── Claude Code 执行: PR 信息获取、CI 状态检查、merge 操作
            │
            ├── 触发 @codex review: 通过 PR comment 触发
            │
            ├── Claude Code 审核: 解析 codex review 结果、决策是否合并
            │
            └── 问题修复: 委托 Codeagent (Codex) 通过 gh-issue-implement
```

**Key Principle**:
- Code review by Codex (via @codex comment)
- Bug fixes by Codeagent (via gh-issue-implement)
- Claude Code only orchestrates and makes decisions, NO direct code writing

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
| auto_fix | No | true | Auto-call gh-issue-implement on failure |

## Workflow

1. **Fetch PR info** via `gh pr view <number>`
2. **Check CI status** via `gh pr checks <number>`
3. **Trigger Codex review** via `gh pr comment <number> --body "@codex review"`
4. **Poll for review result** via `gh pr view <number> --json reviews`
5. **Process result**:
   - Pass + CI pass → Approve & merge
   - CI fail → Comment + trigger fix
   - Review fail → Request changes + trigger fix
   - Blocked → Comment issue

## Code Review via Codex

**Review Time Expectation:**
- GitHub Codex review 通常需要 **5～10 分钟** 完成
- 轮询间隔：首次 5 分钟后检查，之后每 3 分钟检查一次
- 超时阈值：**15 分钟**无响应视为超时

**Trigger review by commenting on PR:**
```bash
# Basic review
gh pr comment <PR_NUMBER> --body "@codex review"

# With focus areas
gh pr comment <PR_NUMBER> --body "@codex review
Focus on:
- Security vulnerabilities
- Performance implications
- Test coverage gaps"
```

**Fetch review results (MUST use both commands):**
```bash
# Get all comments AND reviews - DO NOT skip any!
gh pr view <PR_NUMBER> --comments && \
echo "=== Reviews ===" && \
gh api repos/<OWNER>/<REPO>/pulls/<PR_NUMBER>/reviews --jq '.[] | {
  author: .user.login,
  state: .state,
  body: .body,
  submitted_at: .submitted_at
}'
```

**Why both commands:**
- `--comments`: Captures inline comments and discussion
- `/reviews` API: Captures formal review decisions (APPROVED/CHANGES_REQUESTED)
- Using only one will miss review opinions!

**Review dimensions:**
- Code quality (naming, complexity, DRY)
- Security (injection, XSS, secrets)
- Performance (N+1, algorithm complexity)
- Test coverage (critical paths, edge cases)
- Requirements alignment (vs issue acceptance criteria)

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

## Auto-Fix Flow

When `auto_fix=true` and status is `CI_FAILED` or `CHANGES_REQUESTED`:

```
Call gh-issue-implement skill with:
  - input: linked issue number
  - task: Fix issues from PR review comments
  - push to same branch
  - triggers new CI/review cycle
```

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

**Codex Review Fallback (超时/用量耗尽):**

当出现以下情况时，切换到 Codeagent 本地 review：
1. **超时**：15 分钟内无 Codex review 响应
2. **用量耗尽**：GitHub Codex comment 配额用尽

**Fallback 执行方式：**
```
调用 codeagent skill:
  - backend: codex
  - 命令: /review PR#<PR_NUMBER>
  - codeagent 自行获取 PR diff 并执行 review
```

**判断用量耗尽的信号：**
- Codex 回复包含 "Codex usage limits have been reached for code reviews"
- 15 分钟内无任何 Codex review comment
- `gh pr comment` 返回 403 或 rate limit 错误

| Error | Resolution |
|-------|------------|
| PR not found | Return error JSON |
| No merge permission | Return BLOCKED |
| Merge conflict | Return BLOCKED with conflict info |
| CI pending | Wait or return BLOCKED |
