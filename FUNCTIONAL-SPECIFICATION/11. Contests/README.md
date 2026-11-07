# DEZOLVER — Module 11: Contests

**Document ID:** DEZ-CON-FS-001  
**Version:** 0.2 — legacy Event cross-check and cross-module reconciliation  
**Status:** Detailed draft aligned to Functional Specification Standard v1.0; product, security, and architecture decisions remain open  
**Approval status:** Not approved for implementation  
**Governing standard:** [Functional Specification Standard v1.0](../FUNCTIONAL-SPECIFICATION-STANDARD.md)  
**Function-level specifications:** [Contest Management Functions](./FUNCTION-SPECIFICATIONS.md)  
**QA companion:** [Contest QA Test Cases](./QA-TEST-CASES.md)

> This module covers Contest management and the six index-defined submodules. The source documents confirm conceptual entities, high-level workflows, role permissions, and page mappings, but do not define all operational rules. Recommendations and decision gaps remain clearly marked. Committing this document versions the draft; it does not approve target product behavior.

## 1. Purpose and outcome

DEZOLVER Contests support a coding activity flow in which a contest is associated with contest problems, students participate, submit code, receive evaluation, and contribute to contest score/rank.

Business Flow v2 describes the following conceptual flow:

Group → Contest → Contest Problem → Student → Participation → Code Submission → Judge0 → Contest Scoring → Score/Rank.

The Student flow also depicts Contest → Contest Problem → Submit → Code Submission → Judge0 → Contest Score → Score → Rank.

This module specifies the available source-supported scope and identifies what must be decided before implementation.

## 2. Scope

### 2.1 Included
- Contest list and details
- Create/edit Contest
- Contest problem association, order, and pointsWeight
- Contest status/active lifecycle
- Group-to-Contest association context
- Student Contest discovery and participation
- Contest-specific submission linkage and evaluation handoff
- Contest scoring and score aggregation
- Penalty calculation and storage
- Contest ranking and leaderboard
- Tie-breaking and rank finalization
- Access control, privacy, failure handling, audit/notification dependencies

### 2.2 Excluded / owned elsewhere
- Problem content, test case authoring, testcase secrecy: Module 5
- Code submission processing and Judge0 integration: Module 6
- Verdict/testcase evaluation: Module 7
- Group creation and membership lifecycle: Module 10
- Notification infrastructure and delivery: Module 13
- Audit event catalog and retention: Module 15
- Student analytics/reporting: Module 16
- Canonical state machines: Module 17
- Shared retry, timeout, idempotency, concurrency and failure behavior: Module 18
- Certificate issuance for Contest outcomes: Module 12
- Payment/subscription/seat enforcement: Module 3; Contest-specific eligibility is not stated in sources

## 3. Source and evidence baseline

