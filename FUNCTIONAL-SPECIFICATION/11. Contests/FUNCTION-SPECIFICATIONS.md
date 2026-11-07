# DEZOLVER — Module 11: Contest Management — Function-Level Specifications

**Document ID:** DEZ-CON-FS-002 | **Version:** 0.2 | **Status:** Detailed draft; decisions open  
**Parent:** [Module 11 README](./README.md) | **Standard:** [Functional Specification Standard v1.0](../FUNCTIONAL-SPECIFICATION-STANDARD.md)

> Source facts, recommendations, and TBD decisions are distinct. This document does not approve proposed behavior.

## 1. Shared evidence and authorization

**Source-confirmed:** Contest schema and enum in docs/source-text/scheema.md; Contest page/entity mappings in both Page-to-DB mappings; high-level workflow in both Business Flow diagrams; role permission/scope in RBAC.md.  
**Derived:** List/detail/mutation must respect actor scope.  
**Recommendation:** Backend rechecks permissions/scope for every operation.  
**TBD:** Department/Section scope, Manager scope, Finance view, status/isActive, time boundaries, publish, registration.

### Shared permission matrix
| Role | View/list | Create | Update | Delete | Scope |
|---|---|---|---|---|---|
| SUPER_ADMIN | Allow | Allow | Allow | Allow per Full CRUD | Global |
| FINANCE_TEAM | Source View; exact page/fields TBD | Deny | Deny | Deny | Assigned College |
| HOD | Allow | Allow | Allow | Allow per Full CRUD | College+Department; Contest schema lacks departmentId |
| MANAGER | Allow* | Allow* | Allow* | TBD/deny pending conflict | College+Department+Section/permitted Group |
| STUDENT | View/Participate | Deny | Deny | Deny | Own/permitted academic Contest |

RBAC matrices differ on Manager delete. Page/API matrices show V/C/U*, not Delete. Do not infer Manager delete permission. Frontend visibility is not authorization. Client-supplied role, tenant, Student, or scope IDs are never authoritative.

### Conceptual entities
Contest: id, collegeId, createdById, title, description, startsAt, endsAt, status, isActive, createdAt, updatedAt. ContestStatus: UPCOMING, LIVE, ENDED.  
ContestProblem: id, contestId, problemId, pointsWeight, orderIndex, createdAt.  
ContestParticipant: id, contestId, studentId, joinedAt, createdAt, updatedAt.  
ContestScoreRank: id, contestId, studentId, totalScore, penaltyTime, rank, completedAt, createdAt, updatedAt.  
GroupContest: groupId, contestId, createdAt.  
Conceptual schema does not show a Contest-specific CodeSubmission relation or a team submission/score model.

---

## 1.1 Historical function crosswalk — Event subsystem is not Contest implementation

**Evidence status: LEGACY-SPEC-REPORTED.** The old-code Module 11 forensic specification claims that the following behavior was traced in legacy source. This pass reviewed the report artifact only; raw application source files were not independently fetched. These findings are not approved target behavior.

