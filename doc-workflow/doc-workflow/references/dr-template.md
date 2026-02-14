# DR Report Template

Every DR report generated in Phase 3 (RESEARCH) must follow this structure.
The deep-research skill produces reports in its own format; post-process to match this template.

## Required Structure

```markdown
# DR-{NNN}: {Title}

**Checklist Item:** #{item_number} — {original suspicious point}
**Type:** {type_tag}
**Date:** {YYYY-MM-DD}
**Status:** verified | refuted | inconclusive | partially-verified

## Executive Summary

2-3 sentences: what was investigated, what was found, and the verdict.

## Key Findings

- **Finding 1**: Brief explanation [1]
- **Finding 2**: Brief explanation [2]
- **Finding 3**: Brief explanation [3]

## Detailed Analysis

### {Subtopic 1}
In-depth analysis with inline citations [N].

### {Subtopic 2}
In-depth analysis with inline citations [N].

## Impact on Main Document

Specific recommendations for how the main document should be updated:
- [ ] Section X: {change description}
- [ ] Section Y: {change description}

## Confidence Assessment

- **Overall confidence**: High / Medium / Low
- **Data quality**: {assessment}
- **Source diversity**: {number of independent sources}

## Sources

[1] {Full citation with URL if available}
[2] {Full citation with URL if available}

## Gaps

Items that could not be fully resolved and may need follow-up:
- {Gap description}
```

## Status Definitions

- **verified**: The claim in the main document is confirmed by multiple credible sources
- **refuted**: The claim is contradicted by evidence; main document must be corrected
- **inconclusive**: Insufficient evidence to confirm or deny; flagged for next iteration
- **partially-verified**: Some aspects confirmed, others need further investigation

## Quality Gate

A DR report is considered **invalid** and must be regenerated if:
1. Missing Executive Summary section
2. Missing Sources section or zero sources cited
3. Missing Impact on Main Document section
4. Status field is empty
