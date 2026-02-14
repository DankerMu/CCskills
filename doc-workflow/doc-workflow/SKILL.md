---
name: doc-workflow
description: |
  DR-driven iterative document workflow that produces high-quality documents through
  multi-round deep-research validation. Orchestrates brainstorming, analysis, deep-research,
  and codeagent skills into an automated pipeline with parent-child document clusters.
  Use when: user wants to create a rigorously validated document (PRD, paper, tech design,
  or any document type), says "doc-workflow", "DR workflow", "document iteration",
  "iterative document refinement", or needs a document that has been verified through
  multiple rounds of deep research before finalization.
---

# Doc Workflow

DR-driven iterative document workflow. Produces validated documents through multi-round
deep-research cycles that form parent-child document clusters.

## Quick Start

```
/doc-workflow <idea or document path> [--type prd|paper|tech-design|custom]
```

**Input detection:**
- Path to existing `.md` file → skip to Phase 2 (ANALYZE)
- Text description → start from Phase 1 (DRAFT)

**Default type:** `prd`. Override with `--type`.

## Workflow Overview

```
DRAFT → ANALYZE → RESEARCH → LINK → REWRITE → REVIEW
                                                  ↓
                                        score ≥ 90 AND new_dr = 0 → FINALIZE
                                        otherwise → back to ANALYZE (version++)
```

7 phases, tracked via `manifest.json`. Each iteration produces a versioned directory with
main document, checklist, DR reports, and review.

## Directory Structure

```
docs/dr-workflow/<project>/
├── manifest.json
├── v1/
│   ├── main.md
│   ├── checklist.md
│   ├── dr/
│   │   ├── dr-001-<slug>.md
│   │   └── ...
│   └── review.md
├── v2/ ...
└── final/
    ├── main.md
    └── milestones.md
```

## Phase Execution

### Phase 1: DRAFT

**If input is a file path:**
1. Copy file to `v1/main.md`
2. Initialize `manifest.json` with `current_phase: "ANALYZE"`, `current_version: 1`
3. Skip to Phase 2

**If input is an idea:**
1. Initialize `manifest.json` with `current_phase: "DRAFT"`, `current_version: 1`
2. Create directory structure: `docs/dr-workflow/<project>/v1/` and `v1/dr/`
3. Invoke `brainstorming` skill interactively to refine the idea
4. Apply document template based on `--type` (see `references/templates.md`)
5. Write result to `v1/main.md`
6. Update manifest: `current_phase: "ANALYZE"`

### Phase 2: ANALYZE

Read `vN/main.md` and generate a checklist of suspicious points.

Prompt approach — act as a senior reviewer and identify:
1. Claims with questionable technical feasibility
2. Conclusions lacking data or evidence
3. Architecture choices not validated
4. Unverified market or product assumptions
5. Contradictions with existing DR reports (v2+ only)

Output format for `vN/checklist.md`:
```markdown
# Checklist v{N}

| # | Type | Suspicious Point | Research Direction |
|---|------|------------------|--------------------|
| 1 | tech | ... | ... |
| 2 | product | ... | ... |
```

Type tags: `tech`, `product`, `market`, `architecture`, `data`, `methodology`, `feasibility`.

Update manifest: `current_phase: "RESEARCH"`.

### Phase 3: RESEARCH

Parse `vN/checklist.md` into individual items. For each item, invoke `deep-research` skill.

**Parallel strategy:**
- Items ≤ 5 → all parallel via Task agents
- Items > 5 → batch of 5, sequential batches

Each DR report writes to `vN/dr/dr-{NNN}-{slug}.md` using the format defined in
`references/dr-template.md`.

**Error handling:** If a DR item fails (network, search failure):
- Mark as `status: skipped` in manifest
- Continue with remaining items
- Skipped items auto-carry to next iteration's checklist

Update manifest: add `dr_reports` list, set `current_phase: "LINK"`.

### Phase 4: LINK