| Source | Source-supported content | Limits |
|---|---|---|
| [Business Flow v2](../../docs/source-text/Business%20Flow%20v2.md) | Groups mediate collaborative Labs and Contests; Contest Problems, Student participation, code submission, Judge0, evaluation, scoring, score/rank are depicted. | Does not specify registration, team-vs-individual behavior, scoring formula, penalty, ranking, tie-break, contest windows, or submission limits. |
| [Business Flow original](../../docs/source-text/Business%20Flow.md) | Contest and Student coding flow; Contest Problems and participant/score concepts. | Less detail than v2; no scoring/penalty/tie rules. |
| [Page-to-DB Mapping v2](../../docs/source-text/Page%20-%20DB%20Mapping%20v2.md) | HOD, Manager, Student Contests pages; Leaderboard; Contest, ContestProblem, ContestParticipant, ContestScoreRank, Group, GroupMember, GroupContest, Problem, CodeSubmission. | Page/entity map, not screen behavior or permissions for every operation. |
| [Page-to-DB Mapping original](../../docs/source-text/Page%20-%20DB%20Mapping.md) | HOD/Manager/Student Contests pages; Student Leaderboard; related Contest entities, Problem, Group. | No complete interaction details. |
| [RBAC](../../docs/source-text/RBAC.md) | SUPER_ADMIN global; Finance assigned College; HOD College+Department; Manager College+Department+Section/permitted Group; Student own/permitted academic data. Contest role matrix and Contest Ranking role matrix. | Role scope and Contest-specific direct relationships do not fully align with schema. |
| [Conceptual Schema](../../docs/source-text/scheema.md) | Contest, ContestProblem, ContestParticipant, ContestScoreRank, GroupContest; ContestStatus enum. | Conceptual schema only; not proof of deployed constraints or full target data model. |
| [Functional Specification Standard](../FUNCTIONAL-SPECIFICATION-STANDARD.md) | Function-level detail, evidence labels, acceptance criteria, QA, decision log, readiness checklist. |
| [Historical Module 11 forensic specification](../../functional-spec-from-old%20code/module-11-contests/SPECIFICATION.md) and [legacy summary](../../functional-spec-from-old%20code/module-11-contests/README.md) | Legacy Event model, routes, state watcher, registration/payment flow, Assessment-proxy submission, scoring/Redis leaderboard, reported defects, ELO and certificate side effects. | Governs authoring; does not itself decide Contest behavior. |

### Evidence labels
- **Source-confirmed:** explicitly present in the supplied source documents.
- **Derived:** logical consequence of source facts, not stated as a complete rule.
- **Recommendation:** suggested target behavior, pending product-owner approval.
- **TBD / Decision Required:** absent, ambiguous, or conflicting source evidence.
- **LEGACY-SPEC-REPORTED:** behavior reported by the historical Module 11 forensic specification. This review inspected that report, not raw legacy application files; findings are not claimed as independently reverified.
- **Legacy-verified:** not claimed here. Raw legacy application source was not independently retrieved during this pass.

### 3.1 Historical Module 11 forensic cross-check — legacy Event is not target Contest

**Evidence status:** LEGACY-SPEC-REPORTED. The historical Module 11 artifact labels its reconstruction `COMPLETED & FORENSICALLY VERIFIED` and cites `src/events/`, `prisma/events/`, billing and frontend files. This review inspected the report artifact only; raw legacy application source files were not independently retrieved.

The historical implementation is an **Event** subsystem. The current target documentation defines a **Contest** entity and separate participation, submission, scoring, penalty, ranking and tie-breaking areas. These concepts overlap, but the schemas, lifecycle, API contracts and side effects are not interchangeable.

