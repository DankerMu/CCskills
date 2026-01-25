---
name: gh-pr-review
description: |
  Review and (optionally) merge a GitHub PR via gh: fetch metadata, inspect diff, run/check CI, request changes or apply fixes, and merge when ready.
  Use when a user asks to review/merge a PR or as part of gh-flow.
---

# GitHub PR Review & Merge (Codex)

## Goal

给定 `pr_number`，完成「信息获取 → diff 审查 →（可选）修复 → CI 通过 → 合并」并返回标准化结果。

## Priority Rule (P0/P1 必须修复)

- **合并前必须修复所有 P0 与 P1 问题**（否则不得 merge，返回 `CHANGES_REQUESTED` 或 `BLOCKED`）。
- **P2/P3** 可作为建议项保留（用评论/清单记录），是否修复由用户决定。

优先级建议定义：
- **P0**：阻断合并（安全漏洞、数据损坏/丢失、构建或测试失败、严重回归等）
- **P1**：阻断合并（明显功能缺陷、关键路径缺测试、明显性能回退等）
- **P2**：可延期（非关键缺陷、可读性/维护性改进）
- **P3**：nit（格式/命名/非阻断建议）

## Parameters (conceptual)

| Parameter | Required | Default | Description |
|-----------|----------|---------|-------------|
| pr_number | Yes | - | PR number |
| merge_strategy | No | squash | `squash`, `merge`, `rebase` |
| auto_fix | No | true | 尝试在 PR 分支上直接修复可自动修复的问题 |
| max_retries | No | 2 | 自动修复轮次上限 |
| mode | No | manual | `manual` 合并前确认，`auto` 直接合并 |

## Workflow

1. **Fetch PR info**
   ```bash
   gh pr view <PR_NUMBER> --json number,title,body,state,isDraft,mergeable,baseRefName,headRefName,author,url
   ```

2. **Check CI**
   ```bash
   gh pr checks <PR_NUMBER>
   ```
   - 若 CI pending：可选择等待（`gh pr checks <PR_NUMBER> --watch`）或先做代码审查再决定。

3. **Review diff**
   - 快速查看：
     ```bash
     gh pr diff <PR_NUMBER>
     ```
   - 需要本地运行测试/复现时：
     ```bash
     gh pr checkout <PR_NUMBER>
     ```
     然后按仓库约定运行测试/lint。

4. **Decision**
   - 先将发现的问题按 **P0/P1/P2/P3** 分级，并在输出 `issues` 中用前缀标注（例如：`P0: ...`）。
   - **CI 失败**：视为 **P0**；若 `auto_fix=true`，在 PR 分支上修复后推送，等待 CI 复跑；最多 `max_retries` 次。
   - **代码需要改动**：P0/P1 优先直接修复（小改动）并推送；否则用 `gh pr review --request-changes` 给出可执行清单（必须覆盖所有 P0/P1）。
   - **通过**：CI 全绿且 **无未解决 P0/P1** 时才进入合并步骤（P2/P3 可保留为建议项）。

5. **Merge**
   - `mode=manual`：合并前向用户确认。
   - 执行合并：
     ```bash
     gh pr merge <PR_NUMBER> --<merge_strategy> --delete-branch
     ```

## Return Format

```json
{
  "pr_number": 200,
  "status": "MERGED",
  "merged_sha": "abc123def456",
  "issues": []
}
```

## Status Codes

| Status | Meaning |
|--------|---------|
| MERGED | PR 已合并 |
| CI_FAILED | CI 失败（重试耗尽或 auto_fix=false） |
| CHANGES_REQUESTED | 审查发现问题且需要作者修改 |
| BLOCKED | 权限/冲突/保护规则等导致无法继续 |

## Error Handling

| Error | Resolution |
|-------|------------|
| PR not found | 确认仓库/编号是否正确 |
| No merge permission | 返回 `BLOCKED` 并提示权限问题 |
| Merge conflict | 返回 `BLOCKED` 并提示需要 rebase/resolve |
| Branch protection | 使用 `gh pr merge --auto` 或等待 required checks/approvals |

## Related Skills

- [gh-flow](../gh-flow/SKILL.md) - Orchestration
- [gh-issue-implement](../gh-issue-implement/SKILL.md) - Implement/fix code when needed
