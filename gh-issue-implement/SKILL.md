---
name: gh-issue-implement
description: Implement GitHub issue by number. Fetches issue details via gh cli, calls codeagent skill (backend=codex) for development, creates PR linked to issue. Requires 90% test coverage.
---

# GitHub Issue Implementation

## Role in Architecture

```
Claude Code (主控)
    │
    └── gh-issue-implement (本 skill)
            │
            ├── Claude Code 执行: issue 获取、repo clone (并行模式)、分支创建、push、PR 创建
            │
            └── 委托 Codeagent (Codex):
                  - 代码实现
                  - 测试编写
                  - 测试验证
                  - git commit (本地)
```

**Key Principle**: Claude Code does NOT write code directly. All implementation delegated to codeagent.

## Overview

Given an issue number, fetch its details and implement the feature. Uses codeagent skill (codex backend) for development, ensuring 90% test coverage, then creates a PR.

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
| workdir | No | Working directory (default: current). For parallel mode, use isolated clone path like `/tmp/repo-clone-<issue-num>` |

## Workflow

1. **Fetch issue** via `gh issue view <number>`
2. **Create dedicated branch** (每个 issue 必须独立分支):
   ```bash
   git checkout main && git pull origin main
   git checkout -b feature/issue-<number>-<short-desc>
   ```
3. **Call codeagent skill** with issue content (backend=codex)
4. **Push branch & create PR**:
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
                    └── commits by codeagent
```

1. **创建**: 从最新 main 分支切出
2. **开发**: codeagent 在分支上提交代码
3. **推送**: `git push -u origin <branch>`
4. **PR**: 创建 PR 并关联 issue
5. **合并**: review 通过后 squash merge
6. **清理**: 合并后删除 feature 分支

## Codeagent Invocation

> **前置条件**: 执行前需加载 `codeagent` skill，了解超时设置（默认 2 小时）和任务状态检查方法。

**Command**:
```bash
codeagent-wrapper --backend codex - <<'EOF'
<task content>
EOF
```

**Task Content Template**:
```
## Issue #<number>: <title>

## Requirements
<issue body content>

## Acceptance Criteria
<from issue, or derive from requirements>

## Technical Constraints
- Test coverage >= 90%
- All tests must pass before completion
- Follow existing code patterns in the repository
- Include unit tests for all new functions

## Context Files
@<relevant source files>
@<relevant test files>
@<config files if needed>

## Deliverables
1. Implementation code
2. Unit tests with >= 90% coverage
3. Run tests and verify passing
4. Git commit with descriptive message
```

**Codeagent Responsibilities**:
- Analyze requirements and plan implementation
- Write implementation code
- Write comprehensive unit tests
- Run tests and fix failures
- Verify coverage meets 90% threshold
- Git commit (local)

**Claude Code Responsibilities** (before/after codeagent):
- Fetch issue details via `gh issue view`
- Create feature branch
- After codeagent completes: push and create PR

## Parallel Mode Usage

When called by gh-flow in parallel mode, this skill operates in isolated repo clones:

**Serial Mode (default)**:
```bash
# In main repo
gh-issue-implement 123
# Creates branch, calls codeagent, pushes, creates PR
```

**Parallel Mode (via gh-flow)**:
```bash
# gh-flow creates isolated clone first
CLONE_PATH=/tmp/repo-clone-123
git clone <repo-url> $CLONE_PATH

# Then calls gh-issue-implement with workdir
gh-issue-implement 123 --workdir $CLONE_PATH

# Workflow in isolated clone:
cd $CLONE_PATH
git checkout -b feature/issue-123-xxx
# codeagent implements in $CLONE_PATH
git push -u origin feature/issue-123-xxx
gh pr create --title "..." --body "Closes #123"

# After PR merged, gh-flow cleans up:
rm -rf $CLONE_PATH
```

**Key Points**:
- Isolated clones prevent branch conflicts during concurrent development
- Each issue gets its own `/tmp/repo-clone-<issue-num>` directory
- Cleanup happens after PR merge
- Serial merge order ensures no conflicts

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
| Tests fail | Codeagent retries until pass |
| Push rejected | Pull and rebase, then retry push |

## Related Skills

- [gh-flow](../gh-flow/SKILL.md) - Workflow orchestration (parallel mode)
- [gh-pr-review](../gh-pr-review/SKILL.md) - Review created PRs
