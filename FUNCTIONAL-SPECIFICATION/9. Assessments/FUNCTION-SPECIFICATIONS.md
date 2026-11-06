# Module 9 — Assessments: Function-Level Specifications

**Version:** 0.2  
**Status:** Detailed source-grounded draft — target decisions remain open  
**Governing standard:** [Functional Specification Standard v1.0](../FUNCTIONAL-SPECIFICATION-STANDARD.md)  
**Parent module:** [Module 9 — Assessments](./README.md)

> Source-confirmed facts, legacy-verified behavior, derived requirements, recommendations, and TBD decisions remain separate. This document does not approve recommendations or add target schema fields.

## 1. Shared evidence, authorization, and validation rules

### 1.1 Evidence labels
- **[SOURCE—RBAC]**: role action/scope listed in docs/source-text/RBAC.md.
- **[SOURCE—PAGE]**: page/entity listed in Page-to-DB Mapping original or v2.
- **[SOURCE—SCHEMA]**: conceptual field, relationship, or enum in docs/source-text/scheema.md.
- **[SOURCE—BUSINESS]**: workflow stated in Business Flow original/v2.
- **[LEGACY-VERIFIED]**: behavior described in the user-provided Assessment forensic report. Legacy behavior is not automatically target behavior.
- **[DERIVED]**: conclusion drawn from sources.
- **[RECOMMENDATION]**: suggested behavior pending approval.
- **[TBD]**: missing/conflicting decision.

### 1.2 Common authorization rules
- **[SOURCE—RBAC]** SUPER_ADMIN is global; FINANCE_TEAM is assigned-college and billing-oriented; HOD is College + Department; MANAGER is College + Department + assigned Section/permitted Group; STUDENT is own records + permitted academic resources.
- **[SOURCE—RBAC]** Assessment matrix: SUPER_ADMIN V/C/U/D; FINANCE_TEAM —; HOD V/C/U/D; MANAGER V/C/U*; STUDENT V/ATTEMPT. Asterisk means additional scope restriction.
- Assessment page visibility does not override role, scope, or ownership.
- **[RECOMMENDATION]** Backend authorization is required for every operation. Never rely on UI hiding or client-supplied College/Department/Group/Student IDs.
- The source RBAC does not separately define publish, cancel, archive, re-evaluate, score adjustment, integrity review, or export rights. Those action-level permissions are TBD.

### 1.3 Conceptual data reference
**[SOURCE—SCHEMA]** Assessment fields: id, collegeId, departmentId, groupId, createdById, title (varchar 255), description (text), type (AssessmentType), durationMinutes (int), passingPercentage (decimal 5,2), startsAt, endsAt, status (PublishStatus), createdAt, updatedAt.

**[SOURCE—SCHEMA]** AssessmentProblem fields: assessmentId, problemId, orderIndex, createdAt. AssessmentType currently contains CODING; PublishStatus contains DRAFT, PUBLISHED, ARCHIVED.

**Schema gap:** AssessmentAttempt, attempt counters/timestamps, Assessment-level score/result records, Problem weights, snapshots/revisions, integrity records, and manual score history are not shown in the conceptual schema. They are architecture/decision gaps, not confirmed entities.

### 1.4 Shared validation and response principles
- Backend validates identity, role, scope, ownership, state, time, and entity relationships.
- Client validation is for usability only.
- Exact routes, button labels, error strings, API contracts, and response payloads are not supplied; they remain TBD.
- Unresolved behavior must not be silently decided during implementation.
- Pending/operational Judge0 or evaluation failure must not be represented as a Student verdict/final zero without an approved policy.
- Student-facing responses must not expose hidden testcase input/expected output, protected diagnostics, another Student’s records, or staff-only data.

## 2. Functional inventory

