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

## Overview

End-to-end development workflow from PRD to merged code. Orchestrates gh-create-issue, gh-issue-implement, gh-pr-review, and gh-release skills.

## When to Use

- User requests "complete implementation of a feature"
- User requests "end-to-end development"
- User requests "from requirements to release"

## Usage

**Invoke via skill call:**
```
Call gh-flow skill with:
  - input: PRD or requirements
  - mode: auto | manual
  - generate_release: true | false
  - output: Execution report
```

## Parameters

| Parameter | Required | Default | Description |
|-----------|----------|---------|-------------|
| prd_content | Yes | - | PRD or requirements document |
| mode | No | manual | auto (unattended) or manual (confirm each step) |
| parallel | No | false | Enable parallel execution by dependency layers |
| max_concurrency | No | 3 | Max concurrent issues in parallel mode |
| generate_release | No | false | Generate release after merge |

## Workflow

### Serial Mode (parallel=false, default)
```
For each issue:
  Issue → Branch → Code → PR → Review → Merge
                                    ↓
                              Next Issue
```

### Parallel Mode (parallel=true)
```
Stage 1: Create issues with dependency info
Stage 2: Build dependency DAG → Topological sort → Execution layers
Stage 3: Execute layers (issues in same layer run concurrently)
Stage 4: Release (optional)
```

### Stage 1: Issue Creation
```
Call gh-create-issue skill:
  input: PRD content
  output: { epic_num, issues: [{ num, title, priority, depends_on }] }
```

### Stage 2: Dependency Analysis (parallel mode only)
```
# Build dependency graph
GRAPH = {}
For each issue in issues:
  GRAPH[issue.num] = issue.depends_on

# Detect cycles
If has_cycle(GRAPH):
  Error: "Circular dependency detected"

# Topological sort into layers (Kahn's algorithm)
LAYERS = topological_sort_layers(GRAPH)
# Example output:
#   LAYERS[0] = [101, 104]  # No dependencies
#   LAYERS[1] = [102]       # Depends on 101
#   LAYERS[2] = [103]       # Depends on 102
```

### Stage 3a: Serial Development Loop (parallel=false)
```
MERGED_PRS = []

For each issue_num in ISSUE_LIST:
  # Implement
  Call gh-issue-implement skill:
    input: issue_num
    output: pr_num

  # Review & Merge (retry until success)
  Loop:
    Call gh-pr-review skill:
      input: pr_num
      output: { status, merged_sha, issues }

    If status == MERGED:
      Append pr_num to MERGED_PRS
      Break → Next issue
    Elif status == BLOCKED:
      Pause for manual resolution
    Else:
      Wait for fix cycle (auto-triggered by gh-pr-review)
      Continue loop
```

### Stage 3b: Parallel Development Loop (parallel=true)
```
MERGED_PRS = []
FAILED_ISSUES = []
SKIPPED_ISSUES = []

For each layer in LAYERS:
  # Filter out issues blocked by failed dependencies
  executable = filter_by_deps(layer, FAILED_ISSUES)
  skipped = layer - executable
  SKIPPED_ISSUES.extend(skipped)

  # Execute issues in this layer concurrently (up to max_concurrency)
  results = parallel_for issue_num in executable (max=max_concurrency):
    # Implement
    Call gh-issue-implement skill:
      input: issue_num
      output: pr_num

    # Review & Merge (retry until success)
    Loop:
      Call gh-pr-review skill:
        input: pr_num
        output: { status, merged_sha, issues }

      If status == MERGED:
        return { issue_num, pr_num, status: SUCCESS }
      Elif status == BLOCKED:
        return { issue_num, pr_num, status: BLOCKED }
      Else:
        Wait for fix cycle
        Continue loop (max 3 retries)
    return { issue_num, pr_num, status: FAILED }

  # Collect results
  For each result in results:
    If result.status == SUCCESS:
      Append result.pr_num to MERGED_PRS
    Else:
      Append result.issue_num to FAILED_ISSUES

  # Wait for all in layer before proceeding to next layer
  wait_all()
```

### Stage 4: Release (optional)
```
If generate_release && MERGED_PRS not empty:
  Call gh-release skill:
    input: MERGED_PRS, EPIC_NUM
    output: Release URL
```

## Serial vs Parallel Mode

### Serial Mode (parallel=false, default)
```
issue1 → PR1 → review1 → merge1
                            ↓
issue2 → PR2 → review2 → merge2
                            ↓
issue3 → PR3 → review3 → merge3
```

Benefits:
- Simple, predictable execution order
- No branch conflicts between PRs
- Suitable for tightly coupled issues

### Parallel Mode (parallel=true)
```
Given dependencies:
  101 ← 102 ← 103
  104 (independent)

Execution:
  Layer 0: [101, 104] → concurrent
           ↓
  Layer 1: [102]      → after Layer 0 complete
           ↓
  Layer 2: [103]      → after Layer 1 complete
```

Benefits:
- Faster for independent issues
- Respects dependency order
- Configurable concurrency limit

### Merge Strategy in Parallel Mode

Within same layer:
- All issues develop concurrently on separate branches
- Merge in issue number order (deterministic)
- If conflict detected, rebase and retry once

Between layers:
- Lower layer must fully complete before upper layer starts
- Ensures dependency code is merged first

## Skill Orchestration

```
gh-flow (orchestrator)
    │
    ├── gh-create-issue
    │
    ├── gh-issue-implement ──→ codeagent (codex)
    │
    ├── gh-pr-review ──→ codeagent (codex)
    │       │
    │       └──→ gh-issue-implement (on failure)
    │
    └── gh-release
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
| Concurrent merge conflict | Rebase and retry once, then BLOCKED |
| Max retries exceeded | Mark as FAILED, continue with other issues |

### Status Codes (Parallel Mode)

| Status | Meaning |
|--------|---------|
| SUCCESS | Issue implemented and PR merged |
| BLOCKED | Requires manual intervention |
| FAILED | Exceeded retry limit |
| DEPENDENCY_BLOCKED | Skipped due to failed dependency |
| SKIPPED | Filtered out before execution |

### Return Format (Parallel Mode)

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
