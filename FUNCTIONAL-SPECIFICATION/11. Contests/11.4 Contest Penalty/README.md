# DEZOLVER — 11.4 Contest Penalty

**Document ID:** CON-PEN-FS-001 | **Version:** 0.1 | **Status:** Detailed draft; decisions open  
**Governing standard:** [Functional Specification Standard v1.0](../../FUNCTIONAL-SPECIFICATION-STANDARD.md)  
**Parent module:** [Module 11 Contests](../README.md)  
**Core contest management:** [Function-Level Specifications](../FUNCTION-SPECIFICATIONS.md)

> Source facts, derived behavior, recommendations, and TBD decisions remain separate. This draft does not approve proposed behavior.

## Evidence
ContestScoreRank.penaltyTime integer is in schema. No penalty unit, formula, event history, or penalty configuration is source-confirmed. Do not assume standard contest penalty conventions.

### CPEN-F001 — Calculate penalty
- **Purpose:** Compute participant penalty under an approved Contest-specific policy.
- **Trigger:** evaluated submissions, problem completion, finalization; exact trigger TBD.
- **Inputs:** Contest, participant, submissions/verdicts, timestamps, accepted completion data.
- **Preconditions:** penalty formula/unit approved; evaluation records valid.
- **Flow:** read authorized Contest submission/evaluation history → identify penalty events → compute penalty under approved policy → aggregate → update ContestScoreRank.penaltyTime → notify Ranking.
- **TBD:** seconds/minutes; wrong answer/compile/runtime treatment; time-from-start vs joinedAt; accepted-time; penalty per problem or contest; caps, rounding, unsolved penalties, late submissions, rejudge. CON-D16.
- **Validation:** wrong Contest/participant, duplicate event, invalid/missing timestamp, incomplete Judge result.
- **Data:** update penaltyTime only under approved design; no history entity shown.
- **Failure:** missing operational data not treated as penalty by guess; retry Module 18.
- **Acceptance:** formula/unit explicit, reproducible, duplicate-safe.
- **QA:** CON-QA-093–098.

### CPEN-F002 — Recompute/reconcile penalty
- **Trigger:** rejudge, evaluation correction, score recomputation, finalization.
- **Flow:** load canonical event history → recompute penalty → update field → trigger rank refresh.
- **Rules:** event ordering, rank published, manual adjustment, rejudge policy CON-D16/17. No penalty UI confirmed.
- **Validation:** participant/Contest linkage, idempotent processing.
- **Data:** penaltyTime and updatedAt.
- **Failure/edge:** out-of-order Judge result, duplicate job, reopened Contest, missing historical submission.
- **Acceptance:** identical source data gives same penalty; no double penalty; rank recalculated after change; manual adjustment auditable if allowed.
- **QA:** CON-QA-099–102.

**Decision IDs:** CON-D16–18/23.
