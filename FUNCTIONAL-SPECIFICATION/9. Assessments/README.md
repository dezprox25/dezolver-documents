# 9. Assessments

**Document type:** Functional Specification  
**Version:** 1.0 — Adopted target baseline  
**Status:** Product/architecture/security decisions adopted (ASMT-D01–ASMT-D36); engineering and QA execution pending  
**Module owner:** DEZOLVER Academic Activities / Assessments  
**Related modules:** College / Department / Section Management, Course & Enrollment, Problem Management, Code Submission & Judge0, Test Case Evaluation, Labs & Lab Scoring, Groups, Contests, Notifications, Audit Logging, Student Analytics, Functional State Machines, Edge Cases & Failure Handling

> **Adopted target baseline:** Conceptual source fields, verified legacy-code audit facts and adopted target behavior remain distinguished. All 36 ASMT-D decisions are now adopted. Sections 26–31 and ADOPTED-DECISIONS.md are normative and supersede earlier recommendation/TBD annotations. Engineering implementation and QA execution remain pending.

## 1. Purpose

Define how DEZOLVER creates, assigns, publishes, manages, takes, evaluates, scores, and reports Coding Assessments.

An Assessment is a scheduled or otherwise available academic evaluation activity associated with an academic scope and one or more Problems. Students participate individually in the Assessment unless collaborative behavior is separately approved. Assessment submissions reuse the Code Submission and Test Case Evaluation capabilities rather than creating a separate execution/evaluation engine.

This draft establishes source-grounded scope and proposed workflows. Decisions affecting target architecture, database design, scoring, timing, attempts, and Student workflow remain explicitly open.

## 2. Source evidence and interpretation

### 2.1 Business Flow v2

**[SOURCE—BUSINESS]** Business Flow v2 establishes a high-level coding workflow in which a Student submits code through the DEZOLVER backend, code is executed through Judge0, evaluation produces a result, and the result contributes to a score.

This is a general code submission/evaluation flow. It does not define Assessment-specific timing, attempt counting, scheduling, score aggregation, result release, or anti-cheat rules.

### 2.2 Page-to-DB Mapping v2

**[SOURCE—BUSINESS]**
- HOD and Manager page/workflow mappings include Assessments.
- Assessment-related entities include Assessment, AssessmentProblem, Problem, CodeSubmission, User, Department, and College.
- The Student page inventory includes an Assessments page. The specific page layout, filters, status groupings, and detail workflow are not defined by the mapping.

The mapping identifies page/entity relationships; it does not fully specify user interactions or action-level permissions.

### 2.3 Conceptual schema

**[SOURCE—SCHEMA]** The conceptual Assessment entity includes:

| Field | Conceptual meaning |
|---|---|
| id | Assessment identifier |
| collegeId | College association |
| departmentId | Department association |
| groupId | Group association |
| createdById | Creator/User association |
| title | Assessment title |
| description | Assessment instructions/description |
| type | AssessmentType |
| durationMinutes | Assessment duration |
| passingPercentage | Passing threshold |
| startsAt | Scheduled availability start |
| endsAt | Scheduled availability end |
| status | PublishStatus |
| createdAt / updatedAt | Record timestamps |

Conceptual enums:
- AssessmentType: CODING
- PublishStatus: DRAFT, PUBLISHED, ARCHIVED

The conceptual AssessmentProblem association includes assessmentId, problemId, orderIndex, and createdAt. It connects an Assessment to Problems and represents ordering.

**Important:** These are conceptual source fields and relationships. They are not proof that the old implementation contained them or that every associated behavior has already been decided.

### 2.4 RBAC source

**[SOURCE—RBAC]** General role scope in the RBAC documents:
- SUPER_ADMIN: global
- FINANCE_TEAM: college billing
- HOD: College and Department
- MANAGER: College, Department, Section/Group
- STUDENT: own or authorized data

The RBAC source grants SUPER_ADMIN/HOD V/C/U/D, MANAGER V/C/U* and STUDENT V/ATTEMPT; Finance is denied. Module 9 resolves lifecycle/release/rejudge to narrower actions in ASMT-D01/D10/D24/D27/D28.

### 2.5 Related module boundaries

- Module 5 owns Problem authoring, Problem settings, and test cases.
- Module 6 owns code submission intake and Judge0 integration.
- Module 7 owns testcase evaluation, verdicts, and Problem-level score.
- Module 8 owns Lab assignment/scoring; Assessment rules must not silently inherit Lab rules.
- Module 10 owns Group creation and membership.
- Module 11 owns Contest-specific participation, penalty, ranking, and tie-breaking.
- Module 13 owns notification infrastructure.
- Module 15 owns audit event definitions.
- Module 17 owns canonical state-machine definitions.
- Module 18 owns cross-module failure/recovery behavior.

### 2.6 Legacy code forensic report

**[AUDIT—LEGACY]** The completed user-provided Assessment forensic report is the factual reference for old-code behavior. Section 3 summarizes its findings.

The report is not a target requirements document. Do not copy its schema or behavior automatically.

## 3. Confirmed legacy implementation facts

This section describes the audited older source snapshot only.

### 3.1 Legacy Assessment model and relationships

The old physical Assessment model contained fields including:
- id, kind, title
- problemId, roomId
- maxAttempts, timeLimitMinutes
- collectAntiCheat
- createdByUserId
- createdAt, updatedAt

The audited legacy model did not contain the conceptual Assessment fields status, startsAt, endsAt, passingPercentage, or groupId. It did not use the conceptual AssessmentProblem junction to associate multiple Problems.

Legacy Assessment referenced at most one Problem using a scalar problemId. The code audit found no Problem content snapshot/versioning mechanism; content was read from the current Problem record.

