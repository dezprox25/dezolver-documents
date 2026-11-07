# DEZOLVER — 11.5 Contest Ranking

**Document ID:** CON-RANK-FS-001 | **Version:** 0.2 | **Status:** Detailed draft; decisions open  
**Governing standard:** [Functional Specification Standard v1.0](../../FUNCTIONAL-SPECIFICATION-STANDARD.md)  
**Parent module:** [Module 11 Contests](../README.md)  
**Core contest management:** [Function-Level Specifications](../FUNCTION-SPECIFICATIONS.md)

> Source facts, derived behavior, recommendations, and TBD decisions remain separate. This draft does not approve proposed behavior.

## Evidence
Page mapping includes Student Leaderboard using ContestScoreRank, Contest, User (v2 also GroupContest). RBAC Ranking: SUPER_ADMIN View/Manage (platform matrix), Finance View, HOD View/Manage, Manager View, Student View/own/permitted. Schema rank, totalScore, penaltyTime. Rank algorithm and visibility are not specified.

### CRANK-F001 — Calculate ranks
- **Purpose:** Derive ranks from authorized Contest score/penalty inputs.
- **Trigger:** score/penalty update, Contest finalization, ranking refresh.
- **Inputs:** participants, totalScore, penaltyTime, approved tie policy.
- **Preconditions:** scoring/penalty outputs valid; rank policy approved.
- **Flow:** fetch Contest participant scores → handle pending/incomplete rows by policy → sort using approved criteria → apply tie rule → assign ranks → persist → mark provisional/final only if model supports.
- **TBD:** score-first, penalty ordering, rank numbering (ordinal/competition/dense), incomplete participants, equal values, provisional/final, rejudge. CON-D18/19.
- **Data:** update ContestScoreRank.rank; unique score row constraint not explicit.
- **Failure:** stale/incomplete ranks not shown as final without approval.
- **Acceptance:** deterministic, reproducible, score/penalty changes trigger recompute, no cross-contest mixing.
- **QA:** CON-QA-103–108.

### CRANK-F002 — View leaderboard
- **Actors:** Student, HOD, Manager, SUPER_ADMIN; Finance according to RBAC.
- **Entry:** Contest leaderboard / Student Leaderboard.
- **Inputs:** contestId, filter/page; identity auth-derived.
- **Flow:** authorize Contest/leaderboard → apply provisional/final visibility policy → query ranking → filter scope/privacy → display permitted fields.
- **TBD:** Student own rank vs full leaderboard; names, penalty, score, release timing; Finance page absent; ranking matrix own/permitted nuance.
- **Validation:** foreign Contest, forged Student ID, unauthorized identity/private score.
- **Data:** read ContestScoreRank/Contest/User projection.
- **Acceptance:** scope and result release enforced; private data/code/testcases hidden.
- **QA:** CON-QA-109–113.

### CRANK-F003 — Publish/finalize standings
- **Trigger:** Contest end and approved evaluation/scoring/penalty/tie completion criteria.
- **Flow:** reconcile pending results → finalize score/penalty → rank/tie processing → persist standings → expose final status if supported → notify.
- **TBD:** schema has no publication/final flag; Contest ENDED does not guarantee results finalized; rejudge/late result effects CON-D17/18/22.
- **Acceptance:** publish only after explicit criteria; final status not shown prematurely; post-final changes auditable.
- **QA:** CON-QA-114–117.

### Historical cross-check — legacy Event evidence

**Evidence status: LEGACY-SPEC-REPORTED.** The old report describes Redis leaderboard:event updates and an inverse-rank defect from ascending zrank. Final grading reportedly sorted raw EventResult.score and assigned sequential ranks; equal scores could receive arbitrary order. A global leaderboard query reportedly referenced a nonexistent model. These are legacy findings, not target behavior. Target ContestScoreRank ordering, visibility and real-time transport remain CON-D18/D28; platform rating is separate and open under CON-D27/Module 16. Do not port zrank, old cron cadence, arbitrary ordering or the broken global query. Raw old source was not independently inspected.

**Decision IDs:** CON-D18/19/22–25.
