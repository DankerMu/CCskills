---
name: gh-issue-implement
description: |
  Implement a GitHub issue end-to-end via gh+git: fetch issue details, create a feature branch, implement code + tests, run checks, commit, push, and open a PR that closes the issue.
  Use when a user asks to implement issue #N or as part of gh-flow.
---

# GitHub Issue Implementation (Codex)

## Goal

给定 `issue_number`，在当前仓库中完成实现并创建一个关联 issue 的 PR（`Closes #<issue_number>`）。

## Parameters (conceptual)

| Parameter | Required | Default | Description |
|-----------|----------|---------|-------------|
| issue_number | Yes | - | GitHub issue number |
| base_branch | No | repo default | Base branch to branch from (usually `main`) |
| draft_pr | No | false | Create PR as draft |

## Workflow

1. **Preflight**
   ```bash
   gh auth status
   git status --porcelain
   ```

2. **Fetch issue details**
   ```bash
   gh issue view <ISSUE_NUMBER> --json number,title,body,state,labels,url
   ```
   - 若 issue 已关闭：停止并询问是否跳过/重新打开。

3. **Determine base branch**
   - 优先使用仓库默认分支：
     ```bash
     gh repo view --json defaultBranchRef --jq .defaultBranchRef.name
     ```

4. **Create branch**
   - 命名：`feature/issue-<number>-<short-slug>`（2-4 个词，kebab-case）
   ```bash
   git fetch origin
   git checkout <BASE_BRANCH>
   git pull --ff-only origin <BASE_BRANCH>
   git checkout -b feature/issue-<number>-<short-slug>
   ```

5. **Implement code + tests**
   - 按仓库既有结构实现功能与必要文档
   - 为新增/修改逻辑添加单元测试（或仓库主流测试类型）
   - 运行仓库测试/校验（按项目实际命令）

6. **Commit**
   ```bash
   git add -A
   git commit -m "<type>: <summary> (#<ISSUE_NUMBER>)"
   ```

7. **Push**
   ```bash
   git push -u origin feature/issue-<number>-<short-slug>
   ```

8. **Create PR**
   ```bash
   gh pr create \
     --title "<issue title>" \
     --body $'Closes #<ISSUE_NUMBER>\n\n## Summary\n- ...\n\n## Testing\n- ...\n'
   ```
   - 若该分支已有 PR：改为更新分支提交，并在现有 PR 下继续工作。

## Return Format (for orchestration)

```json
{
  "issue_number": 123,
  "branch": "feature/issue-123-user-login",
  "pr_number": 200,
  "url": "https://github.com/owner/repo/pull/200"
}
```

## Error Handling

| Error | Resolution |
|-------|------------|
| Issue not found | 确认仓库/编号是否正确 |
| Issue closed | 询问是否跳过或 `gh issue reopen <n>` |
| Dirty working tree | stash/commit 或在新目录操作 |
| Branch exists | `git checkout <branch>` 并确认与远端一致 |
| Push rejected | `git pull --rebase` 后重试（避免强推，必要时先确认） |
| Tests fail | 优先修复测试与实现，再提交/推送 |

## Related Skills

- [gh-flow](../gh-flow/SKILL.md) - End-to-end orchestration
- [gh-pr-review](../gh-pr-review/SKILL.md) - Review/merge PR after implementation
