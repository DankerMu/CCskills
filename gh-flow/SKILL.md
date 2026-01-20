---
name: gh-flow
description: |
  GitHub 完整开发工作流编排器。从 PRD 到代码合并的端到端自动化流程。
  串联调用 gh-create-issue、gh-issue-implement、gh-pr-review、gh-release 四个 skills。
  支持全自动模式和半自动模式（关键步骤需用户确认）。
  触发条件：用户要求"完整实现某功能"、"从需求到上线"、"端到端开发"时使用此技能。
---

# GitHub Flow Orchestrator

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    Claude Code (主控)                        │
│         调度 skills / 审核结果 / 决策 / 直接实现代码          │
├─────────────────────────────────────────────────────────────┤
│  gh-flow (编排层)                                            │
│    │                                                         │
│    ├── gh-create-issue ──→ 分析需求、拆分任务                 │
│    │                                                         │
│    ├── gh-issue-implement ──→ 直接实现代码和测试              │
│    │                                                         │
│    ├── gh-pr-review ──→ 使用 /review 命令审查代码            │
│    │                                                         │
│    └── gh-release ──→ 生成 release notes                    │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

**Responsibility Separation:**
- **Claude Code**: Orchestration, implementation, code review, and decision-making.
- **Skills**: Define workflow steps and execute tasks.

**Skill Loading (执行前必须加载对应 skill):**
| Stage | Skill | 加载命令 |
|-------|-------|---------|
| Issue 创建 | gh-create-issue | `/gh-create-issue` |
| Issue 实现 | gh-issue-implement | `/gh-issue-implement` |
| PR Review | gh-pr-review | `/gh-pr-review` |
| Release | gh-release | `/gh-release` |

## Parameters

| Parameter | Required | Default | Description |
|-----------|----------|---------|-------------|
| prd_content | Yes | - | PRD or requirements document |
| mode | No | manual | auto (unattended) or manual (confirm each step) |
| generate_release | No | false | Generate release after merge |

## Workflow

### Stage 1: Issue Creation
```bash
# Call gh-create-issue skill
# Output: { epic_num: 100, issues: [
#   { num: 101, title: "Login API", depends_on: [] },
#   { num: 102, title: "JWT management", depends_on: [101] },
#   { num: 104, title: "Logging", depends_on: [] }
# ]}
```

### Stage 2: Implementation (串行模式)

```bash
# Issue 101
# Step 1: gh-issue-implement → 直接实现代码 + 创建 PR #200
# Step 2: gh-pr-review #200 → CI 检测 + /review 审查 + 合并
#         (CI pass + Review pass 才合并，否则修复重试)

# Issue 102 (after 101 merged)
# ... repeat: gh-issue-implement → gh-pr-review

# Issue 104 (after 102 merged)
# ... repeat: gh-issue-implement → gh-pr-review
```

### Stage 3: Release (optional)
```bash
# If generate_release=true
# Pass epic_number from Stage 1 (gh-create-issue output)
gh release create v1.2.0 --generate-notes --notes "Epic #100 complete"

# Or call gh-release skill with epic mode:
# gh-release --mode epic --epic_number 100 --prs 200,201,202
```

## Execution Modes

| Mode | Behavior |
|------|----------|
| auto | Runs unattended, pauses only on errors |
| manual | Confirms before each major step |

**Manual mode checkpoints:**
- After issue creation: "Continue to development?"
- After each PR: "Continue to next issue?"
- Before merge: "Merge this PR?"
- Before release: "Generate release?"

## Partial Execution

Start from specific stage:
```
--stage create-issue --prd "content"     # Only create issues
--stage implement --issues 101,102,103   # Start from existing issues
--stage review --prs 200,201,202         # Only review/merge PRs
--stage release --prs 200,201 --tag v1.0 # Only generate release
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

Epic #100 Progress:
- [x] #101 Login API
- [x] #102 JWT management
- [ ] #103 Permission middleware
========================================
```

## Prerequisites

```bash
gh auth status   # Must be authenticated
command -v jq    # Required for JSON parsing
command -v git   # Required for version control
```

## Error Handling

| Stage | Error | Resolution |
|-------|-------|------------|
| Issue creation | Permission denied | Check gh auth scope |
| Development | Tests fail | Fix and retry |
| Review | CI fail | gh-pr-review triggers fix |
| Review | Changes requested | gh-pr-review triggers fix |
| Merge | Conflict | Return BLOCKED, manual resolution |
| Release | Tag exists | gh-release auto-increments |

### Status Code Mapping

Integration with gh-pr-review status codes:

| gh-pr-review | gh-flow | Meaning |
|--------------|---------|---------|
| MERGED | SUCCESS | PR merged successfully |
| CI_FAILED | FAILED | CI checks failed, retry exhausted |
| CHANGES_REQUESTED | FAILED | Code review failed, retry exhausted |
| BLOCKED | BLOCKED | Manual intervention required |

## Return Format

```
========================================
GitHub Flow Complete
========================================
📋 Issues: 3
💻 PRs: 3
✅ Merged: 3
📦 Release: v1.2.0

Epic #100 Progress:
- [x] #101 Login API
- [x] #102 JWT management
- [x] #103 Permission middleware
========================================
```

## Related Skills

- [gh-create-issue](../gh-create-issue/SKILL.md) - Issue creation and epic management
- [gh-issue-implement](../gh-issue-implement/SKILL.md) - Issue implementation
- [gh-pr-review](../gh-pr-review/SKILL.md) - PR review and merge
- [gh-release](../gh-release/SKILL.md) - Release notes generation
