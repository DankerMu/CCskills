---
name: gh-flow
description: |
  GitHub 完整开发工作流编排器。从 PRD 到代码合并的端到端自动化流程。
  串联调用 gh-create-issue、gh-issue-implement、gh-pr-review、gh-release 四个 skills。
  支持串行模式和并发模式（按依赖关系分层并发执行）。
  支持全自动模式和半自动模式（关键步骤需用户确认）。
  触发条件：用户要求"完整实现某功能"、"从需求到上线"、"端到端开发"时使用此技能。
---

# GitHub Flow Orchestrator

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    Claude Code (主控)                        │
│         调度 skills / 审核结果 / 决策 / 不写代码              │
├─────────────────────────────────────────────────────────────┤
│  gh-flow (编排层)                                            │
│    │                                                         │
│    ├── gh-create-issue ──→ 分析需求、拆分任务                 │
│    │                                                         │
│    ├── gh-issue-implement ──→ 委托 codeagent 实现            │
│    │                                                         │
│    ├── gh-pr-review ──→ 触发 @codex review + 解析结果        │
│    │                                                         │
│    └── gh-release ──→ 生成 release notes                    │
│                                                              │
├──────────────────────────┬──────────────────────────────────┤
│                          │ 委托执行                          │
│                          ▼                                   │
│         ┌────────────────────────────────┐                  │
│         │  Codeagent + Codex Backend     │                  │
│         │  - 代码实现                     │                  │
│         │  - 测试编写                     │                  │
│         │  - 文档编写                     │                  │
│         │  - Code Review                 │                  │
│         └────────────────────────────────┘                  │
└─────────────────────────────────────────────────────────────┘
```

**Responsibility Separation:**
- **Claude Code**: Orchestration, verification, decision-making. NO direct code/test/doc writing.
- **Skills**: Define workflow steps and invoke codeagent.
- **Codeagent (Codex)**: Execute ALL implementation tasks.

## Parameters

| Parameter | Required | Default | Description |
|-----------|----------|---------|-------------|
| prd_content | Yes | - | PRD or requirements document |
| mode | No | manual | auto (unattended) or manual (confirm each step) |
| parallel | No | false | Enable parallel execution by dependency layers |
| max_concurrency | No | 3 | Max concurrent issues in parallel mode |
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

### Stage 2: Implementation

**Serial Mode (parallel=false, default)**:
```bash
# Issue 101
gh issue view 101
git checkout -b feature/issue-101-login-api
# Call gh-issue-implement → codeagent implements
git push -u origin feature/issue-101-login-api
gh pr create --title "Login API" --body "Closes #101"
# Call gh-pr-review → merge PR #200

# Issue 102 (after 101 merged)
git checkout main && git pull
git checkout -b feature/issue-102-jwt-management
# ... repeat process
```

**Parallel Mode (parallel=true)**:

并行模式使用独立 repo 副本避免分支冲突：

```bash
# Layer 0: Issues 101 and 104 (no dependencies) - concurrent development

# Task 1: Issue 101 in isolated repo clone
CLONE_101=/tmp/repo-clone-101
git clone <repo-url> $CLONE_101

# Call gh-issue-implement with workdir=$CLONE_101
# gh-issue-implement will:
#   cd $CLONE_101
#   git checkout -b feature/issue-101-login-api
#   call codeagent (codex) to implement in $CLONE_101
#   git push -u origin feature/issue-101-login-api
#   gh pr create --title "Login API" --body "Closes #101"  # PR #200

# Task 2: Issue 104 (concurrent with Task 1, same process)
CLONE_104=/tmp/repo-clone-104
git clone <repo-url> $CLONE_104
# Call gh-issue-implement with workdir=$CLONE_104 → PR #201

# Review & Merge (serial, in issue number order)
gh pr view 200 --json statusCheckRollup,reviewDecision
# If CI fail or review rejected: call gh-issue-implement with workdir to fix, retry
gh pr merge 200 --squash
rm -rf $CLONE_101

gh pr view 201 --json statusCheckRollup,reviewDecision
# If fail: same fix process
gh pr merge 201 --squash
rm -rf $CLONE_104

# Layer 1: Issue 102 (depends on 101) - after Layer 0 complete
git checkout main && git pull
git checkout -b feature/issue-102-jwt-management
# Call gh-issue-implement in main repo
```

### Stage 3: Release (optional)
```bash
# If generate_release=true
# Pass epic_number from Stage 1 (gh-create-issue output)
gh release create v1.2.0 --generate-notes --notes "Epic #100 complete"

