---
name: gh-release
description: Generate Release Notes and publish a GitHub Release via gh CLI. Calculate a version tag and generate notes from merged PRs (auto/custom/epic modes). Use after PRs are merged (often as the last stage of gh-flow).
---

# GitHub Release Notes Generation

## Role in gh-flow

Used by [gh-flow](../gh-flow/SKILL.md) Stage 3 to generate release notes and publish a release via `gh release create`.

## Overview

Given a list of merged PRs, calculate version number, generate Release Notes, and create a GitHub Release. Supports three generation modes.

## When to Use

- User requests Release Notes generation
- User requests publishing a new version
- Called by gh-flow after PR merges

## Usage

**Invoke via skill call:**
```
Call gh-release skill with:
  - input: merged PR numbers (e.g., 200, 201, 202)
  - options: epic_number, version_tag, mode
  - output: Release URL
```

## Parameters

| Parameter | Required | Default | Description |
|-----------|----------|---------|-------------|
| pr_list | Yes | - | List of merged PR numbers |
| epic_number | No | - | Epic issue number for epic mode |
| version_tag | No | auto | Version tag (e.g., v1.2.0) |
| mode | No | custom | Generation mode: auto, custom, epic |

## Version Calculation

```
If user specifies tag → use specified tag
Else if no previous releases → v1.0.0
Else → increment patch version (vX.Y.Z → vX.Y.Z+1)
```

## Generation Modes

### Auto Mode
Uses GitHub's built-in release notes generation:
```bash
gh release create <tag> --generate-notes
```

### Custom Mode (default)
Groups PRs by label and formats:
```markdown
## What's Changed

### 🚀 Features
* Feature title (#200) @author

### 🐛 Bug Fixes
* Fix title (#201) @author

### 📦 Other Changes
* Other title (#202) @author

### 👥 Contributors
author1 author2
```

### Epic Mode
Generates from Epic issue content:
```markdown
## <Epic Title>

<Epic Overview>

### Included PRs
* PR title (#200)
* PR title (#201)

### Related Issues
* Epic: #100
* #101, #102, #103
```

## Return Format

```json
{
  "tag": "v1.2.0",
  "url": "https://github.com/owner/repo/releases/tag/v1.2.0",
  "pr_count": 3,
  "mode": "custom"
}
```

**Terminal output:**
```
Version: v1.1.0 → v1.2.0

## What's Changed
...

✅ Release v1.2.0 created
URL: https://github.com/owner/repo/releases/tag/v1.2.0
```

## Label Mapping

| Label | Category |
|-------|----------|
| `feature`, `feat` | 🚀 Features |
| `bug`, `fix` | 🐛 Bug Fixes |
| other | 📦 Other Changes |

## Error Handling

| Error | Resolution |
|-------|------------|
| Tag exists | Auto-increment or prompt for new tag |
| No merged PRs | Return error, skip release |
| Epic not found | Fallback to custom mode |
| Permission denied | Return error with permission info |

## Related Skills

- [gh-flow](../gh-flow/SKILL.md) - Workflow orchestration
- [gh-create-issue](../gh-create-issue/SKILL.md) - Epic structure for epic mode
