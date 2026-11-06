# Module 16 — Student Analytics: Function-Level Specifications

**Document ID:** STUDENT-ANALYTICS-FUNCTIONS-001 | **Version:** 0.1 | **Status:** Draft; policy-dependent  
**Standard:** [Functional Specification Standard](../FUNCTIONAL-SPECIFICATION-STANDARD.md)  
**Parent:** [Overview](./README.md) · [QA cases](./QA-TEST-CASES.md)

> Legacy behaviors are attributed to the prior Module 16 forensic reconstruction and are not target requirements unless approved. Unknown metrics, permissions, and business rules remain decisions.

## Shared authorization and privacy invariants
- Personal /me access derives identity from trusted principal, never client user IDs.
- Backend authorization, tenant/department/section/group scope, and ownership checks are mandatory.
- Scoped analytics apply server-derived scope before aggregation, filtering, pagination, and export.
- Aggregate results can still expose individual data; privacy review is required.
- Analytics are derived data. Freshness and source-of-truth definitions must be explicit.
- Legacy behavior does not automatically become target behavior.

## ANA-F001 — View personal learning paths and progress
**Status:** Legacy-reported; target semantics TBD. **Actor:** Student.
**Purpose:** Show paths associated with the learner and their aggregate progress.
**Entry:** My Progress page; legacy report references GET /me/paths.
**Preconditions:** Authenticated student; progress/path records available.
**Inputs:** Authenticated principal ID and tenant context; filters/pagination TBD.
**Candidate flow:** (1) Student opens page. (2) UI requests personal paths. (3) Backend derives identity/scope. (4) Backend queries authorized progress/path records. (5) UI renders list, loading, empty, or error state.
**Rules:** Never trust client-supplied user ID. Inclusion/order (active/completed/archived), freshness, and pagination TBD.
**Failures:** Authentication failure; empty state; service failure must not be presented as current data.
**Acceptance:** Student A cannot fetch Student B’s progress; list follows approved inclusion/order; loading/empty/error are distinct.
**QA:** ANA-QA-001–006. **Decisions:** ANA-D01, D02, D07, D21.

## ANA-F002 — View progress detail for a path
**Status:** Legacy-reported; API contract TBD. **Actor:** Student.
**Purpose:** Display progress for one authorized path.
**Entry:** Select path; legacy report references GET /me/paths/:id/progress.
**Inputs:** Path ID; user and tenant from auth context.
**Flow:** Validate path/access → query progress scoped to user/tenant → return count, percentage, activity/completion timestamps as approved → render.
**Rules:** No cross-user progress. Inclusion, rounding, denominator, and missing-progress behavior TBD.
**Failures:** Inaccessible path safe denial/not-found; missing progress follows initialization policy; stale aggregation follows freshness policy.
**Acceptance:** Scope enforced; percentage/counts match approved metric definition; missing data handled consistently.
**QA:** ANA-QA-007–012. **Decisions:** ANA-D01, D04, D07, D20.

## ANA-F003 — Resolve next learning step
**Status:** Legacy-reported; completion semantics TBD. **Actor:** Student.
**Purpose:** Resume at next eligible path step.
**Entry:** Continue/Resume; legacy report references GET /me/paths/:id/next-step.
**Inputs:** Path ID, principal, canonical path order and progress.
**Flow:** Validate access → load path/progress → resolve next eligible step → return descriptor or terminal state → UI navigates.
**Rules:** Legacy report says stepsCompleted index is used; this can drift with optional/skipped/reordered steps. Do not assume target logic. Prerequisites/locked content TBD.
**Failures:** Completed path/no next step; deleted/hidden step; invalid content reference; unauthorized path.
**Acceptance:** Resolver matches approved state, handles ordering and skipped/optional steps, distinguishes terminal from failure.
**QA:** ANA-QA-013–018. **Decisions:** ANA-D04, D06, D07, D20.

## ANA-F004 — Start, skip, or complete a room
**Status:** Legacy-reported; academic completion policy TBD. **Actor:** Student.
**Entry:** Room Start/Skip/Complete actions; prior report describes POST /me/rooms/:roomId/{start,skip,complete}.
**Inputs:** Room ID from route; principal/tenant from server; action; server timestamp.
**Preconditions:** Authenticated student; room belongs to tenant and authorized/enrolled path (reported legacy check).
**Flow:** Authenticate → validate room/path membership and action → validate current state → persist transition → emit progress event under approved consistency strategy → return state → UI updates.
**Rules:** Legacy states: in_progress, skipped, completed. Repeated actions/transitions require Module 17 rules. Skip must not bypass certificate eligibility unless expressly approved (Modules 12/17).
**Failures:** Unauthorized room, invalid transition, duplicate request, persistence/event failure; no false success.
**Acceptance:** Scope/ownership enforced; correct state/time stored; transitions are idempotent as approved; skip cannot silently bypass credential requirements.
**QA:** ANA-QA-019–028. **Decisions:** ANA-D04, D05, D07, Module 17/18.

