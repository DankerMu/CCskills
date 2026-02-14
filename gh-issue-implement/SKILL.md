---
name: gh-issue-implement
description: Implement GitHub issue by number. Fetches issue details via gh cli, calls codeagent skill (backend=codex) for development, creates PR linked to issue. Requires 90% test coverage.
---

# GitHub Issue Implementation

## Architecture

```
Claude Code (主控)
    │
    └── gh-issue-implement
            ├── Claude Code: issue获取、分支创建、push、PR创建
            └── 委托 Codeagent (Codex): 代码实现、测试、commit
```

**Key Principle**: Claude Code does NOT write code directly. All implementation delegated to codeagent.

## Parameters

| Parameter | Required | Description |
|-----------|----------|-------------|
| issue_number | Yes | GitHub issue number to implement |

## Workflow

```
1. gh issue view <N>                           → Get issue details
2. git checkout main && git pull
3. git checkout -b feature/issue-<N>-<desc>    → Create branch
4. Call codeagent skill (backend=codex)        → Implement + test (可能需要数小时；不要因为耗时而中断/kill)
5. git push -u origin <branch>
6. gh pr create --body "Closes #<N>"           → Create PR
```

## Branch Naming

```
feature/issue-{number}-{short-description}

Examples:
- feature/issue-123-user-login
- feature/issue-456-jwt-management
```

## Codeagent Invocation

### Long-running `codeagent-wrapper` (IMPORTANT)

`codeagent-wrapper` 执行时间可能长达数小时。**不要因为运行时间过长而 SIGINT / kill 掉进程**（除非用户明确要求取消，或你有充分证据表明进程已卡死）。

- `timeout`/等待时间建议按任务复杂度动态设置：简单 **30m（1800000ms）** / 中等 **1h（3600000ms）** / 复杂 **2h（7200000ms）**（不确定先用 1h）。
- 避免使用 shell 的 `timeout ...` 这类“到点自动 kill”的包装器；优先使用执行环境/工具层的 timeout 配置（例如 `CODEX_TIMEOUT`），并在需要时延长等待（不要 kill）。
- 如果 `codeagent-wrapper` 输出了 `session_id`，请记录；意外中断时可用 `codeagent-wrapper resume ...` 恢复（以 `codeagent-wrapper --help` 为准）。

**Command**:
```bash
codeagent-wrapper --backend codex - <<'EOF'
<task content>
EOF
```

**Task Template**:
```
## Issue #<number>: <title>

## Requirements
<issue body content>

## Technical Constraints
- Test coverage >= 90%
- All tests must pass
- Follow existing code patterns

## Context Files
@<relevant files>

## Deliverables
1. Implementation code
2. Unit tests (>= 90% coverage)
3. Git commit
```

## Return Format

```json
{
  "issue_number": 123,
  "branch": "feature/issue-123-user-login",
  "pr_number": 200,
  "coverage": "92%",
  "ci_status": "pending"
}
```

## PR Template

```markdown
## Summary
Implements #<issue_number>

## Changes
- Change 1
- Change 2

## Testing
- [x] Unit tests passing
- [x] Coverage >= 90%

Closes #<issue_number>
```

## Error Handling

| Error | Resolution |
|-------|------------|
| Issue not found | Return error |
| Branch exists | Switch to existing, verify clean |
| `codeagent-wrapper` takes a long time | Expected sometimes; **do not kill by runtime**. Wait; only intervene if the user cancels or it's clearly stuck. |
| Tests fail | Codeagent retries until pass |
| Push rejected | Rebase and retry |