| Area | Legacy Event behavior reported | Current target Contest evidence / interpretation |
|---|---|---|
| Main entity | `Event`: tenantId, kind, title, description, startsAt, endsAt, maxParticipants, visibility, paid, registrationFeePaise, string status, createdByUserId, timestamps. Kinds included competition, workshop, hackathon, webinar. | Target `Contest`: id, collegeId, createdById, title, description, startsAt, endsAt, ContestStatus, isActive, timestamps. Do not import legacy kind, visibility, capacity, paid-fee or tenant fields. |
| Registration | `Registration`: eventId, userId, tenantId, optional teamName, status, registeredAt, cancelledAt; reported unique (eventId,userId). | Target `ContestParticipant`: contestId, studentId, joinedAt, timestamps. Capacity, withdrawal, payment, team semantics and uniqueness remain open. |
| Event routes | GET/POST `/events`, GET `/events/:id`, PATCH `/events/:id`, publish, cancel, extend; PATCH reported draft-only. | Target CON-F001–F006 define Contest CRUD, Problem configuration and lifecycle. Legacy routes/permission strings are not target API contracts. |
| Problem association | No legacy EventProblem/ContestProblem table or Event config JSON column reported, despite frontend EventConfig/CompetitionProblem types. | Target conceptual schema explicitly defines `ContestProblem(contestId, problemId, pointsWeight, orderIndex, createdAt)`. Version pinning remains CON-D11 / Module 5. |
| Participation | Legacy registration guard reportedly accepted only `published` while watcher changed to `registration_open`, causing a lockout. Capacity used maxParticipants; audience/cohort scope reportedly not validated. | Target registration windows, capacity, eligibility, boundaries and scope must follow target ContestStatus/RBAC; do not reproduce the reported defects. |
| Submission | `POST /events/:id/submissions` was a stub; actual code used Assessment submission with `metadata.eventId`. | Module 6 now defines target `contextType=CONTEST`, `contextId=Contest.id`, and CodeSubmission → SubmissionRun → JudgeRun. Do not route target Contest submissions through Assessment. |
| Scoring | Legacy consumer accepted `accepted`/`partial`, upserted EventResult.score and updated Redis. | Module 7's adopted evaluation result is the evaluation input. Target pointsWeight meaning, per-Problem score model, attempt selection and totalScore aggregation remain open under CON-D15/17/23. |
| Penalty | No ICPC penalty calculation, wrong-attempt counter or cumulative solve-time tracker reported. | Target `ContestScoreRank.penaltyTime` has no unit/formula. Do not assume a 20-minute penalty; CON-D16 remains open. |
| Ranking/ties | Redis `leaderboard:event:${eventId}`; ascending `zrank` defect; final grading sorted score descending; equal scores could receive arbitrary order. | Target ordering, provisional/final visibility and tie treatment remain open under CON-D18/19. Do not copy `zrank` or arbitrary row-order tie results. |
| Other side effects | Legacy EventCompleted reportedly applied ELO K=32/initial 1200/floor 100; top-three certificate eligibility; global leaderboard query defect; 24h/1h reminders; cancellation refunds. | Not target Contest defaults. Rating Module 16; certificates Module 12; notifications Module 13; payments Module 3; lifecycle/recovery Modules 17/18. Open target questions CON-D22/D26–D28. |

**Historical section numbering differs from target submodule names.**

| Historical report section | Historical topic | Current target destination |
|---|---|---|
| Legacy 11.1 | Event creation/scheduling/state | CON-F003/004/006; Module 17 owns transitions |
| Legacy 11.2 | Event Problem config/submission indirection | CON-F005; CSUB-F001/F002; Module 5 owns Problem versioning |
| Legacy 11.3 | Registration/eligibility | CPART-F001–F004 |
| Legacy 11.4 | Contest window/Judge0/realtime state | CON-F006; CSUB-F001–003; CON-D28 |
| Legacy 11.5 | Score, absent ICPC penalty, Redis leaderboard | CSCORE-F001–003; CPEN-F001/002; CRANK-F001/002 |
| Legacy 11.6 | Final grading/ranking/ELO/certificates | CSCORE-F003; CRANK; CTIE; Modules 12 and 16 |

**Do not transfer these as target requirements:** Event-specific states/cadence; legacy registration guard; Event/Registration/EventResult schema; Assessment proxy; accepted/partial score handling; Redis `zrank`; missing ICPC penalty; arbitrary tie ordering; ELO constants; top-three certificates; Razorpay event fees/refunds; 24h/1h reminders; or legacy WebSocket behavior.
## 4. Terminology

| Term | Source-grounded meaning / status |
|---|---|
| Contest | A coding activity entity with college, creator, title, description, start/end timestamps, ContestStatus, and isActive. |
| Contest Problem | A Problem associated with a Contest through ContestProblem; has pointsWeight and orderIndex. |
| Participant | Student associated with Contest through ContestParticipant. |
| Group Contest association | GroupContest links Group and Contest. The source does not establish team submission/scoring semantics. |
| Contest submission | Code submission made in the Contest context. The conceptual CodeSubmission entity has problemId/studentId but no contestId or contestProblemId. Context linkage is unresolved. |
| Contest score | ContestScoreRank.totalScore; calculation formula is not specified. |
| Penalty | ContestScoreRank.penaltyTime; unit and calculation formula are not specified. |
| Rank | ContestScoreRank.rank; ordering and tie behavior are not specified. |
| completedAt | ContestScoreRank field; the meaning and update event are not specified. |