# Or call gh-release skill with epic mode:
# gh-release --mode epic --epic_number 100 --prs 200,201,202
```

## Serial vs Parallel Mode

### Serial Mode (parallel=false, default)
```
issue1 → branch → code → PR1 → review → merge
                                    ↓
issue2 → branch → code → PR2 → review → merge
```

**Benefits**:
- Simple, single repo directory
- No branch conflicts
- Suitable for tightly coupled issues

### Parallel Mode (parallel=true)
```
Given dependencies: 101 ← 102 ← 103, 104 (independent)

Layer 0 (concurrent development):
  /tmp/repo-clone-101: issue 101 → PR #200
  /tmp/repo-clone-104: issue 104 → PR #201

Layer 0 (serial merge):
  Review PR #200 → merge → cleanup /tmp/repo-clone-101
  Review PR #201 → merge → cleanup /tmp/repo-clone-104

Layer 1:
  Main repo: issue 102 → PR #202 → review → merge

Layer 2:
  Main repo: issue 103 → PR #203 → review → merge
```

**Benefits**:
- Faster for independent issues (concurrent development)
- Respects dependency order (layered execution)
- Isolated repo clones prevent branch conflicts

**Key Mechanism**:
- Each concurrent issue uses isolated repo clone (`/tmp/repo-clone-<issue-num>`)
- Development happens in parallel
- Review and merge happen serially (in issue number order)
- Cleanup repo clone after merge
- Next layer starts after previous layer fully merged

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

### Common Errors

| Stage | Error | Resolution |
|-------|-------|------------|
| Issue creation | Permission denied | Check gh auth scope |
| Development | Tests fail | Handled by codeagent |
| Review | CI fail | gh-pr-review triggers fix |
| Review | Changes requested | gh-pr-review triggers fix |
| Merge | Conflict | Return BLOCKED, manual resolution |
| Release | Tag exists | gh-release auto-increments |

### Parallel Mode Specific

| Error | Resolution |
|-------|------------|
| Circular dependency | Abort with error, require manual fix in issues |
| Dependency failed | Skip all downstream issues, mark as DEPENDENCY_BLOCKED |
| Repo clone exists | Remove existing clone: `rm -rf /tmp/repo-clone-<num>` |
| Clone cleanup fail | Log warning, continue (orphaned clones cleaned by system) |
| Concurrent merge conflict | Impossible (merge is serial), but if occurs: rebase in clone and retry |
| Max retries exceeded | Mark as FAILED, cleanup clone, continue with other issues |

### Status Codes (Parallel Mode)

| Status | Meaning |
|--------|---------|
| SUCCESS | Issue implemented and PR merged |
| BLOCKED | Requires manual intervention |
| FAILED | Exceeded retry limit |
| DEPENDENCY_BLOCKED | Skipped due to failed dependency |
| SKIPPED | Filtered out before execution |

### Status Code Mapping

Integration with gh-pr-review status codes:

| gh-pr-review | gh-flow | Meaning |
|--------------|---------|---------|
| MERGED | SUCCESS | PR merged successfully |
| CI_FAILED | FAILED | CI checks failed, retry exhausted |
| CHANGES_REQUESTED | FAILED | Code review failed, retry exhausted |
| BLOCKED | BLOCKED | Manual intervention required |

## Return Format

**Serial Mode**:
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

**Parallel Mode** (adds execution layers and status details):
```
========================================
GitHub Flow Complete (Parallel)
========================================
📋 Issues: 4
💻 PRs: 3
✅ Merged: 3
❌ Failed: 0
⏭️ Skipped: 1 (dependency blocked)
📦 Release: v1.2.0

Execution Layers:
Layer 0: ✅ #101, ✅ #104
Layer 1: ✅ #102
Layer 2: ⏭️ #103 (blocked by #102)

Epic #100 Progress:
- [x] #101 Login API
- [x] #102 JWT management
- [ ] #103 Permission middleware (skipped)
- [x] #104 Logging module
========================================
```

## Related Skills

- [gh-create-issue](../gh-create-issue/SKILL.md) - Issue creation and epic management
- [gh-issue-implement](../gh-issue-implement/SKILL.md) - Issue implementation
- [gh-pr-review](../gh-pr-review/SKILL.md) - PR review and merge
- [gh-release](../gh-release/SKILL.md) - Release notes generation