| Function ID | Function | Primary actor | Entry point | Type | Status |
|---|---|---|---|---|---|
| ASMT-F001 | View Assessment management list | SUPER_ADMIN, HOD, MANAGER | Assessment management page | Read | Draft |
| ASMT-F002 | Create Assessment draft | SUPER_ADMIN, HOD, MANAGER | Create action | Create | Draft |
| ASMT-F003 | View/edit Assessment configuration | Authorized staff | Assessment record | Read/Update | Draft |
| ASMT-F004 | Add/remove/order Problems | Authorized staff | Problem configuration | Workflow | Draft |
| ASMT-F005 | Configure assignment and eligibility | Authorized staff | Assignment configuration | Workflow | Decision required |
| ASMT-F006 | Publish Assessment | Authorized staff | Publish action | Lifecycle | Decision required |
| ASMT-F007 | Archive/cancel/reschedule Assessment | Authorized staff | Lifecycle actions | Lifecycle | Decision required |
| ASMT-F008 | Browse Student Assessments/details | STUDENT | Student Assessments page | Read | Draft |
| ASMT-F009 | Start Assessment attempt | STUDENT | Start action | Workflow | Attempt policy TBD |
| ASMT-F010 | Resume active workspace | STUDENT | Workspace | Workflow | Resume policy TBD |
| ASMT-F011 | Submit code for Assessment Problem | STUDENT | Submit action | Submit | Modules 6/7 |
| ASMT-F012 | Enforce deadline/expiry | System | Start/timer/submit | System | Timing TBD |
| ASMT-F013 | Process evaluation result | System | Module 7 result | Integration | Draft |
| ASMT-F014 | Aggregate score/pass status | System | Evaluation completion | Calculation | Formula TBD |
| ASMT-F015 | Release/view Student results | Staff/STUDENT | Result view/release | Workflow/Read | Release TBD |
| ASMT-F016 | Review roster/attempts/submissions | HOD/MANAGER/SUPER_ADMIN | Staff review page | Read | Permission TBD |
| ASMT-F017 | Re-evaluate submission | Authorized staff/system | Re-evaluate action | Workflow | Permission TBD |
| ASMT-F018 | Adjust Assessment score | Authorized staff | Score correction | Update | Feature TBD |
| ASMT-F019 | Collect/review integrity signals | System/staff | Workspace/review | System/review | Telemetry TBD |
| ASMT-F020 | Generate reports/export | Authorized staff | Report page/action | Read/export | Export TBD |
| ASMT-F021 | Emit Assessment notification events | System | Lifecycle/result event | Integration | Module 13 |
| ASMT-F022 | Enforce attempt count/grant extra attempts | System/staff | Start/exception action | Workflow | Policy/schema TBD |

---

## 3. Detailed function specifications

### ASMT-F001 — View Assessment management list
**Requirement status:** [SOURCE—PAGE], [SOURCE—RBAC], [DERIVED]  
**Specification status:** Draft / decision required | **Priority:** TBD  
**Actors:** SUPER_ADMIN, HOD, MANAGER | **System actor:** backend query/authorization service

- **Purpose/outcome:** Display Assessments the actor is permitted to manage/review. Page-to-DB Mapping v2 identifies Assessments for HOD and Manager and maps Assessment, AssessmentProblem, Problem, CodeSubmission, User, Department, College.
- **Permissions/scope:** SUPER_ADMIN global; HOD College+Department; Manager College+Department+permitted Section/Group. Finance Team has no Assessment permission in RBAC. Super Admin page/route is not specified in page mapping.
- **Preconditions:** authenticated, active account, Assessment view permission, trusted scope context.
- **Entry/UI:** HOD/Manager Assessments page. List columns, filters, sorting, paging, empty/loading/error UI are TBD. Selecting a record opens details after a separate authorization check.
- **Inputs:** optional search/filter/sort/page; status values from PublishStatus; scope derived server-side. Query syntax and supported filter fields TBD.
- **Main flow:** open page → resolve role/scope → query in-scope records → apply approved filters/paging → display records → reauthorize selected detail.
- **Rules/system/data:** constrain query by role scope before returning rows; do not expose foreign record existence; no data mutation.
- **Validation/failure:** unauthenticated/unauthorized request denied; out-of-scope filter cannot broaden query; query failure returns safe recoverable error; empty results show empty state.
- **Acceptance:** [ ] only scoped records returned; [ ] scope cannot be overridden by client; [ ] Finance denied; [ ] detail endpoint rechecks permission.

### ASMT-F002 — Create Assessment draft
**Requirement status:** [SOURCE—RBAC], [SOURCE—SCHEMA]; workflow details TBD  
**Specification status:** Draft / decision required | **Priority:** TBD  
**Actors:** SUPER_ADMIN, HOD, MANAGER | **System actor:** backend

- **Purpose:** Create a new Assessment record in the approved initial lifecycle state.
- **Permissions/scope:** broad RBAC Create for Super Admin/HOD and Manager within scope; exact draft creation permission/action remains to confirm. Finance/Student cannot create.
- **Preconditions:** authenticated, scoped create permission, valid College/Department context.
- **Entry/UI:** Assessment page → Create action; form and post-save route not source-defined.
- **Inputs:** title (varchar 255); description (text); type (AssessmentType, source currently CODING); collegeId/departmentId (scope); groupId (nullable/schema); createdById (authenticated actor); durationMinutes (int); passingPercentage (decimal 5,2); startsAt/endsAt (timestamps); status (PublishStatus). Requiredness/defaults and field constraints beyond schema types are TBD.
- **Main flow:** open create form → enter approved fields → server checks permissions/scope/relationships/validation → create Assessment and creator association → return record/ID → show saved state.
- **Rules/system/data:** initial DRAFT status is a recommendation until lifecycle approval; type must be enum value; createdById must derive from auth context; save must not auto-publish absent approval.
- **Validation/failure:** invalid enum/foreign scope/relationship rejected; required-field list TBD; duplicate submit/idempotency TBD; persistence failure must not display success.
- **Acceptance:** [ ] creator identity server-derived; [ ] scope validated; [ ] source enum enforced; [ ] creation does not unintentionally expose Assessment to Students.