## 5. Actors, permissions, and organizational scope

### 5.1 Role scope from RBAC
| Role | Scope |
|---|---|
| SUPER_ADMIN | Global |
| FINANCE_TEAM | Assigned College |
| HOD | Assigned College + Department |
| MANAGER | Assigned College + Department + assigned Section / permitted Group |
| STUDENT | Own records + permitted academic resources |

### 5.2 Contest permissions from source matrices
| Role | Contest management / page matrix | Contest API matrix | Contest Ranking |
|---|---|---|---|
| SUPER_ADMIN | Full CRUD | Full CRUD | View/Manage (platform matrix); V in page matrix |
| FINANCE_TEAM | View | View* | View |
| HOD | Full CRUD | Full CRUD* | View/Manage* |
| MANAGER | Create/Update* (platform matrix) | V/C/U* | View |
| STUDENT | View/Participate | View/Participate | View; own/permitted ranking |

**RBAC conflict/interpretation note:** Platform-level RBAC gives Manager Contest Create/Update*; Page/API matrices show V/C/U*. Delete is not explicit in Manager page/API permissions. Treat Manager Delete as denied/not granted unless later approved. Do not expand permission by inference.

**Backend enforcement:** Every Contest operation must validate authenticated identity, action permission, College/Department/Section/Group scope, membership/eligibility, and ownership where applicable. Frontend visibility is not the security boundary. Do not trust client-supplied role, collegeId, departmentId, sectionId, studentId, or Group membership.

### 5.3 Scope mismatch requiring resolution
- RBAC HOD scope includes College + Department, but Contest schema has only collegeId and createdById; no departmentId.
- Manager scope is Section/Group-based, but Contest schema has no sectionId or manager assignment.
- GroupContest connects a Group to a Contest, and Group has departmentId, but this relation does not by itself define every Contest’s visibility or authorization.
- Finance Contest View is in the RBAC matrices, but Page-to-DB mapping does not show a Finance Contest page and Finance is primarily subscription/billing-focused.

Do not guess the missing relationship or use creator identity as a substitute for explicit scope.

## 6. Module and submodule structure

| Module | Scope | Detailed specification |
|---|---|---|
| 11 — Contest Management | Contest list/detail, create/edit, Contest Problems, status/active lifecycle, Group association | [Contest Management Functions](./FUNCTION-SPECIFICATIONS.md) |
| 11.1 — Contest Participation | Student discovery, eligibility, registration/participation, participant records, Group participation context | [Participation](./11.1%20Contest%20Participation/README.md) |
| 11.2 — Contest Submission | Problem access in Contest context, submission handoff, evaluation result visibility | [Submission](./11.2%20Contest%20Submission/README.md) |
| 11.3 — Contest Scoring | Problem score, weighted Contest score, total-score aggregation/finalization | [Scoring](./11.3%20Contest%20Scoring/README.md) |
| 11.4 — Contest Penalty | Penalty input, calculation, storage and recomputation | [Penalty](./11.4%20Contest%20Penalty/README.md) |
| 11.5 — Contest Ranking | Rank calculation, standings, leaderboard visibility | [Ranking](./11.5%20Contest%20Ranking/README.md) |
| 11.6 — Contest Tie Breaking | Tie detection, tie-break application, final rank handling | [Tie Breaking](./11.6%20Contest%20Tie Breaking/README.md) |

**Historical numbering note:** the old-code report's six sections use a different decomposition from the six current target subfolders. Current target folder names/function IDs are authoritative; the historical mapping is traceability only.

## 7. Functional inventory