The old code used individual user submissions and had no Group-based Assessment assignment/membership model.

### 3.2 Legacy lifecycle and timer

- Assessments were effectively active after creation; status was not persisted as the conceptual lifecycle.
- Frontend Publish/Archive status updates were not correctly persisted by the old backend.
- The timer was browser-side, initialized when the workspace mounted, and stored using localStorage.
- The backend did not enforce a shared Assessment deadline.
- Client timer expiration disabled submission in the UI; it did not automatically submit the latest work.
- Legacy timeLimitMinutes was also passed as a Judge0 CPU execution limit, mixing Assessment duration and code execution limits.
- Scheduling, late-start policy, cancellation, and rescheduling were not implemented.

### 3.3 Legacy attempts and evaluation

- maxAttempts existed and was checked in the submission path.
- A legacy attempt was counted when the code-submission API was invoked, not when the Student opened the workspace.
- Legacy submissions were individual records; they did not represent a full Assessment session containing multiple Problem-level submissions.
- There was no official final-score selection rule across attempts (e.g. highest vs latest).
- Evaluation jobs used retries; the forensic report records up to three retries with exponential backoff. Terminal evaluation failure could set an error/internal_error result. The attempt was consumed and was not refunded.

### 3.4 Legacy scoring and result delivery

- Problem-level coding score used weighted testcase credit: passed testcase weight / total testcase weight × 100, rounded.
- There was no Assessment-level Problem weight, passingPercentage threshold, or defined multi-Problem Assessment aggregation.
- Results were delivered immediately after grading through realtime delivery and/or polling.
- Student result details included score/verdict/runtime/memory and testcase pass/fail information; hidden testcase output was protected, though the audit flagged potential disclosure through some diagnostic fields.
- Manual score override was not implemented.

### 3.5 Legacy integrity, UI, and reporting

- Workspace telemetry included tab/window blur and paste events.
- Anti-cheat warnings/toasts existed, but the background flagging path failed due to a schema mismatch. It was not a reliable automated enforcement feature.
- Student routes included /assessments and /assessments/:id/workspace. There was no dedicated My Assessments dashboard organized by upcoming/in-progress/completed.
- Assessment publication/deadline/result-release notifications were not implemented.
- The audit does not establish a target requirement for cohort/department reporting, export, or manual grading.

**Conclusion:** The legacy implementation is substantially different from the conceptual Assessment model. The target system should be specified from the approved conceptual sources and explicit product decisions.

## 4. Scope

### 4.1 Included
- Assessment creation and configuration
- Assessment–Problem association
- Academic scope and assignment eligibility
- Draft, publish, archive lifecycle
- Availability window and per-student duration policy
- Student Assessment list/details/workspace
- Assessment attempts and Problem submissions
- Code execution and evaluation integration
- Problem-level and Assessment-level score aggregation
- Passing threshold
- Result visibility and release policy
- Staff review, score correction, and re-evaluation policy
- Integrity/anti-cheat policy
- Assessment reporting
- Notifications/events
- Authorization, validation, and audit requirements
- Failure cases and acceptance criteria

### 4.2 Out of scope / owned by other modules

| Topic | Owner |
|---|---|
| Problem statement, allowed languages, resource limits, testcases | Module 5 |
| Code submission API and Judge0 execution | Module 6 |
| Testcase evaluation and Problem-level score | Module 7 |
| Lab assignment, Lab score, late policy | Module 8 |
| Group creation and membership | Module 10 |
| Contest rules, penalties, ranks, tie-breakers | Module 11 |
| Notification delivery infrastructure | Module 13 |
| Audit event catalog | Module 15 |
| Student analytics platform | Module 16 |
| Canonical Assessment/submission state machines | Module 17 |
| Shared edge cases, retries, recovery | Module 18 |

**Current conceptual scope:** CODING Assessment only. Quiz/MCQ Assessment types are not confirmed by the conceptual AssessmentType enum and should be treated as a scope extension unless separately approved.

## 5. Actors and permissions

The following target action-level mapping reconciles the supplied RBAC matrix with least-privilege scope. Publish/release/rejudge actions are narrower than general CRUD.

| Role | Proposed Assessment access |
|---|---|
| SUPER_ADMIN | Global manage, lifecycle and audited privileged raw inspection |
| HOD | Create/view/update/publish/archive within assigned College+Department; scoped review |
| MANAGER | View/create/update drafts within assigned Section/permitted Group; no lifecycle/Delete |
| STUDENT | View eligible Assessments, start/submit own attempts, and view released own results |
| FINANCE_TEAM | No Assessment access |

### 5.1 Authorization requirements

**[RECOMMENDATION]**
- Enforce authorization on the backend for every create/read/update/publish/archive/start/submit/result/report action.
- Validate College, Department, Section/Group, Student eligibility, and activity scope server-side.
- Students may access only Assessments assigned to them or to a Group/Section in which they are eligible.
- Students may view only their own attempt/submission details and permitted result data.
- HOD/Manager access must not exceed the role's assigned academic scope.
- Client-supplied studentId, collegeId, departmentId, or groupId must not be trusted as the authorization source.
- Score adjustment, re-evaluation, and access to raw execution artifacts should require explicit permissions and audit records.

### 5.2 Open permission decision

Confirm which roles may:
- Create and edit Assessment drafts
- Publish/unpublish
- Assign to Groups or students
- Cancel/reschedule
- View all Student submissions/source code
- Re-evaluate submissions
- Adjust scores
- Release results
- Export reports

Do not infer these rights solely from a page mapping.

## 6. Conceptual data model

### 6.1 Assessment

The source conceptual fields are listed in Section 2.3. They are reference fields, not a final target schema.