### ASMT-F003 — View/edit Assessment configuration
**Requirement status:** [SOURCE—RBAC], [SOURCE—SCHEMA], [TBD]  
**Specification status:** Draft | **Actors:** authorized staff

- **Purpose:** Inspect and update allowed Assessment fields.
- **Permissions/preconditions:** View and Update rights checked separately; actor and Assessment must be in scope; edit restrictions by status/attempt are TBD.
- **Entry/UI:** select Assessment from list; display configuration and associated Problems; editable fields, save/cancel and confirmation behavior TBD.
- **Inputs:** Assessment ID; title/description; durationMinutes; passingPercentage; startsAt/endsAt; status; groupId. Type/range/required validation follows source schema plus approved rules.
- **Main flow:** request detail → authorize → display record → edit permitted values → backend validates state/scope/relationships → persist → update updatedAt → display saved state.
- **Rules/data:** updatedAt is source-confirmed. Whether published content can change after first start is unresolved. Recommendation to lock scored content is not approved. Snapshot/revision support is not in schema.
- **Failures:** not found/out of scope denied; lifecycle lock blocks edits if approved; stale concurrent edit handling TBD; save failure preserves unsaved edits where possible.
- **Acceptance:** [ ] field-level update permission enforced; [ ] state restriction enforced; [ ] historical results not silently changed.

### ASMT-F004 — Add/remove/order Assessment Problems
**Requirement status:** [SOURCE—SCHEMA] AssessmentProblem association; [DERIVED] multi-Problem ordered list  
**Specification status:** Draft / design decisions open | **Actors:** authorized Assessment editors

- **Purpose:** Configure Problem membership/order.
- **Permissions/preconditions:** Assessment update permission; selectable Problems exist and are in permitted scope; Assessment content state allows changes.
- **Entry/UI:** Assessment configuration → Problem selector/order editor; UI details TBD.
- **Inputs:** assessmentId; problemId; orderIndex; operation add/remove/reorder. Index convention and duplicate rules TBD.
- **Main flow:** open editor → load selectable scoped Problems → add/remove/reorder → validate Assessment/Problem relationship and state → persist association/order → refresh list.
- **Rules/data:** schema fields are assessmentId, problemId, orderIndex, createdAt. Duplicate association constraint, weights, optional Problems, randomization, and snapshot/versioning are TBD. Legacy one-Problem model is not a target requirement.
- **Failures:** foreign Problem rejected; duplicate handling TBD; locked Assessment blocks edit; write failure leaves prior saved ordering intact.
- **Acceptance:** [ ] links valid records; [ ] UI order reflects orderIndex; [ ] multi-Problem model follows conceptual source; [ ] no implicit score weights.

### ASMT-F005 — Configure assignment and eligibility
**Requirement status:** [SOURCE—SCHEMA] collegeId/departmentId/groupId; [SOURCE—RBAC]; assignment behavior TBD  
**Specification status:** Decision required | **Actors:** authorized staff

- **Purpose:** Define who can view/start Assessment.
- **Permissions/preconditions:** assignment permission approved; Assessment and target entities exist and are in staff scope.
- **Entry/UI:** Assessment assignment section; target selection UI is TBD.
- **Inputs:** collegeId, departmentId, groupId; assignment mode/individual Student IDs are not in schema and require approval.
- **Main flow:** load authorized target entities → actor selects target under approved assignment model → server validates College/Department/Group relation and staff scope → persist assignment → show target.
- **Rules/data:** Assessment has one groupId field; this does not establish multi-Group assignment. Section-wide/individual assignment and membership snapshot timing are unresolved. Recommendation to avoid invalidating active attempts on membership changes is not approved.
- **Failures:** foreign/out-of-scope Group rejected; inactive Group behavior TBD; post-publication assignment change TBD.
- **Acceptance:** [ ] eligibility checked server-side; [ ] client IDs cannot broaden access; [ ] multi-Group/individual assignment not assumed.

### ASMT-F006 — Publish Assessment
**Requirement status:** [SOURCE—SCHEMA] PUBLISHED enum; publish action detail TBD  
**Specification status:** Decision required | **Actors:** staff with publish permission

- **Purpose:** Make Assessment available under approved schedule/assignment.
- **Preconditions:** record exists; publish permission explicitly granted; configuration/Problems/assignment/timing satisfy approved readiness checklist.
- **Entry/UI:** Assessment detail → Publish action; preview/confirmation TBD.
- **Inputs:** Assessment ID; lifecycle command; startsAt/endsAt; status transition to PUBLISHED.
- **Main flow:** request publish → backend checks permission/scope/state/configuration → persist PUBLISHED → emit approved event → display status.
- **Rules/data:** PUBLISHED is source enum. Exact readiness checklist and whether publish immediately notifies are TBD. Publish does not itself create attempts.
- **Failures:** incomplete configuration/invalid schedule blocks if required by approved rules; unauthorized action denied; already-published idempotency behavior TBD; notification failure follows Module 13/18.
- **Acceptance:** [ ] publish permission enforced; [ ] readiness checks performed; [ ] status persisted; [ ] student start eligibility separately checked.

