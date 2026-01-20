---
name: gh-issue-implement
description: Implement GitHub issue by number. Fetches issue details via gh cli, directly implements code and tests, creates PR linked to issue. Requires 90% test coverage.
---

# GitHub Issue Implementation

## Role in Architecture

```
Claude Code (主控)
    │
    └── gh-issue-implement (本 skill)
            │
            └── Claude Code 执行:
                  - issue 获取
                  - 分支创建
                  - 代码实现
                  - 测试编写和验证
                  - git commit
                  - push
                  - PR 创建
```

**Key Principle**: Claude Code directly implements code, tests, and commits changes.

## Overview

Given an issue number, fetch its details and directly implement the feature. Ensures 90% test coverage, then creates a PR.

## When to Use

- User provides a GitHub issue number and requests implementation
- Called by gh-flow during development phase
- Called by gh-pr-review for fix requests

## Usage

**Invoke via skill call:**
```
Call gh-issue-implement skill with:
  - input: issue_number (e.g., 123)
  - output: PR number
```

## Parameters

| Parameter | Required | Description |
|-----------|----------|-------------|
| issue_number | Yes | GitHub issue number to implement |

## Workflow

1. **Fetch issue** via `gh issue view <number>`
2. **Create dedicated branch**:
   ```bash
   git checkout main && git pull origin main
   git checkout -b feature/issue-<number>-<short-desc>
   ```
3. **Implement code and tests**:
   - Write implementation code following existing patterns
   - Write comprehensive unit tests (>=90% coverage)
   - Run tests and fix any failures
   - Verify coverage threshold
4. **Commit changes**:
   ```bash
   git add .
   git commit -m "Implement #<number>: <title>"
   ```
5. **Push branch & create PR**:
   ```bash
   git push -u origin feature/issue-<number>-<short-desc>
   gh pr create --title "..." --body "Closes #<number>"
   ```

## Branch Strategy

**核心原则：每个 Issue 必须有独立分支**

每个 issue 的实现必须在独立的 feature 分支上进行，确保：
- 代码变更隔离，便于 review
- PR 与 issue 一一对应
- 支持并行开发多个 issue
- 方便回滚单个功能

### 分支创建流程

```bash
# 1. 确保在最新的 main 分支
git checkout main
git pull origin main

# 2. 创建 feature 分支
git checkout -b feature/issue-<number>-<short-desc>

# 示例
git checkout -b feature/issue-123-user-login
```

### 分支命名规范

```
feature/issue-{number}-{short-description}

格式说明:
- feature/    : 固定前缀，表示功能开发
- issue-      : 标识关联的 GitHub issue
- {number}    : issue 编号（必须）
- {short-desc}: 简短描述，用连字符连接（2-4个单词）

Examples:
- feature/issue-123-user-login
- feature/issue-456-jwt-management
- feature/issue-789-add-export-csv
```

### 分支生命周期

```
main ──────────────────────────────────────────► main
       │                                    ▲
       │ git checkout -b                    │ merge (squash)
       ▼                                    │
       feature/issue-123-xxx ──► PR #200 ───┘
                    │
                    └── commits by Claude Code
```

1. **创建**: 从最新 main 分支切出
2. **开发**: 直接在分支上编写代码和测试
3. **提交**: git commit 提交变更
4. **推送**: `git push -u origin <branch>`
5. **PR**: 创建 PR 并关联 issue
6. **合并**: review 通过后 squash merge
7. **清理**: 合并后删除 feature 分支

## Implementation Guidelines

**Code Quality:**
- Follow existing code patterns in the repository
- Keep functions small and focused
- Use clear, descriptive variable names
- Add comments only for non-obvious logic

**Testing:**
- Write unit tests for all new functions
- Test happy path and edge cases
- Test error handling
- Verify >= 90% test coverage

**Commit Message:**
```
Implement #<issue_number>: <short description>

- Bullet point summary of changes
- Focus on what and why, not how

Closes #<issue_number>
```

## Return Format

```
Issue Development Complete:
- Issue: #123 - User login feature
- Branch: feature/issue-123-user-login
- PR: #200

Changed Files:
- src/auth/login.ts (+120, -5)
- src/auth/login.test.ts (+200)

Test Coverage: 92%
CI Status: pending
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

## Epic Detection

Sub-issues are identified by:
- Having `sub-task` label
- Body contains `Part of #<epic>` or `Epic: #<epic>`

When detected, verifies epic exists and has `epic` label before associating.

## Error Handling

| Error | Resolution |
|-------|------------|
| Issue not found | Return error |
| Issue closed | Prompt to reopen or skip |
| Branch exists | Switch to existing branch, verify clean state |
| Branch diverged | Rebase on main: `git rebase main` |
| PR exists | Update existing PR |
| Tests fail | Fix and retry |
| Push rejected | Pull and rebase, then retry push |

## Related Skills

- [gh-flow](../gh-flow/SKILL.md) - Workflow orchestration
- [gh-pr-review](../gh-pr-review/SKILL.md) - Review created PRs
