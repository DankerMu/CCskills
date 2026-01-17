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
            ├── Claude Code 执行: issue 获取、分支创建、push、PR 创建
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

## Workflow

1. **Fetch issue** via `gh issue view <number>`
2. **Create branch** `feature/issue-<number>-<short-desc>`
3. **Call codeagent skill** with issue content (backend=codex)
4. **Push & create PR** linked to issue with `Closes #<number>`

## Branch Naming

```
feature/issue-{number}-{short-description}

Examples:
- feature/issue-123-user-login
- feature/issue-456-jwt-management
```

## Codeagent Invocation

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
| Branch exists | Switch to existing branch |
| PR exists | Update existing PR |
| Tests fail | Codeagent retries until pass |