### ASMT-F007 — Archive, cancel, or reschedule Assessment
**Requirement status:** [SOURCE—SCHEMA] ARCHIVED enum; cancellation/reschedule TBD  
**Specification status:** Decision required | **Actors:** authorized staff

- **Purpose:** Stop or change availability while preserving history per policy.
- **Preconditions:** Assessment in scope; action permission approved; active attempt effects resolved.
- **Entry/UI:** lifecycle action from Assessment detail; confirmation/reason fields TBD.
- **Inputs:** Assessment ID; archive/cancel/reschedule command; new startsAt/endsAt for reschedule; reason (requiredness TBD).
- **Main flow:** select operation → display current state/impact if supported → confirm → backend validates transition/scope → persist status/date changes → emit audit/event → display updated state.
- **Rules/data:** ARCHIVED is source enum; CANCELLED is not. Whether cancel is a distinct status or operation is TBD. Recommendation: retain submissions/results, not physical delete.
- **Failures:** invalid transition/dates rejected; active attempts require approved policy; actor out of scope denied.
- **Acceptance:** [ ] archive distinct from cancel unless explicitly combined; [ ] historical records retained; [ ] active attempts follow approved rule.

### ASMT-F008 — Browse Student Assessments and details
**Requirement status:** [SOURCE—PAGE] Student Assessments page is listed in original and v2 mapping  
**Specification status:** Draft / display details TBD | **Actor:** STUDENT

- **Purpose:** Find eligible Assessments and inspect safe details.
- **Preconditions:** authenticated/active Student; Assessment visible under publication/assignment policy.
- **Entry/UI:** Student navigation → Assessments page. Groupings such as Upcoming/Available/In Progress/Completed are recommendations, not source facts.
- **Inputs:** authenticated Student identity; optional filters; Assessment ID for detail. Student ID is never trusted from request.
- **Main flow:** open list → backend resolves Student → returns eligible visible Assessments → display → Student opens detail → backend rechecks eligibility → return Student-safe content.
- **Rules/data:** mapping establishes the Student Assessments page but not card fields/grouping. No hidden test data, staff notes, other Student data, or raw Judge0 payload returned.
- **Failures:** unpublished/ineligible record denied or omitted; empty state; safe query error.
- **Acceptance:** [ ] eligible records only; [ ] detail repeats auth; [ ] safe DTO/data; [ ] Student page mapping reflected.

### ASMT-F009 — Start Assessment attempt
**Requirement status:** [SOURCE—RBAC] Student ATTEMPT; start mechanics TBD  
**Specification status:** Decision required | **Actor:** STUDENT

- **Purpose:** Establish an authorized Assessment session.
- **Preconditions:** Student active/eligible; Assessment published; time window and attempt policy permit start; Problems configured.
- **Entry/UI:** Assessment detail → Start; confirmation/route TBD.
- **Inputs:** Assessment ID; Student identity from auth; attempt number server-derived; client timestamp is not authoritative.
- **Main flow:** request Start → backend rechecks eligibility/status/time/limit → create or resume attempt under approved model → record server start/deadline if supported → return workspace context.
- **Rules/data:** RBAC grants ATTEMPT but does not define attempt entity/counting. Legacy attempt was counted at code-submit API invocation, not workspace open. Recommendation: session-level attempt is not approved. Attempt schema absent.
- **Failures:** before start window/after end/limit reached/ineligible denied; duplicate Start behavior TBD; storage failure must not open a phantom attempt.
- **Acceptance:** [ ] server determines start context; [ ] eligibility enforced; [ ] attempt counting matches approved policy; [ ] retries do not duplicate attempt.

### ASMT-F010 — Resume active workspace
**Requirement status:** [LEGACY-VERIFIED] old client timer/localStorage; target resume TBD  
**Specification status:** Decision required | **Actor:** STUDENT

- **Purpose:** Return to an active attempt after refresh/navigation/reconnect if approved.
- **Preconditions:** active attempt exists and belongs to authenticated Student; Assessment still accessible.
- **Entry/UI:** workspace route or Student Assessment page → Resume; behavior and code autosave are TBD.
- **Inputs:** attempt/Assessment reference; authenticated Student; persisted draft if storage policy exists.
- **Main flow:** request resume → backend verifies ownership/state/deadline → returns safe context and saved work if available → restore same attempt.
- **Rules/data:** legacy timer was browser/localStorage and did not enforce backend deadline. Recommendation: refresh must not reset server deadline. Autosave/draft entity not source-confirmed.
- **Failures:** no active attempt directs to start/detail; expired attempt follows expiry policy; foreign attempt denied; lost draft recovery TBD.
- **Acceptance:** [ ] owner check; [ ] no duplicate attempt on refresh; [ ] deadline not reset; [ ] expired attempts not restored as active.