| Current target function(s) | Legacy routes / functions reported | Target treatment |
|---|---|---|
| CON-F001 / CON-F002 | `GET /events`; `GET /events/:id`; old guard permission `read:cohort`. | Historical list/detail analogue only. Do not copy route or permission strings; use target Contest RBAC and current scope. |
| CON-F003 | `POST /events`; legacy Event create initialized status `draft`. | Create workflow analogue only. Legacy kind, visibility, maxParticipants and paid fee fields are not target Contest fields. |
| CON-F004 | `PATCH /events/:id`; report says edits were draft-only. | Target metadata/schedule policy remains CON-D05/D20; old draft-only restriction is not adopted without approval. |
| CON-F005 | No EventProblem/ContestProblem table or Event config JSON field was reported, despite frontend EventConfig/CompetitionProblem types. | Target ContestProblem is current conceptual-schema evidence. ProblemVersion pinning remains CON-D11. |
| CON-F006 | publish/cancel/extend routes; old EventStateWatcherService reported a 30-second watcher and multi-state lifecycle. | Target has ContestStatus UPCOMING/LIVE/ENDED plus isActive. Target transitions are Module 17 / CON-D04/05/20 decisions. |
| CON-F007 | No GroupContest relation was reported in old Event schema. | No legacy equivalent. Target GroupContest exists in current schema; Module 10 owns relation writes. |
| CPART-F001–F004 | POST registration, DELETE self-unregister, organizer roster GET; legacy Registration had unique (eventId,userId), optional teamName, capacity and payment/cancel statuses. | Historical participation analogue. Target uses ContestParticipant; self/staff registration, duplicates, capacity, withdrawal, team and payment rules remain open. |
| CSUB-F001–F003 | `/events/:id/submissions` was a stub; actual code used Assessment submission with `metadata.eventId`. | Do not use Assessment proxy for target Contests. Module 6 adopted `contextType=CONTEST`, `contextId=Contest.id`, CodeSubmission → SubmissionRun → JudgeRun. |
| CSCORE-F001–F003 | SubmissionGradedConsumer accepted `accepted`/`partial`, upserted EventResult.score and updated Redis. | Module 7's adopted evaluation result is authoritative. Target pointsWeight, component persistence, totalScore and finalization remain CON-D15/17/23. |
| CPEN-F001/F002 | No ICPC penalty calculation, wrong-attempt counter or cumulative solve-time tracker was reported. | `penaltyTime` has no target unit/formula. Do not assume a 20-minute penalty; CON-D16 remains open. |
| CRANK-F001–F003 | Redis `leaderboard:event` used ascending `zrank`; final grading sorted score descending and assigned sequential ranks; equal scores could receive arbitrary order. | Target ordering, provisional/final visibility and tie treatment remain open under CON-D18/19. Do not copy `zrank` or arbitrary row-order tie results. |
| CTIE-F001/F002 | No deterministic tie-break algorithm reported; equal scores could receive arbitrary sequential positions from DB row order. | No arbitrary target tie assignment. Criteria/shared rank/manual adjudication remain CON-D19. |

**Other legacy side effects, not target defaults:** Razorpay paid registration/refunds; 24h/1h reminders; top-three certificate eligibility; pairwise ELO parameters K=32/initial 1200/floor 100; WebSocket/Redis broadcasts. These remain open under CON-D22/D26–D28 and owning Modules 3, 12, 13 and 16.
## CON-F001 — List and filter Contests
**Requirement status:** Source page mapping/RBAC; filters/UI TBD. **Actors:** SUPER_ADMIN, FINANCE_TEAM, HOD, MANAGER, STUDENT.

### Purpose
Show only Contests the actor may discover; do not disclose foreign-scope Contest existence/count/details.

### Preconditions/entry
Authenticated actor; role/scope resolved. Entry: HOD/Manager/Student Contests page; SUPER_ADMIN route and Finance page TBD.

### UI
Page mapping identifies Contests, not layout. Proposed fields: title, schedule, status, actions. Search/filter/sort/pagination and Finance fields TBD. No hidden Problem data, source code, private scores, or other Student details in list.

### Inputs
| Input | Type | Required | Constraint |
|---|---|---|---|
| Search | String | No | Proposed; only authorized rows |
| Status | ContestStatus | No | UPCOMING/LIVE/ENDED |
| isActive | Boolean | No | Meaning relative to status TBD |
| College/Department/Section/Group filter | ID | No | Must not widen scope |
| Page/sort | Integer/enum | No | Contract TBD |

### Main flow
1. Actor opens Contests page.
2. UI requests list.
3. Backend derives identity, role, and scope.
4. Backend evaluates view permission.
5. Backend queries authorized Contest records.
6. Filters apply only within scoped results.
7. Return privacy-safe fields.
8. Render results/empty/failure state.

### Rules/validation
- HOD Department and Manager Section scope cannot be reliably mapped from Contest schema (only collegeId/createdById): CON-D01.
- Finance access must remain College-scoped; field/page decisions CON-D03.
- Invalid status/sort/page filters rejected or normalized under Module 18 TBD.
- Never trust client role or tenant ID.

### Backend/data
Read Contest and any approved scope relation. No mutations; creatorId alone must not substitute for explicit scope.

### Success/failure/edge
Authorized list. Empty state without foreign counts. Foreign scope denied/filtered. Expired session follows Module 1. Detail/action reauthorizes after list. Status/isActive mismatch awaits CON-D04.

### Audit/privacy
Read logging Module 15. Do not expose private participants/results/code.

