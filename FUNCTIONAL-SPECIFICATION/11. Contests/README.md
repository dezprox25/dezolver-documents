# DEZOLVER — Module 11: Contests

**Document ID:** DEZ-CON-FS-001  
**Version:** 0.1 — detailed source-grounded draft  
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
| [Functional Specification Standard](../FUNCTIONAL-SPECIFICATION-STANDARD.md) | Function-level detail, evidence labels, acceptance criteria, QA, decision log, readiness checklist. | Governs authoring; does not itself decide Contest behavior. |

### Evidence labels
- **Source-confirmed:** explicitly present in the supplied source documents.
- **Derived:** logical consequence of source facts, not stated as a complete rule.
- **Recommendation:** suggested target behavior, pending product-owner approval.
- **TBD / Decision Required:** absent, ambiguous, or conflicting source evidence.
- **Legacy-verified:** not used in this module because no Contest forensic/legacy audit evidence was supplied for this authoring pass.

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
CodeSubmission contains problemId, studentId, language, code, judge0Token, verdict, executionTimeMs, memoryUsageKb, failedTestCaseIndex, stdout, stderr, submittedAt, startedAt, completedAt. It has no contestId, contestProblemId, participantId, or explicit contest attempt/session field in the conceptual schema.

This is a significant contest-context linkage gap. Do not silently add fields or assume an implementation mechanism.

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

No state transition is approved in this draft. Module 17 must own the canonical transition rules.

## 10. Cross-module dependencies

| Module | Dependency / boundary |
|---|---|
| 1 Authentication & User Lifecycle | Authenticated identity, account status, role/session |
| 2 College / Department / Section | Scope assignment and organizational relationships |
| 3 Subscription & Seat Management | Eligibility only if approved; source does not define Contest subscription gating |
| 5 Problem Management | Problem definition, visibility, language/runtime settings, hidden testcase protection |
| 6 Code Submission & Judge0 | Submission intake, Judge0 job and execution lifecycle |
| 7 Test Case Evaluation | Verdict/evaluation results and testcase processing |
| 10 Groups | Group membership and GroupContest relation; does not establish team participation |
| 12 Certificates | Contest certificate eligibility/issuance if defined there |
| 13 Notifications | Contest lifecycle, participation, results, and rank notifications |
| 15 Audit Logging | Canonical event names, payload, retention, privacy |
| 16 Student Analytics | Contest participation and performance reporting |
| 17 Functional State Machines | Contest/participant/submission/score/rank state transitions |
| 18 Edge Cases & Failure Handling | Timeouts, retry, idempotency, race conditions, partial failures |

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

## 12. Review and readiness status

- [x] Parent module purpose and scope documented.
- [x] Source documents and limitations identified.
- [x] Source role permissions and organizational scope summarized.
- [x] Conceptual schema fields captured without adding fields.
- [x] Contest management functions inventoried and detailed.
- [x] Six submodules linked and separately documented.
- [x] Cross-module boundaries and decision log documented.
- [x] QA companion planned/linked.
- [ ] All CON-D01–D25 decisions resolved.
- [ ] Target behavior approved by product owner.
- [ ] Technical data model/API contracts approved.
- [ ] Canonical state machine defined in Module 17.
- [ ] Implementation readiness approved.

**Review status:** Detailed draft for the all-modules-first authoring pass. Do not resolve the open decisions during this pass.
