---
name: gh-flow
description: |
  GitHub 端到端开发工作流编排（Codex 版）：从 PRD/需求 → 创建 issues（可含 epic）→ 实现 → PR review/merge → release。
  串联 gh-create-issue、gh-issue-implement、gh-pr-review、gh-release 四个 skills。
  仅支持串行执行（无并行/多目录 clone）。
  触发：用户要求“端到端开发 / 从需求到上线 / 完整实现某功能 / 需求→issue→PR→release 全流程”时使用。
---

# GitHub Flow Orchestrator (Codex)

## What this skill does

- 把 PRD/需求拆解为可执行的 GitHub issues（可选：epic + 子任务 + 依赖）
- 按依赖顺序串行实现每个 issue（每个 issue 一个 PR）
- 审查 PR、按需修复，并在 CI 通过时合并
- 可选：基于合并 PR 生成 Release Notes 并发布 GitHub Release

## Inputs (optional)

- `prd_content`: Stage 1 的输入（若你已有 issue 列表，可不提供）
- `mode`: `manual`(default) 或 `auto`
- `generate_release`: `true|false` (default: `false`)
- `stage`: `all|create-issue|implement|review|release`
- `issues`: 已存在的 issue numbers（跳过 Stage 1）
- `prs`: 已存在的 PR numbers（跳过 Stage 2）

## Workflow (serial only)

### Stage 0: Preconditions

```bash
gh auth status
git status --porcelain
```

如未登录：执行 `gh auth login` 并确认对目标仓库有 issue/PR/release 权限。

### Stage 1: Create Issues

按 `../gh-create-issue/SKILL.md` 执行，产出结构化结果供编排使用。

建议输出（示例）：
```json
{
  "epic_num": 100,
  "issues": [
    { "num": 101, "title": "Login API implementation", "priority": 2, "depends_on": [] },
    { "num": 102, "title": "JWT token management", "priority": 2, "depends_on": [101] }
  ]
}
```

### Stage 2: Implement → Review → Merge（逐个 issue）

执行顺序：
1. 先按 `depends_on` 做拓扑排序（依赖优先）；无依赖时按 `priority` → `issue number` 排序
2. 每个 issue 完整跑完一轮：「实现 → review/修复 → CI 绿 → 合并」，再开始下一个

编排伪代码：
```
issues := topo_sort_by_depends_on(issues)
for issue in issues:
  pr := gh-issue-implement(issue.num)
  result := gh-pr-review(pr.pr_number)
  if result.status != MERGED:
    stop and report (BLOCKED/FAILED)
```

### Stage 3: Release (optional)

当 `generate_release=true` 且目标 PR 已合并，按 `../gh-release/SKILL.md` 执行：
- 需要 epic 视角：用 `mode=epic`（传 `epic_number`）
- 只按 PR 列表：用 `mode=custom` 或 `mode=auto`

## Manual mode confirmations

- Stage 1 完成后：确认 issue 列表、依赖关系、实现顺序
- 每个 PR 合并前：确认 merge 策略（默认 squash）与是否要合并
- 发布 release 前：确认 tag 与 release notes

## Status Codes (for summaries)

| Status | Meaning |
|--------|---------|
| SUCCESS | Issue 已实现且 PR 已合并 |
| BLOCKED | 需要人工介入（权限/冲突/保护规则等） |
| FAILED | 自动修复重试耗尽仍失败 |
| DEPENDENCY_BLOCKED | 依赖未合并，无法继续 |
| SKIPPED | 用户选择跳过 |

## Return summary (example)

```
========================================
GitHub Flow Complete
========================================
Issues processed: 3
PRs created: 3
PRs merged: 3
Release: v1.2.0

Epic #100 Progress:
- [x] #101 Login API
- [x] #102 JWT management
- [x] #103 Permission middleware
========================================
```

## Related Skills

- [gh-create-issue](../gh-create-issue/SKILL.md) - Issue creation and epic management
- [gh-issue-implement](../gh-issue-implement/SKILL.md) - Issue implementation (code + tests + PR)
- [gh-pr-review](../gh-pr-review/SKILL.md) - PR review and merge (with CI)
- [gh-release](../gh-release/SKILL.md) - Release notes generation and publishing