| Function ID | Function | Actor | Type | Specification |
|---|---|---|---|---|
| CON-F001 | List and filter Contests | Staff/Student | Read | Function specs |
| CON-F002 | View Contest details | Staff/Student | Read | Function specs |
| CON-F003 | Create Contest | SUPER_ADMIN/HOD/conditional Manager | Create | Function specs |
| CON-F004 | Update Contest metadata and schedule | SUPER_ADMIN/HOD/conditional Manager | Update | Function specs |
| CON-F005 | Add/remove/reorder Contest Problems and weights | Authorized staff | Association/workflow | Function specs |
| CON-F006 | Change Contest lifecycle/active state | Authorized staff/system | Lifecycle | Function specs |
| CON-F007 | Associate Group with Contest | Authorized staff | Association | Function specs |
| CPART-F001–F004 | Discover, validate, join, and inspect participants | Student/staff | Workflow/read | Participation |
| CSUB-F001–F003 | View Contest Problem, submit code, view result | Student/system | Workflow/integration | Submission |
| CSCORE-F001–F003 | Score problem, aggregate total, finalize score | System/staff | Calculation/workflow | Scoring |
| CPEN-F001–F002 | Calculate and recompute penalty | System/staff | Calculation | Penalty |
| CRANK-F001–F003 | Calculate rank, view leaderboard, publish standings | System/staff/student | Calculation/read | Ranking |
| CTIE-F001–F002 | Detect ties and apply tie policy | System/staff | Calculation/decision | Tie Breaking |

## 8. Source-confirmed conceptual data reference

### 8.1 Contest
| Field | Type | Source status / notes |
|---|---|---|
| id | varchar | Schema |
| collegeId | varchar | Schema; scope source of truth unresolved |
| createdById | varchar | Schema |
| title | varchar(255) | Schema |
| description | text | Schema |
| startsAt | timestamp | Schema |
| endsAt | timestamp | Schema |
| status | ContestStatus | Enum: UPCOMING, LIVE, ENDED |
| isActive | boolean | Separate field; meaning/precedence vs status TBD |
| createdAt / updatedAt | timestamp | Schema |

### 8.2 ContestProblem
| Field | Type | Source status / notes |
|---|---|---|
| id | varchar | Schema |
| contestId | varchar | Schema |
| problemId | varchar | Schema |
| pointsWeight | decimal(6,2) | Schema; scoring interpretation TBD |
| orderIndex | int | Schema; ordering field |
| createdAt | timestamp | Schema |

### 8.3 ContestParticipant
| Field | Type | Source status / notes |
|---|---|---|
| id | varchar | Schema |
| contestId | varchar | Schema |
| studentId | varchar | Schema |
| joinedAt | timestamp | Schema |
| createdAt / updatedAt | timestamp | Schema |

### 8.4 ContestScoreRank
| Field | Type | Source status / notes |
|---|---|---|
| id | varchar | Schema |
| contestId | varchar | Schema |
| studentId | varchar | Schema |
| totalScore | decimal(7,2) | Schema; formula TBD |
| penaltyTime | int | Schema; unit/formula TBD |
| rank | int | Schema; ranking/tie policy TBD |
| completedAt | timestamp | Schema; meaning/update rule TBD |
| createdAt / updatedAt | timestamp | Schema |

### 8.5 GroupContest
| Field | Type | Source status / notes |
|---|---|---|
| groupId | varchar | Schema |
| contestId | varchar | Schema |
| createdAt | timestamp | Schema |

### 8.6 Related CodeSubmission
The conceptual source reconstruction lists problemId, studentId, language, code, judge0Token, verdict, executionTimeMs, memoryUsageKb, failedTestCaseIndex, stdout, stderr, submittedAt, startedAt and completedAt; it does not show direct Contest context fields.

**Cross-module reconciliation:** Module 6's adopted target contract adds one activity context to CodeSubmission. For Contest work, use `contextType=CONTEST` and `contextId=Contest.id`; Module 6 persists CodeSubmission → SubmissionRun → JudgeRun. Module 11 validates ContestProblem association and Student eligibility. This resolves generic submission-context ownership at Module 6 level; Contest attempt limits and score/rank semantics remain open.