### 6.2 AssessmentProblem

Conceptual fields:
- assessmentId
- problemId
- orderIndex
- createdAt

The association supports multiple Problems and ordering conceptually.

**Open target design questions:**
- Is the association unique per Assessment–Problem pair?
- Should it contain Problem weight, required/optional flag, max score, or per-Problem settings?
- Is randomization supported?
- Is a Problem revision/snapshot reference required?
- Should duplicate Problems within one Assessment be allowed?

### 6.3 Recommended additional entities/relationships

**[RECOMMENDATION—NOT APPROVED SCHEMA]** Consider a distinct AssessmentAttempt entity if one attempt is intended to represent the full Student Assessment session. It could associate:
- Assessment
- Student
- Attempt number
- Start/finish timestamps
- State
- Score/result snapshot
- Eligibility/content revision
- Completion reason

Problem-level CodeSubmission records would then be linked to the AssessmentAttempt and AssessmentProblem.

This is a recommendation, not a claim that such an entity exists in the conceptual schema or legacy code.

**Architecture dependency:** Decide D6 (attempt model) before finalizing tables, uniqueness constraints, submission context, or APIs.

## 7. Assessment creation and configuration

### 7.1 Proposed creation flow

1. Authorized staff opens Assessment management.
2. Staff creates a draft.
3. Staff enters title and description/instructions.
4. Staff chooses academic scope (College/Department and applicable Group/assignment target).
5. Staff adds one or more Problems.
6. Staff configures Problem order and any approved weights/settings.
7. Staff configures availability window and duration.
8. Staff configures attempts, scoring, pass threshold, and result visibility.
9. Staff reviews the Assessment.
10. Staff publishes it.

Steps 4–8 depend on product decisions. The UI should not expose unapproved policy settings.

### 7.2 Draft validation

**[RECOMMENDATION]** Before publication, validate:
- Title is present and within length limits.
- Assessment has at least one Problem.
- All Problems are available and in creator's authorized scope.
- Assignment target is valid.
- Timing fields are internally consistent.
- Duration is positive.
- Attempt configuration is valid.
- Passing percentage is within 0–100.
- Scoring configuration is valid.
- Student-facing instructions and result policy are configured.
- Any required anti-cheat notice is present if telemetry is enabled.

### 7.3 Problem association

Conceptual sources support multiple Problems via AssessmentProblem. The legacy system supported one Problem only.

**Adopted:** use the conceptual multi-Problem AssessmentProblem junction, ordered by zero-based orderIndex. Each published Assessment revision pins each Problem's published Module 5 ProblemVersion; see ASMT-D03–D05 and D11–D13.

## 8. Assignment and eligibility

### 8.1 Conceptual scope

The conceptual Assessment includes collegeId, departmentId, and groupId. The source does not definitively establish individual Student assignment, multi-Group assignment, or how Group membership changes affect eligibility.

### 8.2 Assignment options requiring decision

- One Assessment assigned to one Group
- One Assessment assigned to multiple Groups
- Group assignment plus individual Student exceptions
- Individual Student assignment
- Department/Section-wide assignment

**Adopted:** one optional groupId plus explicit AssessmentSection rows. Individual Student assignment and multiple Groups per Assessment are deferred in v1; Module 10 remains the authority for Group membership.

### 8.3 Membership changes

**[RECOMMENDATION]** Define eligibility at a stable checkpoint, preferably when the Student starts an attempt:
- A newly eligible Student may start if the Assessment remains available.
- An already-started attempt should not be unexpectedly invalidated solely by a membership change.
- Removed Students should not start new attempts.
- Historical attempts/results remain retained.

Adopted by ASMT-D07/D08: eligibility is rechecked at list/detail/start; membership removal blocks new starts but does not invalidate an active attempt.

## 9. Assessment lifecycle

### 9.1 Conceptual statuses

The conceptual enum includes:
- DRAFT
- PUBLISHED
- ARCHIVED

No additional persisted status is source-confirmed.

### 9.2 Proposed lifecycle behavior

| Status | Proposed meaning |
|---|---|
| DRAFT | Staff-only configuration; not available to Students |
| PUBLISHED | Assigned/eligible Students may access it according to schedule |
| ARCHIVED | Historical record retained; no new starts |

Adopted by ASMT-D09/D10: DRAFT/PUBLISHED/ARCHIVED only, archive is history-only, restore to DRAFT and no hard delete.

### 9.3 Editing after publication

**[OPEN DECISION]** Choose:
- Allow all published fields to be edited
- Lock after first attempt starts
- Published Assessment immutable; clone to change
- Permit limited metadata/date edits while locking Problem/scoring content

Adopted by ASMT-D11–D13: every publish creates an immutable AssessmentRevision; any published content/configuration change requires a new draft revision and republish.

## 10. Student Assessment workflow

### 10.1 Student list / My Assessments

**[PAGE-MAPPING]** HOD/Manager Assessment pages are mapped. A dedicated Student My Assessments page is not clearly established in the source inventory.

**[RECOMMENDATION]** Provide a Student Assessment list organized by:
- Upcoming
- Available
- In Progress
- Completed
- Closed/Archived (history)

Each card/row should show title, availability, duration, attempt policy, status, and result visibility state.

### 10.2 Assessment details

Student should be able to view:
- Title and instructions
- Problem list and order
- Availability window
- Duration
- Attempt limit/current attempt count
- Allowed languages per Problem
- Submission rules
- Result/feedback release policy
- Any integrity monitoring notice

Hidden testcase details must follow Module 7 confidentiality rules.

### 10.3 Workspace and navigation

