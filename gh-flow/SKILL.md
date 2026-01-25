---
name: gh-flow
description: |
  GitHub 完整开发工作流编排器。从 PRD 到代码合并的端到端自动化流程。
  串联调用 gh-create-issue、gh-issue-implement、gh-pr-review、gh-release 四个 skills。
  触发条件：用户要求"完整实现某功能"、"从需求到上线"、"端到端开发"时使用此技能。
---

# GitHub Flow Orchestrator

## Architecture

```
gh-flow (编排层)
  │
  ├── Call gh-create-issue    → 分析需求、创建 issues
  │
  ├── For each issue:
  │     ├── Call gh-issue-implement → 实现代码、创建 PR
  │     └── Call gh-pr-review       → Review、合并 PR
  │
  └── Call gh-release (可选)  → 生成 release notes
```

**原则**: Claude Code 只编排和决策，不直接写代码。

## Parameters

| Parameter | Default | Description |
|-----------|---------|-------------|
| prd_content | required | PRD or requirements |
| mode | manual | auto (无人值守) / manual (每步确认) |
| generate_release | false | 完成后生成 release |

## Workflow

### Stage 1: Issue Creation

```
Call gh-create-issue skill:
  - input: prd_content
  - output: { epic_num, issues: [{num, title, depends_on}, ...] }
```

### Stage 2: Implementation (按依赖顺序)

```
For each issue:

  Step 1: Call gh-issue-implement skill
    - input: issue_number
    - output: { pr_number }
    - 仅创建 PR，不 merge

  Step 2: Call gh-pr-review skill (必须调用，不可跳过)
    - input: pr_number
    - 执行: CI 检查 → 本地 codeagent review → 根据结果决定 merge 或修复
    - output: { status: MERGED|CHANGES_REQUESTED|CI_FAILED|BLOCKED }

  等待 Step 2 返回 status=MERGED 才继续下一个 issue
```

**重要**: gh-issue-implement 只创建 PR，merge 决策由 gh-pr-review 负责。

### Stage 3: Release (可选)

```
If generate_release=true:
  Call gh-release skill:
    - input: epic_number, merged_prs
```

## Execution Modes

| Mode | Behavior |
|------|----------|
| auto | 无人值守，仅错误时暂停 |
| manual | 每个主要步骤前确认 |

## Partial Execution

```
--stage create-issue --prd "content"   → Call gh-create-issue
--stage implement --issues 101,102     → Call gh-issue-implement + gh-pr-review
--stage review --prs 200,201           → Call gh-pr-review
--stage release --tag v1.0             → Call gh-release
```

## Return Format

```
========================================
GitHub Flow Complete
========================================
📋 Issues: 3
💻 PRs: 3
✅ Merged: 3
📦 Release: v1.2.0

Epic #100:
- [x] #101 Login API
- [x] #102 JWT management
- [x] #103 Permission middleware
========================================
```

## Error Handling

| Stage | Error | Resolution |
|-------|-------|------------|
| Issue creation | Permission denied | Check gh auth |
| Implementation | Tests fail | gh-issue-implement 内部重试 |
| Review | CI/Review fail | gh-pr-review 内部调用 gh-issue-implement 修复 |
| Merge | Conflict | BLOCKED, 需手动解决 |

## Related Skills

- [gh-create-issue](../gh-create-issue/SKILL.md)
- [gh-issue-implement](../gh-issue-implement/SKILL.md)
- [gh-pr-review](../gh-pr-review/SKILL.md)
- [gh-release](../gh-release/SKILL.md)