## 9. Source-supported Contest status values and lifecycle caveat

| Enum value | Source definition |
|---|---|
| UPCOMING | Present in ContestStatus enum |
| LIVE | Present in ContestStatus enum |
| ENDED | Present in ContestStatus enum |

The source does not define:
- whether status is manually or automatically transitioned;
- exact start/end boundary behavior;
- whether isActive overrides status;
- whether inactive contests can still be viewed or ranked;
- cancellation, archival, deletion, rescheduling, or reopening;
- whether a contest may be published/visible before start;
- whether the server or a scheduled process changes status.

No target state transition is approved in this draft. The historical Event report describes a different lifecycle (draft, published, registration_open, registration_closed, live, grading, completed, plus cancellation behavior) driven by a 30-second watcher and delayed final grading. Those reported states/timings do not map directly to the target ContestStatus enum and are not imported. Module 17 owns target transitions; align FSM-D04/D05 with CON-D04/D05/D17/D20.

## 10. Cross-module dependencies

| Module | Dependency / boundary |
|---|---|
| 1 Authentication & User Lifecycle | Authenticated identity, account status, role/session |
| 2 College / Department / Section | Scope assignment and organizational relationships |
| 3 Subscription & Seat Management | Target source does not define paid Contest entry. Legacy Event reported paid orders/refunds; do not import them. Payment/fee/refund need and ownership are open under CON-D26 and Module 3. |
| 5 Problem Management | Problem definition, visibility, language/runtime settings, hidden testcase protection and immutable ProblemVersion. ContestProblem has no version field in the current conceptual schema; pinning/snapshot strategy is CON-D11. |
| 6 Code Submission & Judge0 | Adopted context contract: `contextType=CONTEST`, `contextId=Contest.id`; CodeSubmission → SubmissionRun → JudgeRun. Contest validates eligibility and Problem association before Module 6 accepts a submission. |
| 7 Test Case Evaluation | Consume completed Module 7 score/verdict results. Do not adopt legacy Event `accepted`/`partial` behavior; Module 7's finalized evaluation contract is authoritative. Contest pointsWeight conversion remains CON-D15. |
| 10 Groups | Group membership and GroupContest association. Module 10 owns GroupContest writes; a GroupContest link does not automatically create ContestParticipant rows or imply team submission/scoring. |
| 12 Certificates | Owns certificate issuance and award criteria. The legacy top-three certificate rule is not target policy. |
| 13 Notifications | Owns notification triggers, recipients and delivery. Legacy 24h/1h reminders are historical only; target schedules remain open under CON-D22. |
| 15 Audit Logging | Canonical event names, payload, retention, privacy |
| 16 Student Analytics | Consumes approved Contest standings/metrics. Legacy ELO/global leaderboard behavior is not target policy; Module 16 ANA-D10/D12/D13/D14 owns analytics/rating decisions (CON-D27). |
| 17 Functional State Machines | Owns canonical Contest/participant/score/rank transitions; old Event cron states/cadence are not copied. Align CON-D04/D05/D17/D20 with FSM-D04/D05. |
| 18 Edge Cases & Failure Handling | Owns retry, idempotency, concurrency, partial failure, scheduler recovery and outbox behavior; no legacy timing/retry values are reused. |

## 11. Decision log

