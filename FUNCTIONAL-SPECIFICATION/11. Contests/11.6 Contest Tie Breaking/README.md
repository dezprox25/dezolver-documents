# DEZOLVER — 11.6 Contest Tie Breaking

**Document ID:** CON-TIE-FS-001 | **Version:** 0.1 | **Status:** Detailed draft; decisions open  
**Governing standard:** [Functional Specification Standard v1.0](../../FUNCTIONAL-SPECIFICATION-STANDARD.md)  
**Parent module:** [Module 11 Contests](../README.md)  
**Core contest management:** [Function-Level Specifications](../FUNCTION-SPECIFICATIONS.md)

> Source facts, derived behavior, recommendations, and TBD decisions remain separate. This draft does not approve proposed behavior.

## Evidence
Schema includes totalScore, penaltyTime, rank, completedAt. Business Flow says score/rank. No tie-break algorithm, configuration, tie entity, manual adjudication, or shared-rank semantics exists in source. These rules are entirely unresolved.

### CTIE-F001 — Detect tied participants
- **Purpose:** Identify equal participants under approved rank criteria.
- **Trigger:** ranking calculation.
- **Inputs:** Contest participants, scores, penalties, timestamps, approved comparison rules.
- **Preconditions:** ranking and score precision rules approved.
- **Flow:** load same-Contest rows → compare primary score criterion → group equal values under approved equality/rounding → identify ties → pass to resolution.
- **TBD:** score precision, whether penalty breaks tie, completedAt meaning, equality rules. Do not use schema fields automatically as tie-breakers.
- **Validation:** same Contest only; data finalized/eligible; no duplicate participant score rows.
- **Data:** detection transient; no tie-group schema.
- **Acceptance:** reproducible; only approved criteria; no arbitrary sorting to break tie.
- **QA:** CON-QA-118–121.

### CTIE-F002 — Resolve ties and assign ranks
- **Purpose:** apply approved rule or retain shared rank.
- **Actors:** system; staff manual resolution only with explicit permission.
- **Inputs:** tie set, configured sequence of rules/approved adjudication.
- **Flow:** detect tie → apply approved ordered criteria → if still tied, use approved shared-rank or adjudication policy → persist rank → audit manual intervention.
- **Options only (not source requirements):** lower penalty, earlier accepted time, completion time, extra challenge, shared rank, manual review. No option is approved.
- **Permissions:** RBAC Ranking Manage does not explicitly grant tie adjudication authority.
- **Data:** update rank; tie-break history not in schema.
- **Edge:** exact score/penalty tie, missing timestamp, concurrent rank job, rejudge after resolution, Contest reopen, rank published.
- **Acceptance:** explicit deterministic policy; no rule means preserve unresolved/shared tie as approved; manual changes auditable; repeatable.
- **QA:** CON-QA-122–126.

**Decision IDs:** CON-D19, D17–18, D22–25.
