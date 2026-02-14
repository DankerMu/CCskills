# Review Scoring Dimensions

Type-specific scoring rubrics for Phase 6 (REVIEW). Total: 100 points.

## PRD

| Dimension | Weight | Criteria |
|-----------|--------|----------|
| Argument Completeness | 25 | Core claims have supporting evidence from DR reports |
| Technical Feasibility | 25 | Architecture/tech choices validated by DR; no unverified assumptions |
| Internal Consistency | 20 | No contradictions between sections; DR findings integrated coherently |
| Evidence Sufficiency | 15 | DR reports cover all critical decision points; no major gaps |
| Executability | 15 | Can be directly handed off to development; milestones are actionable |

## Paper (Academic / Research)

| Dimension | Weight | Criteria |
|-----------|--------|----------|
| Argument Completeness | 25 | Thesis and supporting arguments are logically complete |
| Methodological Feasibility | 25 | Research methods validated; data collection approach verified |
| Internal Consistency | 20 | Literature review, methods, and conclusions align |
| Evidence Sufficiency | 20 | Citations adequate; DR confirms key claims against existing literature |
| Academic Rigor | 10 | Proper structure, terminology, and scholarly conventions followed |

## Tech Design

| Dimension | Weight | Criteria |
|-----------|--------|----------|
| Argument Completeness | 25 | All design decisions documented with rationale |
| Technical Feasibility | 25 | Component interactions verified; performance claims validated by DR |
| Internal Consistency | 20 | API contracts, data flows, and component boundaries are coherent |
| Evidence Sufficiency | 15 | Benchmarks, capacity estimates, and trade-offs backed by DR data |
| Executability | 15 | Module decomposition clear enough for parallel implementation |

## Custom (General Purpose)

| Dimension | Weight | Criteria |
|-----------|--------|----------|
| Argument Completeness | 25 | Main thesis fully developed with supporting points |
| Feasibility | 25 | Proposals are realistic and validated where applicable |
| Internal Consistency | 20 | Document is self-coherent; no conflicting statements |
| Evidence Sufficiency | 20 | Key claims supported by DR research or credible references |
| Practical Value | 10 | Actionable insights or clear conclusions present |

## Scoring Guide

- **23-25 / 18-20 / 13-15 / 8-10**: Excellent — dimension fully satisfied
- **18-22 / 14-17 / 10-12 / 6-7**: Good — minor gaps, no blocking issues
- **12-17 / 10-13 / 7-9 / 4-5**: Needs work — specific DR items required
- **<12 / <10 / <7 / <4**: Critical gap — multiple DR items needed

## New Item Detection

During review, identify any new suspicious points NOT covered by existing DR reports.
Each new item must specify:
- Type tag (tech/product/market/architecture/data/methodology/feasibility)
- Brief description of what needs verification
- Suggested research direction