## ANA-F005 — Aggregate room progress into path progress
**Status:** Legacy-reported; consistency policy TBD. **Actor:** Background consumer.
**Trigger:** Prior report describes RoomProgressUpdated → ProgressAggregatorConsumer.
**Inputs:** Event identity/version, user/tenant/path/room, path steps, room states.
**Flow:** Persist mutation + event per approved outbox strategy → consume event → validate scope → load canonical steps/states → calculate aggregate → upsert progress → emit completion only if approved → record processing status.
**Rules:** Reported formula completed/total × 100. Definitions of complete, optional, skipped, non-room, zero-step and completion must be approved. Skip should not count as academic completion by default.
**Failures:** Duplicate/out-of-order event, DB failure, retry exhaustion, path edit, zero steps, missing room, partial commit.
**Acceptance:** Replay safe; aggregation reconciles with canonical states; lag observable; no skip-only certificate eligibility.
**QA:** ANA-QA-029–039. **Decisions:** ANA-D04–D07, D14, D22, Module 18.

## ANA-F006 — Display student dashboard KPIs and activity
**Status:** Prior report says mocked dashboard; target metrics TBD. **Actor:** Student; other roles only if approved.
**Purpose:** Present concise learning/activity summary.
**Candidate metrics:** Active paths, completion, solved problems, upcoming events, certificates, recent activity. Not approved until definitions are set.
**Flow:** Dashboard loads → authorized API request → server scopes and aggregates approved source records → returns metric values/freshness → UI renders.
**Rules:** Prior report says synthetic mock data with artificial delay; verify current source. Metric formulas, period, timezone, refresh, and empty states are ANA-D08/D20/D21.
**Acceptance:** Metrics backed by approved data; scoped/privacy-safe; loading/zero/unavailable/stale states distinct; mocks never appear as real values.
**QA:** ANA-QA-040–046. **Decisions:** ANA-D02, D08, D15, D16, D21.

## ANA-F007 — View assessment analytics
**Status:** Legacy report says frontend-derived; target metrics TBD. **Actors:** Student self-view; authorized staff/cohort viewers.
**Inputs:** Assessment attempts, scores, grades, assessment ID, time period, cohort scope.
**Flow:** Open analytics → authorize assessment/scope → retrieve canonical outcomes → aggregate using approved definitions → render metrics and allowed drill-downs.
**Rules:** Mean/median/pass rate/attempts/percentile/distribution are candidate metrics only. Module 9 owns assessment grading/attempt rules. Cohort privacy threshold TBD.
**Acceptance:** Metrics agree with canonical grades; scope enforced; formulas/attempt filtering documented; frontend does not redefine metrics.
**QA:** ANA-QA-047–053. **Decisions:** ANA-D02, D09, D11, D16, D20.

## ANA-F008 — View event/contest analytics
**Status:** Legacy report says client-side aggregates; target metrics TBD. **Actors:** Event organizer/staff/student per permission.
**Inputs:** Event ID, registrations, participation, scores/ranks, completion status.
**Flow:** Open event analytics → authorize event/scope → retrieve canonical event records → aggregate → render summary and permitted details.
**Rules:** Module 11 owns event lifecycle. Definitions for registered/started/submitted/completed/disqualified/withdrawn, ties, cancellation, and late submissions must align.
**Acceptance:** Metrics match final event state; tenant boundaries enforced; cancelled/withdrawn/disqualified treatment is explicit.
**QA:** ANA-QA-054–060. **Decisions:** ANA-D02, D10, D12, D16, D20.

## ANA-F009 — View platform analytics and growth
**Status:** Prior report says client-side reductions over raw lists; target API TBD. **Actor:** SUPER_ADMIN unless approved otherwise.
**Candidate metrics:** user/tenant growth, active users, content/problem activity, event participation. Metric definitions TBD.
**Flow:** Admin opens analytics → global permission checked → backend aggregates approved sources/time period → returns metrics/trends → UI renders.
**Rules:** Global metrics restricted to authorized role; raw global data downloads not presumed acceptable; timezone, active definition, bucket, and backfill TBD.
**Acceptance:** Global access enforced; metric formulas and date buckets documented; response limits defined; no unauthorized PII.
**QA:** ANA-QA-061–067. **Decisions:** ANA-D02, D08, D15, D16, D20.