**[RECOMMENDATION]**
- Display server-derived remaining time.
- Allow navigation among Problems if the approved policy permits it.
- Preserve code drafts safely if auto-save/resume is approved.
- Show submission state and evaluation progress.
- Do not treat workspace open/refresh as an attempt unless product decision explicitly says so.
- Provide clear behavior for browser refresh, disconnect, and reconnect.

### 10.4 Resume/reconnect

**[OPEN DECISION]** Should Students resume an active attempt after closing the browser, disconnecting, or navigating away?

**[RECOMMENDATION]** Resume the same active attempt within the original deadline. Do not reset timer or create a new attempt due to refresh/reconnect.

## 11. Timing, start, and expiry

### 11.1 Source facts

Conceptual fields include startsAt, endsAt, and durationMinutes. The source does not define their precedence or deadline semantics.

Legacy timer behavior was client-side and did not enforce server-side expiry. This is not the target recommendation.

### 11.2 Timing decisions

Resolve:
- Whether duration begins on explicit Start or another event
- Whether duration is per Student attempt or shared
- Whether late start is allowed
- Whether the scheduled endsAt cuts off all attempts
- Whether active attempts may continue after endsAt
- Whether expiry auto-submits, submits the last saved code, or locks further submissions
- Whether pause/resume is allowed

### 11.3 Technical recommendation

Use server-authoritative timing:
- Server records attempt start time.
- Remaining time is derived from server timestamps.
- Browser timer is presentation only.
- Submission API validates deadline server-side.
- Assessment duration must not be reused as Judge0 CPU time limit; execution limits belong to Problem/Judge configuration.
- Deadline and grace-period logic should be explicit and testable.

Adopted by ASMT-D14–D16: server-authoritative deadline is min(server start + duration, revision endsAt when present); submissions are accepted at the inclusive deadline.

### 11.4 Expiry and autosubmit

**[RECOMMENDATION]** If auto-submit is desired, save drafts server-side and define what is submitted at expiry. Do not assume the browser has the latest code. If auto-submit is not desired, lock submission at deadline and preserve the last accepted submission/draft according to approved policy.

## 12. Attempts and submission model

### 12.1 Legacy facts

Legacy maxAttempts was enforced at the submission API. Opening workspace did not consume an attempt; calling the code submission endpoint did. The old system did not model a multi-Problem Assessment session as a parent attempt.

### 12.2 Target decisions

- Does an Assessment attempt represent a full session or a single code submission?
- When is attempt number consumed: explicit Start, first code submission, or another event?
- Are multiple submissions per Problem allowed within one attempt?
- Are multiple attempts configurable?
- Does evaluation failure consume an attempt?
- Can staff grant an extra attempt?
- What happens when an attempt is abandoned or expires?

### 12.3 Technical recommendation

**Recommended architecture:** one AssessmentAttempt represents a full Assessment session. Each Problem may have one or more CodeSubmission records under the attempt. Count maxAttempts at the session level, not per Submit click.

Suggested semantics:
- Create an attempt on server-accepted Start.
- Resume the existing active attempt after refresh/reconnect.
- Preserve every code submission and evaluation record.
- Do not refund an attempt for a student-caused submission; define separate recovery for infrastructure failures.
- Do not count re-evaluation as a new Student attempt.
- Store attempt number and timestamps on the parent attempt.

Adopted by ASMT-D18/D19/D33: one AssessmentAttempt represents a full session; maxAttempts defaults to 1 and is counted on successful Start.

## 13. Evaluation and scoring

### 13.1 Evaluation boundary

**[SOURCE—BUSINESS]** Code Submission → Judge0 → Evaluation → Result/Score.

Module 9 should use Module 7's approved Problem-level evaluation result. It should not duplicate testcase execution or comparator logic.

### 13.2 Problem-level score

**[AUDIT—LEGACY]** The old evaluator used weighted testcase scoring, converted into a percentage. The conceptual Assessment model does not confirm the same algorithm as the target rule.

**[RECOMMENDATION]** Treat the approved Module 7 Problem score as the input to Assessment scoring. Do not hardcode legacy scoring logic inside Assessment.

### 13.3 Assessment-level aggregation

The conceptual AssessmentProblem association has orderIndex but no source-confirmed Assessment Problem weight.

**[OPEN DECISION]**
- Equal-weight average
- Configurable per-Problem weights
- Sum of Problem points
- Another aggregation rule

Adopted by ASMT-D05/D21: all AssessmentProblems are required and equally weighted; Assessment score is the arithmetic mean of their selected Module 7 percentages.

### 13.4 Passing threshold

The conceptual model includes passingPercentage. Legacy code did not enforce an Assessment-level pass threshold.

Adopted by ASMT-D17/D21/D22: score is normalized 0–100; unanswered Problems count as 0; pending/infrastructure-failed Problems block finalization; pass when final score >= passingPercentage.

### 13.5 Final score across attempts

**[OPEN DECISION]** Highest, latest, first, average, or another attempt score?

Adopted by ASMT-D23: highest finalized valid AssessmentAttempt is official; latest started attempt breaks ties. Preserve all attempt results.

### 13.6 Partial credit and evaluation failures

**[RECOMMENDATION]**
- Reuse Module 7 partial-credit policy.
- Do not silently treat pending, incomplete, or infrastructure-error evaluation as zero or full credit.
- Clearly distinguish student code failure from Judge0/worker/system failure.
- Define whether terminal infrastructure failure allows retry without consuming an additional Assessment attempt.
- Store grading provenance so a re-evaluation can be explained.

## 14. Result visibility and release

### 14.1 Legacy result delivery

Legacy results were delivered immediately after evaluation. That is a historical behavior only.

### 14.2 Target options

- Immediate score and detailed testcase feedback
- Immediate score/verdict but hidden testcase details withheld
- Results withheld until Assessment closes
- Results released by staff
- Per-Assessment configurable release policy