| ID | Question / conflict | Impact if unresolved | Status |
|---|---|---|---|
| CON-D01 | Contest scope: college-only schema vs HOD Department/Manager Section scope | Authorization/data model | Open |
| CON-D02 | Manager Contest permission (CRUD* vs V/C/U*) | Delete and management permissions | Open |
| CON-D03 | Finance Contest view: purpose, page, allowed fields | UI/data access | Open |
| CON-D04 | Contest status vs isActive precedence and automatic/manual transitions | Lifecycle | Open |
| CON-D05 | Start/end time boundaries, timezone, clock source, scheduled transitions | Participation/submission availability | Open |
| CON-D06 | Publish/registration visibility and pre-start details | UX/security | Open |
| CON-D07 | Student self-registration vs staff registration/assignment | Participation | Open |
| CON-D08 | Eligibility: College/Department/Section/Group/enrollment/subscription/batch | Access | Open |
| CON-D09 | Individual vs Group/team participation and GroupContest meaning | Participant model/scoring | Open |
| CON-D10 | Registration close, unregister, duplicate join, capacity | Participation | Open |
| CON-D11 | Contest problem snapshot/versioning and problem updates during Contest | Fairness/content integrity | Open |
| CON-D12 | Contest-specific submission association with CodeSubmission | Data model/result attribution | Open |
| CON-D13 | Submission limits, rate limits, attempts, retries, late submission | Submission | Open |
| CON-D14 | Judge0 execution time vs Contest duration/time limits | Execution behavior | Open |
| CON-D15 | pointsWeight meaning, per-problem points, partial credit, totalScore formula | Scoring | Open |
| CON-D16 | penaltyTime unit and formula, including wrong answers/compile/runtime failures | Penalty | Open |
| CON-D17 | Score recomputation/finalization, manual override, completedAt meaning | Score integrity | Open |
| CON-D18 | Ranking order, provisional/final standings, leaderboard visibility | Ranking | Open |
| CON-D19 | Tie-break rule and equal-rank handling | Rank outcomes | Open |
| CON-D20 | Contest cancellation/archive/delete/reopen/reschedule and history retention | Lifecycle/history | Open |
| CON-D21 | Member changes/GroupContest unlink during live or ended Contest | Eligibility/history | Open |
| CON-D22 | Notifications and audit events/recipients/retention | Cross-module | Open |
| CON-D23 | Unique constraints/indexes for ContestProblem, Participant, GroupContest, ScoreRank | Data integrity | Open |
| CON-D24 | Hidden testcase/result/score visibility and disclosure timing | Privacy/security | Open |
| CON-D25 | Contest score/rank/certificate eligibility dependency | Module 12/16 integration | Open |
| CON-D26 | Does the target Contest support paid entry, fee collection, refunds on cancellation, or free registration only? If paid, define gateway/order/refund ownership and status linkage. | Participation, billing, cancellation and reconciliation; target Contest schema has no paid/fee fields. | Open |
| CON-D27 | Does Contest completion update a platform-wide competitive rating/global leaderboard, or only ContestScoreRank standings? If rating exists, define algorithm, inputs, tie treatment, versioning and idempotency with Module 16. | Rating integrity, cross-event analytics and global standings; legacy ELO values are not target defaults. | Open |
| CON-D28 | Does the target live leaderboard require real-time WebSocket/push updates, or is a read/polling contract sufficient? Define channel authorization and provisional/final payload fields if real-time is required. | Client UX, authorization, infrastructure and stale/rank consistency; legacy Redis/WebSocket behavior is not adopted. | Open |

## 12. Review and readiness status

- [x] Parent module purpose and scope documented.
- [x] Source documents and limitations identified.
- [x] Source role permissions and organizational scope summarized.
- [x] Conceptual schema fields captured without adding fields.
- [x] Contest management functions inventoried and detailed.
- [x] Six submodules linked and separately documented.
- [x] Cross-module boundaries and decision log documented.
- [x] QA companion planned/linked.
- [ ] All CON-D01–D28 decisions resolved.
- [ ] Target behavior approved by product owner.
- [ ] Technical data model/API contracts approved.
- [ ] Canonical state machine defined in Module 17.
- [ ] Implementation readiness approved.

**Review status:** Source-first Module 11 draft updated after reviewing the target module, all six target submodule notes, the historical old-code Module 11 forensic report, main source documents and cross-module contracts. CON-D01–D28 remain open. Do not adopt decisions during the all-modules-first pass. The historical report was reviewed as a document; raw legacy application source was not independently retrieved in this update.