Read all `vN/dr/*.md` file paths. Update `vN/main.md`:
1. Insert inline references at relevant suspicious points: `[DR-{NNN}: {title}](dr/dr-{NNN}-{slug}.md)`
2. Append a DR index table at the bottom of `vN/main.md`

Update manifest: `current_phase: "REWRITE"`.

### Phase 5: REWRITE

Launch a fresh `codeagent` (backend: **codex**) with all documents as context:

```bash
codeagent-wrapper --backend codex - <project-root> <<'EOF'
You are a document rewrite specialist. Read the complete document set and rewrite the main document.

Main document: @docs/dr-workflow/<project>/vN/main.md
DR reports directory: @docs/dr-workflow/<project>/vN/dr/
Previous review (if exists): @docs/dr-workflow/<project>/v{N-1}/review.md

Requirements:
1. Integrate all DR findings — strengthen or correct original claims
2. Remove claims disproved by DR reports
3. Add evidence from validated DR reports
4. Maintain document structure integrity
5. Write output to: docs/dr-workflow/<project>/v{N+1}/main.md
EOF
```

Timeout: 7200000 (complex task). Create `v{N+1}/` directory and `v{N+1}/dr/` before invocation.

Update manifest: `current_version: N+1`, `current_phase: "REVIEW"`.

### Phase 6: REVIEW

Read `v{N+1}/main.md` plus all historical DR reports. Score using type-specific dimensions.

See `references/scoring.md` for dimension details per document type.

**Universal output format for `vN/review.md`:**
```markdown
# Review v{N}

## Score: {total}/100

| Dimension | Score | Notes |
|-----------|-------|-------|
| ... | .../25 | ... |

## New Suspicious Items
(Items that need DR in next iteration, or "None" if clean)

## Verdict
CONTINUE (score < 90 or new items > 0) | FINALIZE (score ≥ 90 and new items = 0)
```

**Branch logic:**
- Verdict = CONTINUE → new items become next `checklist.md`, back to Phase 2
- Verdict = FINALIZE → proceed to Phase 7
- `current_version > max_iterations` (default 5) → force FINALIZE with warning

**Checkpoint:** Display review summary to user after each iteration. Continue automatically.

### Phase 7: FINALIZE

1. Copy final `main.md` → `final/main.md`
2. Generate `final/milestones.md` based on document type:
   - `prd` → phase/milestone breakdown with acceptance criteria
   - `paper` → chapter-level alignment check
   - `tech-design` → module-level decomposition
   - `custom` → section-level summary
3. Run alignment check: verify milestones cover all content in final document
4. Update manifest: `current_phase: "DONE"`

## Resume Support

On invocation, check for existing `manifest.json` in `docs/dr-workflow/`:
- If found with `current_phase != "DONE"` → ask user: "Detected unfinished project '{name}' at Phase {X} v{N}. Resume?"
- If user confirms → continue from recorded phase
- If user declines → start fresh (new project name)

## Manifest Schema

```json
{
  "project": "string",
  "type": "prd|paper|tech-design|custom",
  "created": "ISO-8601",
  "current_phase": "DRAFT|ANALYZE|RESEARCH|LINK|REWRITE|REVIEW|FINALIZE|DONE",
  "current_version": 1,
  "termination": { "score_threshold": 90, "max_iterations": 5 },
  "iterations": [
    {
      "version": 1,
      "phase_completed": "REVIEW",
      "score": 72,
      "new_dr_items": 5,
      "dr_reports": ["dr/dr-001-auth.md"],
      "skipped_items": [],
      "timestamp": "ISO-8601"
    }
  ]
}
```

## Error Recovery

1. **DR generation failure** → mark item as skipped, auto-retry next iteration
2. **Codeagent timeout** → retry once; if still fails, save partial progress as `REWRITE_PARTIAL`
3. **Session interruption** → manifest preserves state; resume on next invocation
4. **Max iterations reached** → force finalize with warning listing unverified items
5. **Invalid DR report** (missing Executive Summary or Sources) → auto-regenerate once