### 14.3 Recommendation

Allow per-Assessment result policy, with a safe default for scheduled/shared Assessments:
- Student sees submission receipt and processing status immediately.
- Score/verdict may be shown according to configured policy.
- Hidden testcase inputs/expected outputs and revealing diagnostics remain protected.
- Final score/pass status is released according to the Assessment's configured release policy.
- Staff may release results where permitted.
- Result visibility must be enforced by backend APIs, not only UI.

## 15. Staff review, re-evaluation, and score adjustment

### 15.1 Legacy facts

Legacy manual score override was not implemented. This does not decide whether the target system should support it.

### 15.2 Proposed capabilities

Authorized staff may need:
- View Student roster and attempt status
- View permitted submissions and evaluation results
- Re-evaluate a submission
- Review suspicious integrity signals
- Adjust score with a reason
- Release results
- Export scoped Assessment report

### 15.3 Recommendation

- Re-evaluation and manual score adjustment are separate actions.
- Preserve original automated score and evaluation history.
- Score adjustment requires permission, reason, actor, timestamp, and audit record.
- Re-evaluation creates a new evaluation history/version rather than silently erasing old grading evidence.
- Downstream pass/fail and reports should reflect the approved final score.
- Notify Student if score/result changes, subject to notification policy.

Adopted by ASMT-D27/D28: rejudge is SUPER_ADMIN-only under Modules 6/7; manual score override is disabled in v1.

## 16. Integrity / anti-cheating

### 16.1 Legacy facts

The old workspace captured tab/window blur and paste signals. Its automated flagging path failed due to schema mismatch. It did not reliably terminate an Assessment based on suspicious behavior.

### 16.2 Target decision

Choose:
- No browser telemetry
- Limited telemetry (tab/window change, paste) with staff review
- Stronger proctoring as a separately scoped feature

### 16.3 Recommendation

- Keep telemetry minimal and disclose it to Students.
- Collect only signals necessary for approved policy.
- Treat browser events as review signals, not proof of misconduct.
- Do not automatically fail or terminate based only on tab/paste events.
- Define review, appeal, retention, and staff permissions.
- Do not introduce webcam/biometric/proctoring requirements without explicit approval and privacy review.

## 17. Reporting and analytics

### 17.1 Student view

**[RECOMMENDATION]** Student may see:
- Assessment status
- Attempts used/remaining
- Submission/evaluation status
- Score and pass/fail when released
- Attempt history according to policy
- Feedback and permitted testcase details

### 17.2 Staff view

**[RECOMMENDATION]** Scoped staff report should include:
- Student roster
- Eligible/started/submitted/completed status
- Attempt count
- Final score and pass/fail
- Late/expired/abandoned status where applicable
- Evaluation error state
- Result release state

CSV export is a product decision. Reports must enforce HOD/Manager scope and should not expose hidden testcase secrets.

### 17.3 Course progress

The source does not establish whether Assessment completion affects Course progress, certificates, or other academic progress indicators.

Adopted by ASMT-D30: Assessment completion does not automatically update CourseEnrollment.completedAt, Course progress, certificates or seat usage.

## 18. Notifications

Notification delivery is owned by Module 13. Module 9 defines relevant events.

Potential events:
- Assessment published/assigned
- Assessment start approaching
- Deadline approaching
- Assessment cancelled/rescheduled
- Attempt started (staff event, if needed)
- Submission evaluated
- Results released
- Score adjusted/re-evaluated

Adopted by ASMT-D31: Module 9 emits safe committed events; Module 13 owns recipients, templates, channels, reminders and delivery.

## 19. Edge cases and failure handling

The following cases require explicit target behavior and should be covered by Module 18:

- Student opens Assessment before start time
- Student starts near endsAt
- Student is late to start
- Browser refresh/disconnect during attempt
- Student reconnects after deadline
- Student attempts submission after expiry
- Student has no Problem submissions
- Student submits one Problem but leaves others unanswered
- Multiple Problems are associated with one Assessment
- Problem is archived or modified after publication
- Group membership changes after publish or start
- Student is deactivated during an active attempt
- Judge0 is unavailable or times out
- Evaluation worker fails/retries
- Submission result is delayed past deadline
- Duplicate Submit request/network retry
- Attempt reaches maxAttempts
- Staff grants extra attempt
- Assessment is cancelled/rescheduled after Student starts
- Staff archives Assessment with active attempts
- Result release occurs while evaluation is still pending
- Manual score adjustment/rejudge changes pass/fail
- Anti-cheat signal is generated but not reviewed
- Hidden testcase diagnostic could leak
- Notification delivery fails
- Assessment is published with no Problems or invalid configuration

**[RECOMMENDATION]** Failure states should not silently count as successful grading or silently erase Student work. Preserve state/history and provide recoverable handling where possible.

## 20. Acceptance criteria

These are proposed acceptance criteria; unresolved policy-dependent behavior must be confirmed before implementation.

- [ ] Assessment creation is restricted to authorized academic scope.
- [ ] Draft Assessment is not visible/startable by Students.
- [ ] Assessment can be associated with Problems using the approved target relationship.
- [ ] Publication validates required fields and configuration.
- [ ] Student eligibility is enforced server-side.
- [ ] Assessment start/attempt creation follows the approved attempt policy.
- [ ] Assessment timing is enforced server-side.
- [ ] Browser timer cannot extend the authoritative deadline.
- [ ] Assessment duration is separate from Judge0 CPU/runtime limit.
- [ ] CodeSubmission is associated with the correct Student, Problem, Assessment, and attempt context.
- [ ] Problem evaluation is delegated to Module 7.
- [ ] Assessment score follows the approved aggregation and attempt-selection policy.
- [ ] passingPercentage is applied only to the official score.
- [ ] Pending/system-error evaluation is not silently treated as a final grade.
- [ ] Student result visibility follows the configured release policy.
- [ ] Hidden testcase secrets are never exposed through Student-facing APIs.
- [ ] Re-evaluation/manual adjustment is authorized and traceable if supported.
- [ ] Group/academic scope is enforced for staff and Student access.
- [ ] Historical attempts/submissions/results remain traceable.
- [ ] Notifications use Module 13 event/channel infrastructure.
- [ ] Reporting is scoped by role and academic context.