### Acceptance
1. Only permitted Contest records returned.
2. Client scope cannot widen results.
3. Student list limited to permitted Contests.
4. Finance access College-limited.
5. Status/isActive mismatch not silently normalized.

QA: CON-QA-001–006.

---

## CON-F002 — View Contest details
**Requirement status:** Source page/entity mapping; fields/UI derived/TBD. **Actors:** staff and permitted Students.

### Purpose
Display Contest metadata and associated ordered Problems without bypassing Problem, participant, or submission permissions.

### Preconditions/entry
Authenticated actor; Contest exists and is in scope. Entry from Contest list, Group, or academic page.

### UI
Recommendation: title, description, dates, status, ordered Problem list, and permitted participant/leaderboard links. Pre-start Problem details are CON-D06. Exact fields/routes TBD.

### Inputs
contestId required; optional activity tab/filter TBD.

### Main flow
1. Actor selects Contest.
2. Backend authenticates and authorizes.
3. Load Contest.
4. Load ContestProblem associations/order.
5. Independently authorize Problem, participation, ranking.
6. Display permitted details/actions.

### Rules/validation
- GroupContest does not imply team participation.
- Contest detail does not grant registration, submission, hidden testcase, or private score access.
- Unknown/out-of-scope ID safe not-found/deny.
- Problem visibility follows Module 5.

### Backend/data
Read Contest, ContestProblem, authorized Problem metadata, and permitted Group/participant data. No write.

### Alternate/edge
No Problems; hidden/archived Problem; schedule/status mismatch; Group association outside actor scope; inactive Contest. Follow owning modules/decisions.

### Acceptance
- Foreign Contest not disclosed.
- Problem authorization is separate.
- No hidden testcase/private data leakage.
- Group link does not imply team mode.

QA: CON-QA-007–010.

---

## CON-F003 — Create Contest
**Requirement status:** CRUD source-supported for SUPER_ADMIN/HOD; Manager Create appears with scope restriction. Form rules TBD.

### Purpose
Create a Contest record in authorized College scope.

### Permissions/preconditions
SUPER_ADMIN global; HOD College+Department (Contest has no departmentId); Manager conditional scope; Finance/Student deny. Authenticated actor, authorized College, and required eligibility checks. Entry: Contests → Create.

### Inputs
| Field | Type | Required | Source/constraint |
|---|---|---|---|
| title | varchar(255) | Requiredness TBD; recommend required | Schema length 255 |
| description | text | Optional TBD | Length/markup TBD |
| collegeId | ID | System/authorized | Derived from actor scope |
| createdById | ID | System | Authenticated actor; never client override |
| startsAt | timestamp | TBD | Timezone/future rule CON-D05 |
| endsAt | timestamp | TBD | Date validation TBD |
| status | ContestStatus | System/TBD | Enum source |
| isActive | Boolean | System/TBD | Default/meaning CON-D04 |

### Main flow
1. Actor opens Create.
2. UI displays approved fields and College selection if applicable.
3. Actor enters metadata.
4. Backend rechecks permission/scope.
5. Validate fields, College, schedule and approved lifecycle rules.
6. Create Contest with server-controlled creator/timestamps.
7. Return ID/details and show success.

### Rules
- No Department/Section field added to schema.
- No ContestProblem, participant, Group, or score created implicitly.
- Default status/isActive, title uniqueness, schedule, subscription eligibility TBD.

### Validation
Unauthorized → deny/no record. Missing/blank title → reject if required rule approved. >255 → reject. Foreign College → deny. Invalid date/end-before-start → policy TBD (recommend reject). Client creator/scope override → reject/ignore. Retry duplication → Module 18.

### Backend/data
Create Contest only; atomic write; no participant/score rows.

### Failure/edge
Validation/auth failure leaves no record; duplicate name policy TBD; concurrent create/retry must follow idempotency decision; schedule constraints TBD.

### Audit/privacy
CONTEST_CREATED recommended; Module 15 defines event fields.

### Acceptance
Authorized creation; College scope enforced; schema limits enforced; no implicit association/participant; unapproved defaults not treated as final.

QA: CON-QA-011–016.

---

## CON-F004 — Update Contest metadata and schedule
**Requirement status:** Update source-supported for relevant roles; mutable fields/schedule restrictions TBD.

