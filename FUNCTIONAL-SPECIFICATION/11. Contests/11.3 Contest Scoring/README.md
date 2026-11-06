# DEZOLVER — 11.3 Contest Scoring

**Document ID:** CON-SCORE-FS-001 | **Version:** 0.1 | **Status:** Detailed draft; decisions open  
**Governing standard:** [Functional Specification Standard v1.0](../../FUNCTIONAL-SPECIFICATION-STANDARD.md)  
**Parent module:** [Module 11 Contests](../README.md)  
**Core contest management:** [Function-Level Specifications](../FUNCTION-SPECIFICATIONS.md)

> Source facts, derived behavior, recommendations, and TBD decisions remain separate. This draft does not approve proposed behavior.

## Evidence
ContestProblem.pointsWeight decimal(6,2) and ContestScoreRank.totalScore decimal(7,2) are source schema fields. Business Flow says evaluation→contest scoring→score/rank. No scoring formula, per-problem score record, partial credit, score history, or group score is specified.

### CSCORE-F001 — Calculate Problem score
- **Purpose:** Convert Module 7 evaluation into Contest Problem score.
- **Trigger:** evaluation result for Contest-linked submission.
- **Inputs:** Contest, ContestProblem, Problem, submission, verdict, pointsWeight.
- **Preconditions:** contest context linkage and score formula approved.
- **Flow:** validate submission context → consume Module 7 result → apply approved rule → persist score component according to approved model → aggregate.
- **TBD:** pointsWeight multiplier/max/weight meaning; verdict mapping; partial credit; multiple attempts (best/latest/first); wrong answer; precision/rounding; cap. CON-D15.
- **Validation:** mismatched Contest, duplicate evaluation, incomplete evaluation, invalid weight.
- **Data:** no ContestProblemScore entity shown; storage method TBD.
- **Failure:** operational judge error not automatically zero; retry Module 18.
- **Acceptance:** deterministic approved formula; duplicates do not double count; evaluation error separate.
- **QA:** CON-QA-077–082.

### CSCORE-F002 — Aggregate total score
- **Trigger:** component changes/evaluation completion/recompute.
- **Inputs:** participant, Contest Problems, score components, weight rule.
- **Flow:** load eligible submissions → apply approved attempt selection → aggregate and round → update ContestScoreRank.totalScore → notify Ranking.
- **TBD:** unattempted problem treatment, maximum, partial credit, attempt selection, late submissions, score lock, uniqueness per participant. CON-D15/17/23.
- **Data:** update totalScore; schema uniqueness not shown.
- **Failure:** avoid duplicate/partial aggregation; retry/idempotency Module 18.
- **Acceptance:** reproducible score; client cannot set score; no cross-contest inputs.
- **QA:** CON-QA-083–087.

### CSCORE-F003 — Finalize/recompute score
- **Trigger:** Contest end or authorized recomputation.
- **Preconditions:** evaluation completion/finalization policy approved.
- **Flow:** reconcile pending submissions → apply approved wait/failure policy → aggregate → define completedAt → finalize score inputs → trigger ranking.
- **TBD:** provisional/final, rejudge, manual adjustment, completedAt meaning, reopen CON-D17/20.
- **Permissions:** system; staff override only if approved and audited.
- **Data:** ContestScoreRank totalScore/completedAt/timestamps; no history table shown.
- **Edge:** Judge backlog, rejudge after publish, no submissions, Contest reopened.
- **Acceptance:** finalization criteria explicit; unresolved evaluations handled by policy; manual adjustments traceable.
- **QA:** CON-QA-088–092.

**Decision IDs:** CON-D15/17/23/24.