### ASMT-F011 — Submit code for an Assessment Problem
**Requirement status:** [SOURCE—BUSINESS] backend → Judge0 → evaluation; [SOURCE—RBAC] Student attempt/submission  
**Specification status:** Draft / Modules 6–7 integration contract required | **Actor:** STUDENT

- **Purpose:** Submit code in the correct Assessment/attempt/Problem context.
- **Preconditions:** Student eligible; active attempt if required; Problem belongs to Assessment; deadline and attempt policy permit; language valid under Module 5/6.
- **Entry/UI:** workspace Problem → Submit. Editor/language controls are shared-module responsibility.
- **Inputs:** Assessment/attempt context; problemId; code; language; authenticated Student. Client studentId is ignored or rejected.
- **Main flow:** submit → backend authenticates/authorizes → verifies assessment/attempt/problem relation and time → Module 6 creates CodeSubmission and executes → Module 7 evaluates → context is retained → UI shows receipt/pending.
- **Rules/data:** Judge0 is accessed through backend. CodeSubmission schema associates Problem/User; Assessment/attempt relation is not shown in schema. Assessment duration is not Judge0 CPU limit. Counting multiple submissions is TBD.
- **Failures:** foreign Problem/student denied; Module 6 validation error; duplicate request/idempotency TBD; Judge0 outage is operational failure; submission persistence failure must not report accepted.
- **Acceptance:** [ ] server validates context; [ ] Module 6/7 reused; [ ] no client identity spoof; [ ] system error distinct from verdict.

### ASMT-F012 — Enforce deadline and expiry
**Requirement status:** [SOURCE—SCHEMA] startsAt/endsAt/durationMinutes; semantics TBD  
**Specification status:** Decision required | **Actor:** system

- **Purpose:** Enforce Assessment availability and submission deadline.
- **Preconditions:** timing policy approved; source timestamps available; server clock available.
- **Triggers:** list/detail availability, start, workspace remaining time, submission, expiry event.
- **Inputs:** startsAt, endsAt, durationMinutes, attempt start/deadline if modeled, server time. Client timer is display-only.
- **Main flow:** load schedule/attempt → calculate allowed start/deadline using approved policy → return availability → on submit recalculate with server time → accept/reject/apply approved expiry action → persist state.
- **Rules/data:** legacy timer was client-side; backend did not enforce shared deadline; UI expiration disabled submit but did not autosubmit. Recommendation: server authoritative, but unapproved. Suggested deadline=min(start+duration, endsAt) is not approved. Inclusivity, timezone, grace, late start, pause, auto-submit, saved-code snapshot TBD.
- **Failures:** before start reject; after end/expiry apply policy; browser clock cannot extend time; expiry worker failure must not bypass submit-time check.
- **Acceptance:** [ ] server deadline enforced; [ ] no Judge0 CPU-limit conflation; [ ] boundaries/timezone defined before build.

### ASMT-F013 — Process evaluation result
**Requirement status:** [SOURCE—BUSINESS] evaluation workflow; Module 7 boundary  
**Specification status:** Draft / integration contract required | **Actor:** system

- **Purpose:** Associate completed Problem evaluation with the correct Assessment context without duplicating grading.
- **Preconditions:** accepted CodeSubmission; Module 7 result available; event/retrieval contract defined.
- **Trigger:** Module 7 completion event or result query (choice TBD).
- **Inputs:** CodeSubmission ID, Problem ID, verdict/score/status, Assessment/attempt association, result generation/version if supported.
- **Main flow:** Module 7 completes → Module 9 verifies identity/context → records or references evaluation → updates Assessment progress only under approved rules → result visibility handled by F015.
- **Rules/data:** Module 7 owns testcase evaluation and Problem score; Module 9 owns Assessment aggregation/release. Result tables and event version fields are not source schema. Duplicate events must not duplicate side effects.
- **Failures:** mismatched association rejected; pending/incomplete stays pending; infra failure is not Student verdict; duplicate/stale event handling TBD.
- **Acceptance:** [ ] Module 7 remains score authority; [ ] linkage integrity checked; [ ] no duplicate/stale score corruption.

### ASMT-F014 — Aggregate Assessment score and pass status
**Requirement status:** [SOURCE—SCHEMA] passingPercentage; formula TBD  
**Specification status:** Decision required | **Actor:** system

- **Purpose:** Calculate Assessment score/pass from approved Problem results.
- **Preconditions:** aggregation and attempt-selection policy approved; required evaluations available; passingPercentage valid.
- **Trigger:** evaluation completion, attempt completion, score correction, or release calculation.
- **Inputs:** Module 7 Problem scores; AssessmentProblem list/order; Problem weights (not source schema); passingPercentage; official attempt selection (not source-defined).
- **Main flow:** determine counted Problems → check completeness → apply approved formula → select official attempt score → compare with threshold → persist/derive outcome → expose only under release policy.
- **Rules/data:** legacy had no multi-Problem aggregation or pass threshold. AssessmentProblem only contains orderIndex, not weight. Equal average, weighted sum, highest/latest attempt, boundary, rounding, unanswered/problem failure are unresolved. Do not treat pending/infra error as zero unless approved.
- **Failures:** missing results/invalid config defer or fail calculation according to decision; out-of-range score flagged; rounding TBD.
- **Acceptance:** [ ] formula deterministic; [ ] attempt-selection explicit; [ ] boundary/rounding tested; [ ] incomplete/error states defined.