### Purpose
Edit approved Contest metadata without silently changing creator, College scope, participant, submission, score, or rank history.

### Preconditions/entry
Contest exists and is in scope; actor update permission. Contest detail → Edit.

### Inputs/field policy
| Field | Draft handling | Decision |
|---|---|---|
| title/description | Editable | Requiredness/length/content policy TBD |
| startsAt/endsAt | Dedicated schedule edit | CON-D05/D20 |
| collegeId | Not general-editable | Transfer policy TBD |
| createdById | Immutable | System |
| status/isActive | Dedicated lifecycle operation | CON-D04 |
| ContestProblem | Separate F005 | Do not mutate implicitly |

### Main flow
1. Open Edit; backend authorizes and returns editable fields.
2. Actor submits changed fields.
3. Backend rechecks scope, state, schedule, and validation.
4. Update approved fields and updatedAt.
5. Return updated Contest.

### Rules
- General edit cannot change College, creator, or lifecycle state.
- Schedule edit after registration/start requires CON-D05/D20.
- Concurrent updates use Module 18.

### Validation
Foreign Contest denied; title length/requiredness; date policy; stale edit; unsupported field change. Failed update must not partially mutate.

### Backend/data
Update approved Contest fields only; no participant/submission/score/rank mutation by default.

### Failure/edge
Unauthorized; stale record; live Contest schedule change; ended Contest; invalid date; partial backend failure. History must be preserved.

### Acceptance
Authorized metadata edit works; scope enforced; creator/College/status protected; schedule effects not guessed; history intact.

QA: CON-QA-017–021.

---

## CON-F005 — Add/remove/reorder Contest Problems and weights
**Requirement status:** ContestProblem schema source-confirmed; workflow/weight formula TBD.

### Purpose
Maintain Problem association list/order for a Contest.

### Preconditions/permissions
Actor can manage Contest and selected Problem. Manager scope conditional. Entry Contest → Problems.

### Inputs
contestId, problemId, pointsWeight (decimal 6,2), orderIndex (integer), action (add/remove/reorder/edit weight). Range/order rules TBD.

### Main flow
1. Open configuration; backend lists permitted Problems.
2. Actor selects Problem and weight/order/action.
3. Backend reauthorizes both records.
4. Check duplicate, compatibility, Contest state.
5. Create/update/delete ContestProblem.
6. Refresh list.

### Rules
- ContestProblem has id, contestId, problemId, pointsWeight, orderIndex, createdAt.
- Problem itself is not copied/modified.
- pointsWeight does not establish scoring formula.
- Removal after submission must preserve history; exact behavior TBD.
- Unique association constraint not confirmed.
- Live changes to problems/order/weight TBD.

### Validation
Foreign Contest/Problem denied; duplicate policy TBD; weight range/precision TBD; invalid order TBD; live/ended Contest follows CON-D11/20; Problem lifecycle follows Module 5.

### Backend/data
Association row only; no Problem/submission deletion. Transaction if multirow ordering.

### Alternate/edge
Concurrent reorder, duplicate request, Problem archive, removal after submission, live Contest update; follow decision log.

### Audit
CONTEST_PROBLEM_ADDED/REMOVED/UPDATED recommended; Module 15.

### Acceptance
Only authorized Problem linked; no Problem mutation; weight/order persisted under approved rule; history preserved; lifecycle enforced.

QA: CON-QA-022–027.

---

## CON-F006 — Contest lifecycle/status and active flag
**Requirement status:** Enum and isActive field source-confirmed; transitions TBD.

### Purpose
Control Contest availability while retaining history.

### Actors/preconditions
SUPER_ADMIN/HOD/conditional Manager or trusted scheduler; Contest exists/in scope. Entry lifecycle control or scheduled trigger.

### Inputs
contestId; requested operation/state; optional reason/time TBD. Client cannot assign arbitrary lifecycle state.

### Main flow
1. Staff/system requests transition.
2. Backend identifies actor/trusted scheduler.
3. Validate permission, current state, schedule, and participant/submission impact.
4. Apply approved transition.
5. Persist status/isActive according to Module 17.
6. Emit approved audit/notification.
7. Return state.

