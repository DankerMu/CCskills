# Document Templates

Templates applied during Phase 1 (DRAFT) based on `--type` parameter.
The brainstorming skill produces free-form content; structure it into these templates.

## PRD Template

```markdown
# {Project Name} — Product Requirements Document

**Version**: Draft v1
**Date**: {YYYY-MM-DD}
**Type**: PRD

---

## Executive Summary
{2-3 paragraphs: problem, solution, expected impact}

## Problem Statement
**Current situation**: ...
**Proposed solution**: ...
**Business impact**: ...

## Success Metrics
- KPI 1: {metric} — {target}
- KPI 2: {metric} — {target}

## User Personas
### Primary: {Name}
- Role / Goals / Pain points / Technical level

## User Stories & Acceptance Criteria
### Story 1: {Title}
**As a** {persona} **I want to** {action} **So that** {benefit}
- [ ] Acceptance criterion 1
- [ ] Acceptance criterion 2

## Functional Requirements
### Core Features
**Feature 1**: {description, user flow, edge cases, error handling}

### Out of Scope
- {explicitly excluded items}

## Technical Constraints
- Performance: ...
- Security: ...
- Integration: ...

## MVP & Phasing
### Phase 1: MVP
- {core features}
### Phase 2: Enhancements
- {enhancements}

## Risk Assessment
| Risk | Probability | Impact | Mitigation |
|------|------------|--------|------------|

## Dependencies & Blockers
```

## Paper Template

```markdown
# {Title}

**Version**: Draft v1
**Date**: {YYYY-MM-DD}
**Type**: Paper

---

## Abstract
{150-300 words summarizing research question, methods, results, conclusions}

## 1. Introduction
### 1.1 Background
### 1.2 Research Question
### 1.3 Contributions

## 2. Related Work
### 2.1 {Area 1}
### 2.2 {Area 2}

## 3. Methodology
### 3.1 Approach
### 3.2 Data / Materials
### 3.3 Experimental Setup

## 4. Results
### 4.1 {Result area 1}
### 4.2 {Result area 2}

## 5. Discussion
### 5.1 Interpretation
### 5.2 Limitations
### 5.3 Future Work

## 6. Conclusion

## References
```

## Tech Design Template

```markdown
# {Project Name} — Technical Design Document

**Version**: Draft v1
**Date**: {YYYY-MM-DD}
**Type**: Tech Design

---

## Overview
{1-2 paragraphs: what system does and why}

## Goals & Non-Goals
### Goals
### Non-Goals

## Architecture
### System Diagram
### Component Overview

## Detailed Design
### Component 1: {Name}
- Responsibility
- Interface / API
- Data model
- Key algorithms

### Component 2: {Name}
...

## Data Design
### Schema
### Migration strategy

## API Design
### Endpoints / Interfaces

## Infrastructure
### Deployment
### Scaling
### Monitoring

## Security
### Threat model
### Mitigations

## Testing Strategy
### Unit / Integration / E2E

## Migration & Rollout Plan
### Phases
### Rollback plan

## Open Questions
```

## Custom Template

```markdown
# {Title}

**Version**: Draft v1
**Date**: {YYYY-MM-DD}
**Type**: Custom

---

## Summary
{Core thesis or purpose of this document}

## Background & Context
{Why this document exists, what problem it addresses}

## Main Content
{Adapt sections to the specific document's needs}

## Analysis
{Key arguments, evidence, reasoning}

## Conclusions & Recommendations
{Actionable outcomes}

## References
```