## 21. QA scenarios for review

| ID | Scenario | Expected behavior to confirm |
|---|---|---|
| AS-01 | Create valid draft | Draft saved within authorized scope |
| AS-02 | Publish without Problems | Publication blocked |
| AS-03 | Student outside target Group opens Assessment | Access denied |
| AS-04 | Student starts before startsAt | Start blocked |
| AS-05 | Student starts within availability | Attempt created according to policy |
| AS-06 | Student refreshes active workspace | Same attempt/deadline restored if resume is approved |
| AS-07 | Student submits after server deadline | Submission rejected or handled by explicit expiry policy |
| AS-08 | Assessment duration expires | Approved lock/auto-submit behavior occurs |
| AS-09 | Judge0/evaluation fails | System error distinguished from student code failure |
| AS-10 | Multiple Problems in one Assessment | Submission context and aggregation correct |
| AS-11 | Student reaches maxAttempts | Additional attempt blocked |
| AS-12 | Student makes multiple code submissions in one attempt | Attempt count follows approved model |
| AS-13 | Problem changes after publish | Published revision policy preserved |
| AS-14 | Partial testcase score | Module 7 score correctly contributes |
| AS-15 | Assessment score equals passing threshold | Pass/fail uses approved boundary condition |
| AS-16 | Multiple attempts with different scores | Official attempt selection correct |
| AS-17 | Result withheld | Student cannot bypass release policy via API |
| AS-18 | Hidden testcase fails | Feedback does not leak hidden input/expected output |
| AS-19 | Authorized staff adjusts score | Reason/history/permission/audit are recorded |
| AS-20 | HOD/Manager requests out-of-scope report | Access denied |
| AS-21 | Group membership changes during active attempt | Approved membership rule is applied |
| AS-22 | Assessment archived | New starts blocked; historical results retained |
| AS-23 | Assessment cancelled after start | Active attempt behavior follows cancellation policy |
| AS-24 | Notification event fails | Assessment state remains correct; notification can recover |

## 22. Adopted product and architecture decisions

All previously open Assessment decisions are resolved in ASMT-D01–ASMT-D36. No product/architecture/security decision remains open in Module 9.

| Area | Adopted v1 target |
|---|---|
| Role/scope | SUPER_ADMIN/HOD manage; Manager drafts within Section only; Finance denied; Student own access |
| Assessment content | CODING only; multi-Problem ordered AssessmentProblem; equal weighting, no optional/duplicate Problems |
| Audience | One College+Department Assessment; optional one Group and explicit Section targets; no individual assignments |
| Lifecycle/versioning | DRAFT/PUBLISHED/ARCHIVED; immutable AssessmentRevision and pinned ProblemVersions |
| Attempt | Full AssessmentAttempt session; created at explicit Start; maxAttempts default 1; resume does not consume another |
| Timer | Server starts timer; deadline=min(start+duration, endsAt); inclusive submission cutoff; no auto-submit unsent editor code |
| Scoring | Per-Problem Module 7 percent; equal average; unanswered=0; infra/pending blocks finalization; round 2 decimals |
| Official score | Highest completed valid attempt; latest attempt breaks ties |
| Result release | AFTER_CLOSE default; IMMEDIATE_SAFE and STAFF_RELEASE optional |
| Integrity | Minimal telemetry off by default; disclosed review signal only; no automatic penalties or proctoring |
| Grading | Manual score override disabled; SUPER_ADMIN-only Module 6/7 rejudge |

Remaining work is engineering: Assessment revisions/attempt schema, Section audience, role/scope enforcement, server timing, Module 6/7 integration, result release and event/audit wiring.

## 23. Adopted technical baseline summary

1. Add AssessmentRevision + AssessmentRevisionProblem and pin every ProblemVersion at publish.
2. Add AssessmentAttempt + AssessmentAttemptProblem to model one timed multi-Problem session per Student attempt.
3. Keep CodeSubmission → SubmissionRun → JudgeRun canonical; Assessment CodeSubmission context=ASSESSMENT/contextId=AssessmentAttempt.id.
4. Enforce server-authoritative deadlines; never map Assessment duration to Judge0 CPU/memory limits.
5. Use Module 7 integer Problem percentage; Assessment applies equal-weight average and separate passingPercentage.
6. Preserve attempt, evaluation and score history; rejudge is not another Student attempt.
7. Enforce HOD/Manager Section/Department scope, Group membership, Student ownership and release policy in backend APIs.
8. Treat integrity signals as disclosed review evidence only.
9. Integrate Modules 13/15/16/17/18 for notifications, audit, analytics, state and recovery.

## 24. Function-level specification and QA companion\n\n- [Function-Level Specifications](./FUNCTION-SPECIFICATIONS.md) — ASMT-F001–ASMT-F022, inputs, permission/scope, preconditions, entry points, main flows, business rules, validation, data behavior, failure paths, acceptance criteria, and architecture gaps.\n- [QA Test Cases](./QA-TEST-CASES.md) — 188 scenarios ASMT-QA-001–ASMT-QA-188.\n\n## 25. Source references