### Rules
- ContestStatus: UPCOMING, LIVE, ENDED.
- isActive separate Boolean; precedence CON-D04.
- No source-defined CANCELLED/ARCHIVED status.
- ENDED does not imply archived/inactive.
- Scheduler, boundaries CON-D05; reopen/cancel/reschedule/delete CON-D20.

### Validation
Unauthorized transition denied; invalid enum rejected; invalid current-state transition blocked; scheduler/manual race Module 18; repeated transition idempotency TBD; live submissions grace policy TBD.

### Backend/data
Approved state transition only; no participant/score deletion.

### Failure/edge
Invalid transition unchanged; schedule boundary, scheduler delay, clock skew, status/isActive mismatch, repeated event, active submission handling TBD.

### Audit
CONTEST_STATUS_CHANGED recommended; Module 15.

### Acceptance
Only authorized transitions; invalid transition no mutation; state semantics consistent; history retained; boundary handling approved.

QA: CON-QA-028–033.

---

## CON-F007 — Associate Group with Contest
**Requirement status:** GroupContest schema and Group-mediated flow source-confirmed; assignment/team rules TBD.

### Purpose
Link Group and Contest for documented Group-mediated workflow.

### Preconditions/permissions
Actor can manage both Group and Contest; both records in scope. Entry from Group or Contest detail.

### Inputs
groupId, contestId, associate/unassociate action.

### Main flow
1. Open assignment UI; server returns permitted Groups/Contests.
2. Actor selects pair.
3. Backend validates role, scope, eligibility, state.
4. Create GroupContest relation.
5. Show association.

### Rules
- GroupContest fields groupId, contestId, createdAt.
- Business Flow says Group mediates Contest; it does not define team participation, shared submission, Group score, or rank.
- Association does not automatically create ContestParticipant rows unless approved.
- Roster snapshot, join timing, late-member eligibility TBD.
- Unlink must not erase participant/submission/score history.

### Validation
Group/Contest permission failure or foreign scope → deny; duplicate association policy TBD; inactive Group/ended Contest policy TBD; Group changes during live Contest CON-D21.

### Backend/data
Create/delete GroupContest only. No automatic participants/score.

### Failure/edge
Duplicate, cross-scope, inactive Group, Contest live/ended, membership changes, unlink after activity: decision required.

### Audit
GROUP_CONTEST_LINKED/UNLINKED recommended; Module 15.

### Acceptance
Both entities authorized; relation persists; no team mode inferred; unlink retains Contest history.

QA: CON-QA-034–038.

### Cross-module contract reconciliation

- **Module 5 — Problems:** ContestProblem selects a Problem. Target ContestProblem has no ProblemVersion reference, while Module 5 provides immutable ProblemVersion snapshots. Version pinning and behavior after Problem edits remain CON-D11; do not infer a snapshot mechanism.
- **Module 6 — Code Submission & Judge0:** each formal CodeSubmission has one activity context. Contest uses `contextType=CONTEST`, `contextId=Contest.id`; persistence is CodeSubmission → SubmissionRun → JudgeRun. Module 11 validates ContestParticipant, ContestProblem and schedule eligibility; Module 6 owns intake/Judge0. The old Assessment-proxy route is not the target contract.
- **Module 7 — Test Case Evaluation:** consume the completed canonical evaluation result and finalized score/verdict. Infrastructure/incomplete runs are not wrong answers or zero scores. Contest-specific pointsWeight, multiple-submission selection and totalScore aggregation remain CON-D15/17.
- **Module 10 — Groups:** Group membership/GroupContest is not equivalent to ContestParticipant or team submission. A GroupContest link must not create participants or shared submissions without a decision.
- **Module 9 — Assessments:** Contest submissions use CONTEST context and Contest-owned timing/participation policies. Do not inherit AssessmentAttempt/duration/result-release rules or retain the legacy Assessment proxy.
- **Modules 12/13/15/16/17/18:** certificate criteria, notification triggers, audit fields/retention, platform rating, state transitions and recovery policies remain with those modules. Legacy ELO, reminder, WebSocket and cron settings are not adopted.
## Cross-reference## Cross-reference
Submodule specs:
- 11.1 Participation
- 11.2 Submission
- 11.3 Scoring
- 11.4 Penalty
- 11.5 Ranking
- 11.6 Tie Breaking