### ASMT-F015 — Release and view Student results
**Requirement status:** [LEGACY-VERIFIED] immediate result delivery; target release TBD  
**Specification status:** Decision required | **Actors:** staff releaser/STUDENT

- **Purpose:** Control visibility of scores, verdicts, pass status, feedback, and permitted testcase details.
- **Preconditions:** result associated to Student; release policy approved; staff release permission approved if manual release.
- **Entry/UI:** Student result page/panel and staff release action (page details TBD).
- **Inputs:** Assessment/result/attempt ID; authenticated Student; release policy/status (not in source schema); staff command/identity if applicable.
- **Main flow:** request result/release → authorize Student ownership or staff scope/action → check release condition → return permitted fields → display.
- **Rules/data:** legacy immediate result is historical only. Options include immediate, after close, staff release, configurable; none approved. Hidden testcase secrets and protected diagnostics remain excluded.
- **Failures:** withheld result returns safe unavailable state; foreign Student denied; pending evaluation shows pending; event failure handled through Modules 13/18.
- **Acceptance:** [ ] release enforced backend-side; [ ] Student sees own result only; [ ] hidden testcase data excluded; [ ] pending/final distinguished.

### ASMT-F016 — Review roster, attempts, and submissions
**Requirement status:** [SOURCE—PAGE] HOD/Manager Assessments/Student Performance; review rights TBD  
**Specification status:** Draft / permission decision required | **Actors:** HOD/MANAGER/SUPER_ADMIN

- **Purpose:** Monitor Assessment participation and inspect permitted student work/results.
- **Preconditions:** actor has review/view permission; records in scope; source-code/raw execution rights separately approved.
- **Entry/UI:** Assessment staff view, Student Performance, or Assessment detail; page layout/columns TBD.
- **Inputs:** Assessment ID; Student/attempt/submission ID; filters/paging.
- **Main flow:** select Assessment → authorize scope → load permitted roster/status → select Student attempt → recheck scope → display permitted data.
- **Rules/data:** HOD College+Department; Manager assigned Section/permitted Group. View does not imply export, score edit, source-code/raw Judge0 access. Page mapping does not confirm attempt tables.
- **Failures:** foreign scope denied; raw source without permission denied; query errors recoverable.
- **Acceptance:** [ ] scope enforced on every query; [ ] sensitive artifacts permissioned; [ ] hidden data masked.

### ASMT-F017 — Re-evaluate submission
**Requirement status:** [LEGACY-VERIFIED] retry behavior; target re-evaluation action TBD  
**Specification status:** Decision required | **Actors:** authorized staff/system

- **Purpose:** Re-run evaluation under an approved reason/permission.
- **Preconditions:** feature approved; actor/system authorized; submission in scope; state allows re-evaluation.
- **Entry/UI:** staff submission review → Re-evaluate (not source mapped); confirmation/reason TBD.
- **Inputs:** CodeSubmission ID; Assessment context; reason/actor; Problem/testcase version if applicable.
- **Main flow:** request → authorize and check state/concurrency → Module 6/7 re-evaluation → correlate new result → update official score only per policy → audit/notify.
- **Rules/data:** re-evaluation is not a new Student attempt unless approved; preserve old evaluation history; version/snapshot required for reproducibility TBD.
- **Failures:** Student unauthorized; concurrent rejudge policy TBD; infrastructure error does not produce fabricated score; stale result ordering TBD.
- **Acceptance:** [ ] explicit permission; [ ] history preserved; [ ] no unintended attempt count; [ ] stale result handled.

### ASMT-F018 — Adjust Assessment score
**Requirement status:** [LEGACY-VERIFIED] legacy override absent; target feature TBD  
**Specification status:** Decision required | **Actor:** staff with explicit score-adjust permission

- **Purpose:** Apply manual score correction only if approved.
- **Preconditions:** feature enabled/approved; actor authorized/in-scope; result exists; reason/history policy approved.
- **Entry/UI:** staff result review → Adjust score (not source mapped); input/confirmation TBD.
- **Inputs:** Assessment/result/attempt ID; numeric score; reason; actor and timestamp (system).
- **Main flow:** request → authorize → display current score → enter proposed value/reason → validate → persist adjustment/history → recalculate pass status → audit/notify if approved.
- **Rules/data:** legacy manual override was not implemented. Adjustment entity/history not in conceptual schema. Range, precision, precedence against rejudge, and notification policy TBD. Recommendation to preserve original score is unapproved.
- **Failures:** unauthorized actor denied; invalid score/reason rejected; concurrent edit conflict TBD; persistence/audit failure must not claim success.
- **Acceptance:** [ ] disabled unless approved; [ ] actor/reason/time/history recorded if enabled; [ ] pass status reflects approved final score.