### Conceptual source documents
- Business Flow v2: `docs/source-text/Business Flow v2.md`
- Page-to-DB Mapping v2: `docs/source-text/Page - DB Mapping v2.md`
- Schema reconstruction: `docs/source-text/scheema.md`
- RBAC source: `RBAC (1).pdf` (source document maintained outside the Markdown text extraction)

### Related functional specifications
- Module 6 — Code Submission & Judge0
- Module 7 — Test Case Evaluation
- Module 8 — Labs & Lab Scoring
- Module 10 — Groups (next module)

### Legacy factual reference
- User-provided Assessment forensic report: `Pasted text(1).txt` — completed Assessment code audit answering the earlier clarification questions.

**Source interpretation:** conceptual source documents establish target concepts; forensic report establishes facts about the audited old code. Neither source alone resolves the outstanding business decisions.

---

**Review checkpoint:** ASMT-D01–ASMT-D36 are adopted; 22 function specs and 188 QA target cases are defined. Engineering and QA execution remain pending.

## 26. Adopted target contract and authority

Earlier sections preserve conceptual source evidence and verified legacy audit facts. Where an earlier recommendation/TBD differs, Sections 26–31 and ADOPTED-DECISIONS.md are normative for v1. Legacy defects are not target requirements.

## 27. Assessment audience, roles and lifecycle

### Audience and eligibility
- Assessment belongs to one College and Department. A single optional groupId and zero or more AssessmentSection rows define the audience. With no AssessmentSection rows, the entire active Section audience of the owning Department is eligible; when rows exist, only listed active Sections qualify.
- When groupId is set, active Group membership is an additional eligibility requirement. Group membership does not replace College/Department/Section checks. Individual Student assignment and multi-Group Assessment assignment are out of v1.
- Eligibility requires active Student membership, active College, active included Section and an active same-College/Department Group when assigned. CourseEnrollment is not required because the conceptual Assessment has no courseId.
- Recheck eligibility at list, detail and Start. A membership change after an attempt starts does not invalidate that attempt; the Student cannot start a new attempt after removal. Keep the Student's historical records read-only.
- HOD is limited to assigned College+Department. Manager is limited to assigned College+Department+Section/permitted Group and can only create/update drafts; cannot publish/archive/delete. Finance has no Assessment access.

### Lifecycle
- AssessmentType is CODING only. Create in DRAFT; publish only after required fields, Problem set, audience, schedule and ProblemVersion readiness pass. PUBLISHED is schedule- and eligibility-gated. ARCHIVED is history-only and blocks new starts/submissions.
- SUPER_ADMIN/HOD control publish/archive/restore; restore returns to DRAFT. No hard delete and no separate CANCELLED/EXPIRED persisted status in v1.
- Every publish creates an immutable AssessmentRevision snapshot. Any change to Problems/order, ProblemVersion, audience, timing, duration, attempts, threshold, integrity or result policy requires a new draft revision and publish. Existing attempts retain the revision they started with.

### Problem set
- Use multi-Problem AssessmentProblem semantics. The published revision contains one or more Problems; all are required. Duplicates are rejected. orderIndex is zero-based, unique and contiguous. No randomized ordering, Problem weights, optional Problems or per-Problem points in v1.
- Each revision Problem pins a published Module 5 ProblemVersion from the same College. An archived/newer Problem version does not alter a previously published Assessment revision.

## 28. Attempt, timing and submission contract

### Session attempt
- One AssessmentAttempt represents one Student's entire multi-Problem sitting. Attempt is created transactionally on successful explicit Start; opening details/workspace does not consume an attempt. maxAttempts defaults to 1 and may be configured as a positive integer per revision.
- One active attempt per Student+Assessment. Refresh/reconnect resumes the same AssessmentAttempt and deadline; it never consumes a new attempt. Client-supplied attempt number/counter/time is never authoritative.
- Each attempt contains one AssessmentAttemptProblem per revision Problem. A Student can submit a Problem multiple times during the active attempt, subject to Module 6 Idempotency-Key/rate limits. All CodeSubmissions remain in history.
- For a Problem, select the highest complete Module 7 percentage among its submissions; latest accepted submission breaks ties. Rejudge does not create an AssessmentAttempt or change attempt count.

### Server-authoritative timing
- startsAt and endsAt are optional UTC timestamps. Null startsAt means startable immediately after publish; null endsAt means no global close. If both exist, startsAt must precede endsAt.
- Explicit Start is accepted at/after startsAt and strictly before endsAt. Server records startedAt and deadlineAt=min(startedAt+durationMinutes, endsAt when present). Submission acceptance is inclusive at deadlineAt.
- Timer display uses server-derived remaining time; browser/localStorage countdown cannot extend a session. Pause and individual time extensions are not supported in v1.
- At Finish/deadline, lock the attempt. Do not auto-submit unsent browser/editor code. Retain the last accepted CodeSubmission per Problem; an unanswered Problem contributes 0. Accepted submissions awaiting Module 7 remain pending; infrastructure failures are RECOVERY_REQUIRED, not 0.
- Assessment duration is never passed to Judge0 as CPU/memory limits. ProblemVersion's Module 5/6 limits apply to each execution.

## 29. Assessment scoring, result release and integrity

### Score and pass
- Module 7 owns per-Problem testcase execution/verdict and returns an integer evaluation percentage. Module 9 selects the best completed submission per Problem within the attempt.
- Attempt score is the equal arithmetic mean of selected percentages for every Problem in the revision, with unanswered Problems treated as 0 after Finish/deadline. Round half-up once to two decimals.
- A pending/incomplete/unknown/infrastructure evaluation for an accepted submission blocks final attempt finalization until recovered/reconciled; it is not converted to 0. A completed 0% code result is valid and contributes 0.
- passingPercentage is decimal 0.00–100.00, default 50.00. A finalized attempt passes when scorePercent >= passingPercentage; equality passes.
- Official Assessment score is the highest finalized valid attempt; the latest started attempt breaks ties. Preserve all attempts; do not average attempts.