## ANA-F010 — View faculty/cohort progress analytics
**Status:** Prior report says cohort backend absent; role mapping TBD. **Actors:** HOD, MANAGER, faculty/instructor if approved.
**Preconditions:** Staff identity, assigned scope, cohort definition, and analytics permission established.
**Flow:** Staff selects cohort/range → backend checks role/scope → aggregates authorized learners → UI displays overview/distribution and permitted drill-downs.
**Rules:** RBAC has HOD college+department and Manager college+department+section; separate faculty role is not confirmed. Individual learner drill-down vs aggregate-only and small-cohort masking TBD.
**Acceptance:** HOD/Manager cannot cross assigned scope; student-level data only under explicit permission; cohort membership reproducible.
**QA:** ANA-QA-068–075. **Decisions:** ANA-D02, D03, D11, D16.

## ANA-F011 — View global leaderboard
**Status:** Prior report says defective query returns empty; intended policy TBD. **Actors:** Public/authenticated users or staff as approved.
**Inputs:** Limit/cursor, rating scope, optional filters.
**Flow:** Open leaderboard → determine visibility/scope → retrieve eligible rating records → sort/tie rank → paginate → display privacy-approved fields.
**Rules:** Legacy report says query references nonexistent Prisma model. Global/tenant scope, eligibility, ties, privacy and pagination are undecided.
**Acceptance:** Stable ranking/tie behavior; no unauthorized participant data; empty state differs from error; only approved fields.
**QA:** ANA-QA-076–082. **Decisions:** ANA-D12, D13, D16.

## ANA-F012 — View individual competitive standing
**Status:** Prior report says broken query returns 404; target contract TBD. **Actors:** Student (self); staff/public only if approved.
**Inputs:** Person ID for permitted staff query; self identity derived from principal.
**Flow:** Open My Standing → derive/check identity → retrieve rating/rank in approved scope → return standing or no-rating state → render.
**Rules:** No-rating vs nonexistent vs forbidden must not be conflated. Rank must use same eligibility/scope as leaderboard. Public visibility TBD.
**Acceptance:** Student cannot query another person by changing ID; rank aligns with leaderboard; no-rating state distinct.
**QA:** ANA-QA-083–088. **Decisions:** ANA-D02, D12, D13, D16.

## ANA-F013 — Calculate and update platform rating
**Status:** Prior report says ELO consumer exists; values are legacy only. **Actor:** System consumer after event completion.
**Inputs:** Event completion/version, final ranks/scores, eligible participants, current ratings.
**Flow:** Event finalizes (Module 11) → validate eligibility/finality → load prior ratings → compute under approved algorithm → persist → update/invalidate leaderboard cache → record event/version.
**Legacy values reported:** initial rating 1200, floor 100, K=32, pairwise ELO. Not target-approved.
**Rules:** Tie handling, event scope, floor/K, rating version, duplicate completion, corrections/cancellation, cache rebuild are TBD.
**Acceptance:** Final results only; replay does not double-apply; cache converges; corrected results follow policy; algorithm version traceable.
**QA:** ANA-QA-089–096. **Decisions:** ANA-D13, D14, D19, Modules 11/18.

## ANA-F014 — Export, protect, and retain analytics
**Status:** Not source-confirmed; decision required. **Actors:** Authorized staff/admin/system retention worker.
**Candidate:** CSV/XLSX export, aggregate or row-level, async job, limits, expiry, retention, redaction.
**Flow:** Request export → check permission/scope/filters/size → generate approved output → secure delivery → expire artifact → audit export where required.
**Rules:** Do not enable until policy approved. Educational performance and PII are sensitive; minimize and redact. Export access and retention are Module 15 dependencies.
**Acceptance:** Unauthorized export denied; output scope is server-enforced; sensitive fields masked; limits/expiry enforced; export auditable per policy.
**QA:** ANA-QA-097–104. **Decisions:** ANA-D02, D16–D18.

## Function coverage
ANA-F001–F014 drafted. Target analytics metrics, completion/certificate semantics, leaderboard/rating rules, privacy, export, and retention remain blocked by decision log ANA-D01–D22.