### ASMT-F019 — Collect/review integrity signals
**Requirement status:** [LEGACY-VERIFIED] blur/paste telemetry and broken flagging; target policy TBD  
**Specification status:** Decision required | **Actors:** system/reviewer

- **Purpose:** If approved, collect disclosed signals for review without treating them as automatic proof.
- **Preconditions:** telemetry policy, Student notice, privacy/retention/appeal rules approved; reviewer permission exists.
- **Entry/trigger:** workspace event (e.g. blur/paste) and staff review page (not source mapped).
- **Inputs:** event type/time/session/Assessment context; review outcome/reason if supported; data model TBD.
- **Main flow:** approved signal occurs → validate context → store minimal approved event → authorized staff opens review → scope/permission check → inspect and record review outcome if workflow exists.
- **Rules/data:** legacy telemetry existed but background flagging failed due to schema mismatch. It is not target enforcement. Recommendation: minimal telemetry and no auto-fail on blur/paste. Webcam/biometric proctoring not included without approval.
- **Failures:** disabled telemetry means no collection; invalid context rejected; unauthorized reviewer denied; ingestion failure cannot create false flag.
- **Acceptance:** [ ] approved notice/policy required; [ ] access scoped; [ ] signals not treated as proof; [ ] retention/review/appeal defined.

### ASMT-F020 — Generate Assessment reports/export
**Requirement status:** [SOURCE—PAGE] Assessment/Student Performance mappings; report details TBD  
**Specification status:** Decision required | **Actors:** HOD/MANAGER/SUPER_ADMIN

- **Purpose:** Provide scoped Assessment participation/performance information and export only if approved.
- **Preconditions:** report View permission; export separately permissioned; Assessment/records in scope.
- **Entry/UI:** Assessment report/Student Performance; report page/export controls not source-confirmed.
- **Inputs:** Assessment ID; authorized scope filters; optional format/time/attempt/status filters (TBD).
- **Main flow:** open report → authorize → query scoped records → display → if export enabled, authorize separately and generate file.
- **Rules/data:** HOD department scope; Manager Section/Group scope; View does not imply export. Fields, metric definitions, CSV formats, and report state TBD. Hidden testcase/private data excluded.
- **Failures:** out-of-scope filters denied/safely constrained; export permission denied; large report/background job behavior TBD.
- **Acceptance:** [ ] scope enforced; [ ] export permission separate; [ ] sensitive data excluded; [ ] report definitions approved.

### ASMT-F021 — Emit Assessment notification events
**Requirement status:** [SOURCE—SCHEMA] Notification entity; delivery owned by Module 13  
**Specification status:** Draft / event contract required | **Actor:** system

- **Purpose:** Emit approved lifecycle/result milestones for Module 13 delivery.
- **Preconditions:** event/recipient policy approved; relevant operation/result committed.
- **Trigger candidates:** published/assigned, reminder, cancellation/reschedule, result released, score adjusted/re-evaluated. These are candidates, not approved events.
- **Inputs:** event type, Assessment ID, actor/system identity, recipients/scope, minimal metadata.
- **Main flow:** Assessment event occurs → emit event after committed state → Module 13 validates recipients and templates → deliver/retry → record delivery state under Module 13.
- **Rules/data:** Notification source fields: recipientId, title, message, type, linkUrl, isRead, createdAt. NotificationType enum does not explicitly include Assessment; mapping/extension TBD. No hidden testcase/source code in payload.
- **Failures:** invalid recipient rejected; duplicate event/dedup TBD; delivery failure handled by Module 13/18; no unapproved notification.
- **Acceptance:** [ ] approved event only; [ ] valid recipient scope; [ ] no sensitive payload; [ ] retry contract aligned.

### ASMT-F022 — Enforce attempt count and extra-attempt exceptions
**Requirement status:** [LEGACY-VERIFIED] legacy maxAttempts; target field/policy TBD  
**Specification status:** Decision required | **Actors:** system/staff if exception feature approved

- **Purpose:** Enforce approved attempt limit and grant exceptions only with explicit permission.
- **Preconditions:** attempt policy/storage approved; Student eligible; exception permission approved.
- **Entry/trigger:** Start attempt; completion/abandonment/expiry; optional staff grant action.
- **Inputs:** Assessment ID; Student auth identity; attempt limit/count (legacy field only); extra count/reason/actor if enabled.
- **Main flow:** load approved policy → calculate attempts consumed → authorize start or block → persist attempt state → optional authorized exception with reason/audit → display count if allowed.
- **Rules/data:** legacy maxAttempts was checked at submission API invocation, not workspace open; maxAttempts is absent from conceptual Assessment schema. Counting at Start vs first submission, multiple submits, infra failure, abandonment, rejudge, and extra attempts remain TBD. Recommendation: rejudge should not count as Student attempt, pending approval.
- **Failures:** limit reached blocks; client attempt count ignored; unauthorized exception denied; failed persistence cannot increment phantom attempt; duplicate start behavior TBD.
- **Acceptance:** [ ] attempt count server-authoritative; [ ] counting rule approved; [ ] exceptions auditable; [ ] retry/rejudge not accidentally counted.