### Result release
- resultReleasePolicy values: AFTER_CLOSE (default), IMMEDIATE_SAFE, STAFF_RELEASE.
- Student always sees attempt receipt and processing state. AFTER_CLOSE exposes final score/pass only once endsAt has passed and all evaluations are final; where endsAt is null, HOD/SUPER_ADMIN must release explicitly. STAFF_RELEASE requires explicit HOD/SUPER_ADMIN action. IMMEDIATE_SAFE exposes only the Student's own safe finalized result after evaluation.
- Hidden testcase inputs/outputs/status/count/order/diagnostics and raw Judge0 data remain protected under Module 7. HOD/Manager views are scoped summaries only; source code/raw JudgeRun is SUPER_ADMIN privileged/audited.

### Integrity
- Integrity signals are OFF by default. If enabled in a published revision, display notice before Start and collect only tab/window blur and paste counts/timestamps.
- Signals are review-only evidence. They do not automatically fail, terminate, reduce score or consume additional attempts. No webcam, biometric, fullscreen-lock or proctoring requirement in v1.

## 30. Staff review, history, reporting and cross-module boundaries

| Role | Access |
|---|---|
| SUPER_ADMIN | Global Assessment manage, result release, privileged rejudge through Modules 6/7 and audited raw inspection |
| HOD | Assessment create/view/update/publish/archive/restore in assigned College+Department; scoped roster/result summary |
| MANAGER | View/create/update drafts in assigned Section/permitted Group; scoped roster/summary only; no publish/archive/delete |
| FINANCE_TEAM | No Assessment access |
| STUDENT | Eligible Assessment view/start/submit and own results when released |

- Manual score override is disabled in v1. Staff View/CRUD does not imply score adjustment or source code access.
- Reports are read-only and scoped; CSV export is disabled unless a future explicit export permission is approved. No cross-College or foreign Section filter can widen scope.
- Assessment completion does not update CourseEnrollment.completedAt, course progress, certificates or subscription seats. Module 4 owns Course completion; Module 16 owns analytics.
- Module 13 owns notification recipients, templates, channel and delivery. Module 15 owns canonical audit. Module 17 owns state transitions. Module 18 owns retries/reconciliation. Use durable outbox and dedupe eventId/runId.

### Cross-module contract
| Module | Contract |
|---|---|
| 4 — Course & Enrollment | Provides active College/Department/Section/Student membership status and Section audience events; Assessment does not create CourseEnrollment |
| 5 — Problem Management | Provides immutable published ProblemVersion, language and resource policy |
| 6 — Code Submission & Judge0 | Creates CodeSubmission with context=ASSESSMENT/contextId=AssessmentAttempt.id; owns SubmissionRun/JudgeRun and idempotency |
| 7 — Test Case Evaluation | Finalizes Problem percentage/verdict, hidden-data handling and rejudge result events |
| 8 — Labs & Lab Scoring | Separate activity owner; no Lab score/late policy reused for Assessment |
| 10 — Groups & Membership | Owns Group and GroupMember lifecycle; Assessment checks canonical membership |
| 11 — Contests | Contest-only penalty, rank and tie-break rules do not apply to Assessment |
| 13 — Notifications | Owns delivery and recipients for safe committed Assessment events |
| 15 — Audit Logging | Owns canonical audit; records lifecycle, release, integrity review and privileged changes |
| 16 — Student Analytics | Consumes approved assessment analytics; no independent score authority |
| 17/18 — State and Failure | Own canonical transition/recovery/retry semantics |

## 31. Target schema delta and release gates

### Target schema additions
- AssessmentRevision: immutable revision/version, Assessment ID, title/description/type, College/Department/Group/Section audience, duration, startsAt/endsAt, maxAttempts, passingPercentage, resultReleasePolicy, integritySignalsEnabled, status/publishedAt.
- AssessmentRevisionProblem: revisionId, AssessmentProblem/source relation ID, problemId, problemVersionId, zero-based orderIndex; unique revision+Problem and revision+orderIndex.
- AssessmentSection: assessment/revision ID + sectionId; explicit Section audience; validate same College/Department.
- AssessmentAttempt: assessmentId, revisionId, studentId, attemptNumber, status, startedAt, deadlineAt, submittedAt, completedAt, scorePercent, passed, createdAt/updatedAt; unique assessment+student+attemptNumber and one-active-attempt constraint.
- AssessmentAttemptProblem: attemptId, revisionProblemId, status, bestCodeSubmissionId, evaluationPercent, finalizedAt; unique attempt+revisionProblem.
- AssessmentScoreHistory and AssessmentIntegritySignal: append-only score/release/rejudge provenance and disclosed integrity review signals. No raw source or hidden testcase payload in these records.
- Extend Assessment with revision pointer/published state as needed; retain conceptual Assessment and AssessmentProblem identity and fields.

### Release readiness
- [x] 22 function specifications retained (ASMT-F001–ASMT-F022).
- [x] 36 product/architecture/security decisions adopted (ASMT-D01–ASMT-D36).
- [x] 188 target QA cases defined (ASMT-QA-001–ASMT-QA-188).
- [ ] Schema migrations/indexes and API/UI implemented.
- [ ] Modules 4–8 and 10 integrations implemented.
- [ ] Module 13/15/16/17/18 contracts implemented.
- [ ] Execute QA, migration, security, concurrency, load and outbox recovery tests.