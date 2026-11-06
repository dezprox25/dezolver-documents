# 9. Assessments

**Document type:** Functional Specification  
**Version:** 0.2 — Detailed draft aligned to Functional Specification Standard v1.0  
**Status:** Detailed source-grounded draft — product/architecture/security decisions remain open  
**Module owner:** DEZOLVER Academic Activities / Assessments  
**Related modules:** College / Department / Section Management, Course & Enrollment, Problem Management, Code Submission & Judge0, Test Case Evaluation, Labs & Lab Scoring, Groups, Contests, Notifications, Audit Logging, Student Analytics, Functional State Machines, Edge Cases & Failure Handling

> **Interpretation rule:** This specification separates approved conceptual source documents, verified legacy-code audit facts, and proposed target behavior. Legacy behavior is evidence about the old implementation—not an automatic requirement for DEZOLVER. Recommendations are not approved product rules until the product owner confirms them.

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

The RBAC source does not fully specify Assessment action permissions. Create, edit, publish, assign, archive, review, score adjustment, and report permissions remain to be mapped.

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

The following is a proposed action-level mapping based on the general RBAC scope. It requires confirmation against the target RBAC matrix.

| Role | Proposed Assessment access |
|---|---|
| SUPER_ADMIN | Global administrative access, subject to audit |
| HOD | Manage/review Assessments within authorized College/Department scope |
| MANAGER | Manage/review Assessments within authorized College/Department/Section/Group scope |
| STUDENT | View assigned Assessments, start eligible attempts, submit code, and view permitted results |
| FINANCE_TEAM | No academic Assessment access by default |

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

**[RECOMMENDATION]** Use the conceptual multi-Problem association as the target direction, subject to product confirmation. Do not impose the legacy one-Problem limitation by default.

**Open decision:** Should Problem content be frozen at publish time or at each Student's start? See D7.

## 8. Assignment and eligibility

### 8.1 Conceptual scope

The conceptual Assessment includes collegeId, departmentId, and groupId. The source does not definitively establish individual Student assignment, multi-Group assignment, or how Group membership changes affect eligibility.

### 8.2 Assignment options requiring decision

- One Assessment assigned to one Group
- One Assessment assigned to multiple Groups
- Group assignment plus individual Student exceptions
- Individual Student assignment
- Department/Section-wide assignment

**[RECOMMENDATION]** Start with Group assignment if consistent with the Group module; decide multi-Group assignment before schema design. Avoid creating per-Group copies if one shared Assessment identity and results/reporting are required.

### 8.3 Membership changes

**[RECOMMENDATION]** Define eligibility at a stable checkpoint, preferably when the Student starts an attempt:
- A newly eligible Student may start if the Assessment remains available.
- An already-started attempt should not be unexpectedly invalidated solely by a membership change.
- Removed Students should not start new attempts.
- Historical attempts/results remain retained.

This is not an approved policy; see D2.

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

**[RECOMMENDATION]** Model cancellation/closure separately from archival if cancellation is a real use case. Avoid deleting Assessment history.

### 9.3 Editing after publication

**[OPEN DECISION]** Choose:
- Allow all published fields to be edited
- Lock after first attempt starts
- Published Assessment immutable; clone to change
- Permit limited metadata/date edits while locking Problem/scoring content

**[RECOMMENDATION]** Lock evaluated content after the first Student starts; permit carefully controlled metadata changes. Preserve an explicit revision/history.

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

**Recommended policy to consider:** Student's deadline is the earlier of start + duration and Assessment endsAt. This recommendation requires approval.

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

This recommendation requires product approval and schema design.

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

**[RECOMMENDATION]** Use equal-weight normalized percentage for the initial release unless unequal Problem weights are a required business use case. If weights are needed, add weight to the AssessmentProblem association explicitly.

### 13.4 Passing threshold

The conceptual model includes passingPercentage. Legacy code did not enforce an Assessment-level pass threshold.

**[RECOMMENDATION]**
- Assessment score is normalized to 0–100.
- Pass when final score >= passingPercentage.
- Keep passingPercentage separate from Problem-level evaluation.
- Define behavior for unanswered Problems, incomplete evaluation, and zero-weight/invalid configuration.
- Do not finalize pass/fail until the official attempt score is selected.

### 13.5 Final score across attempts

**[OPEN DECISION]** Highest, latest, first, average, or another attempt score?

**[RECOMMENDATION]** Highest completed valid attempt as the official score, while preserving every attempt. This is a proposed policy, not a source-confirmed rule.

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

These are recommendations pending decision.

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

**[OPEN DECISION]** Define whether Assessment completion/score contributes to course completion, progress, certificate eligibility, or gradebook. Do not infer from CodeSubmission existence.

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

**[RECOMMENDATION]** Support publication/assignment and result-release notifications in the initial Assessment workflow. Reminder timing and channels follow Module 13.

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

## 22. Genuine product decisions

The detailed options, decision log, and function-level specifications are maintained in [Function-Level Specifications](./FUNCTION-SPECIFICATIONS.md). Main decisions required before target implementation are:

1. Assignment scope: one Group, multiple Groups, or Group plus individual assignment.
2. Group membership snapshot/live eligibility and behavior for active attempts.
3. Assessment-specific role/action permissions.
4. Lifecycle transitions and editing after publication.
5. Availability window vs per-student duration; late start and expiry.
6. Attempt model: full Assessment session vs each submission; when attempt is consumed.
7. Problem content/testcase versioning: freeze on publish or on start.
8. Problem order/navigation/skip/revisit rules.
9. Assessment Problem weighting and score aggregation.
10. Official score across attempts and passing threshold semantics.
11. Result feedback/release policy.
12. Manual score adjustment and re-evaluation.
13. Integrity telemetry and consequences.
14. Student My Assessments dashboard and staff reports/exports.
15. Assessment notification events.

**Do not treat the recommendations in this draft or checklist as approved choices.** The user’s answers should be recorded before the spec is marked implementation-ready.

## 23. Technical recommendations summary

1. Prefer a parent AssessmentAttempt representing a full Assessment session if the product supports multiple Problems per Assessment.
2. Keep Problem-level submissions under the attempt; do not count every Submit click as a new Assessment attempt.
3. Use server-authoritative start/deadline validation; client timer is display only.
4. Keep Assessment duration separate from Judge0 execution limits.
5. Freeze the Problem/testcase revision for published assessments or each attempt to ensure grading reproducibility.
6. Reuse Module 7 evaluation and scoring; Assessment owns aggregation and pass/fail policy only.
7. Preserve submission/evaluation/score history; manual adjustments must be auditable.
8. Apply backend authorization for Student eligibility, staff scope, result release, and report access.
9. Treat integrity signals as review evidence, not automatic proof or disqualification.
10. Resolve architecture decisions D1, D5, D6, D7, D9, and D10 before finalizing schema/API contracts.

## 24. Function-level specification and QA companion\n\n- [Function-Level Specifications](./FUNCTION-SPECIFICATIONS.md) — ASMT-F001–ASMT-F022, inputs, permission/scope, preconditions, entry points, main flows, business rules, validation, data behavior, failure paths, acceptance criteria, and architecture gaps.\n- [QA Test Cases](./QA-TEST-CASES.md) — 158 scenarios ASMT-QA-001–ASMT-QA-158.\n\n## 25. Source references

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

**Review checkpoint:** Module 9 has been expanded with detailed function specifications and QA coverage. Product decisions remain open. Continue to the next module according to the user's all-modules-first workflow; do not pause to resolve the decision log at this stage.