---

## 4. Cross-function rule and validation register

| Rule ID | Rule | Evidence/status | Applies to |
|---|---|---|---|
| ASMT-R01 | Assessment type limited to source enum CODING unless extended. | [SOURCE—SCHEMA] | F002/F003/F006 |
| ASMT-R02 | Conceptual statuses are DRAFT/PUBLISHED/ARCHIVED. | [SOURCE—SCHEMA] | F003/F006/F007/F008 |
| ASMT-R03 | Student-owned attempts/results use authenticated Student identity. | [SOURCE—RBAC] | F008–F015 |
| ASMT-R04 | HOD scope is College + Department. | [SOURCE—RBAC] | F001–F007/F016/F020 |
| ASMT-R05 | Manager scope is College + Department + permitted Section/Group. | [SOURCE—RBAC] | F001–F007/F016/F020 |
| ASMT-R06 | Judge0 execution is routed through DEZOLVER backend authorization. | [SOURCE—BUSINESS/RBAC] | F011/F013 |
| ASMT-R07 | Assessment duration is separate from Judge0 CPU/runtime limit. | [DERIVED/legacy audit] | F009–F012 |
| ASMT-R08 | Module 7 owns testcase evaluation/Problem score. | [CROSS-MODULE] | F011/F013/F014 |
| ASMT-R09 | Pending/operational evaluation errors are not final grades absent policy. | [RECOMMENDATION] | F013–F015 |
| ASMT-R10 | Hidden testcase data is not Student-visible. | [CROSS-MODULE security] | F008/F011/F015/F016/F020 |
| ASMT-R11 | Published content mutation after attempt start needs explicit policy. | [TBD] | F003/F004/F005 |
| ASMT-R12 | Score aggregation, attempt selection, threshold, rounding, release must be approved. | [TBD] | F014/F015 |
| ASMT-R13 | Legacy timer/attempt/scoring is historical, not target rule. | Functional Specification Standard | F009–F014 |
| ASMT-R14 | View does not imply export, score adjustment, re-evaluation, or raw source access. | [DERIVED/RBAC] | F016–F020 |

## 5. Data/state and architecture gaps

### Source-confirmed conceptual records
Assessment; AssessmentProblem; User; College; Department; Group; Problem; CodeSubmission; Notification; AuditLog.

### Not present in supplied conceptual schema
AssessmentAttempt/session; attempt number/start/deadline/state; AssessmentProblem weight/maxScore/required flag; Assessment content snapshot/revision; Assessment score/result/pass record; result release state; manual score adjustment history; integrity event/review; individual/multi-Group assignment; extra-attempt grant record.

These are gaps—not implicit additions to the target schema.

## 6. Dependencies and ownership

| Dependency | Owner | Boundary |
|---|---|---|
| Problem content/language/resource/testcase configuration | Module 5 | Assessment references Problems |
| Code submission intake, code storage, Judge0 | Module 6 | Assessment provides context; Module 6 executes |
| Testcase evaluation/verdict/Problem score | Module 7 | Assessment consumes results |
| Lab assignment/scoring | Module 8 | Lab rules do not automatically apply to Assessments |
| Groups/membership | Module 10 | Assignment uses approved Group contract |
| Contest rules/ranking/penalty | Module 11 | Contest behavior is distinct |
| Notifications | Module 13 | Module 9 emits events; Module 13 delivers |
| Audit logging | Module 15 | Sensitive action event contract |
| Student analytics | Module 16 | Output mapping requires approval |
| State machines | Module 17 | Assessment/attempt lifecycle alignment |
| Failure/retry/recovery | Module 18 | Shared async/error handling |

## 7. Review readiness checklist
- [ ] Assessment action permissions confirmed.
- [ ] Assignment/eligibility model approved.
- [ ] Attempt/session schema and attempt counting approved.
- [ ] Timing/expiry policy approved.
- [ ] Problem snapshot/version policy approved.
- [ ] Score aggregation and official attempt selection approved.
- [ ] Passing threshold and rounding approved.
- [ ] Result release/feedback policy approved.
- [ ] Re-evaluation/manual adjustment rights and history approved.
- [ ] Integrity telemetry/privacy/appeal policy approved.
- [ ] Reporting/export policy approved.
- [ ] Module 6/7/13/15/17/18 contracts aligned.

**Status:** Detailed function-level draft authored. Decisions intentionally remain open; this module is not implementation-ready.
