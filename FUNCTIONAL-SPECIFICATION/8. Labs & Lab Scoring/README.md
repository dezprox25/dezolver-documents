# 8. Labs & Lab Scoring

**Document type:** Functional Specification  
**Version:** 1.0 — Adopted target baseline  
**Status:** Product/architecture/security decisions adopted (LAB-D01–LAB-D38); engineering and QA execution pending  
**Module owner:** DEZOLVER Academic Activities / Labs  
**Related modules:** Course & Enrollment, Problem Management, Code Submission & Judge0, Test Case Evaluation, Groups, Assessments, Notifications, Audit Logging

> **Adopted target baseline:** Source findings and legacy audit observations remain traceable. All 38 Module 8 decisions are now adopted. Sections 37–42 and ADOPTED-DECISIONS.md are normative and supersede earlier recommendation/TBD annotations. Engineering implementation and QA execution remain pending.

---

## 1. Purpose

Define how DEZOLVER creates, assigns, manages, submits, evaluates, and scores Labs.

A Lab is an academic activity associated with a course and one or more programming Problems. Students complete assigned Labs by submitting code. The system evaluates code using the existing submission and evaluation infrastructure and converts the evaluation result into a Lab score.

This module covers Lab creation/configuration, assignment, Student visibility, submission attempts, evaluation integration, score conversion, due dates, late submissions, completion/progress, feedback, score adjustment, permissions, validation, and audit requirements.

## 2. Source evidence and implementation status

### 2.1 Business-flow evidence

**[SOURCE—BUSINESS]** Business Flow v2 describes a flow broadly equivalent to:

**Group → Lab / Problem → Student Submission → DEZOLVER Backend → Judge0 → Evaluation → Lab Score**

This establishes Labs as an academic activity involving Groups, Problems, submissions, evaluation, and scoring. It does not definitively establish whether submissions are individual or collaborative.

### 2.2 Conceptual schema evidence

**[SOURCE—SCHEMA]** The conceptual schema describes:

**LabAssignment**
- `id`
- `courseId`
- `problemId`
- `title`
- `description`
- `scope` (`ContentScope`)
- `maxScore`
- `dueDate`
- `createdAt`
- `updatedAt`

**LabSubmission**
- `id`
- `labAssignmentId`
- `studentId`
- `codeSubmissionId`
- `score`
- `feedback`
- `submittedAt`

**GroupLab**
- `groupId`
- `labAssignmentId`
- `createdAt`

These are conceptual source fields, not confirmed target-release database schema.

### 2.3 Page-to-DB Mapping evidence

**[SOURCE—BUSINESS]**
- HOD/Manager workflows include Courses & Labs.
- Student workflows include My Labs.
- Lab-related data is associated with courses, problems, groups, and submissions.

### 2.4 Legacy-code audit findings

**[AUDIT—LEGACY]** The user-provided forensic report identifies the following in the audited older code snapshot:

- No dedicated `Lab`, `LabAssignment`, or `LabSubmission` model.
- No `GroupLab` association.
- No dedicated Groups or Sections model.
- No dedicated My Labs route or UI.
- Academic hierarchy was Tenant → Cohort.
- Existing coursework was represented through Rooms and CourseRoom associations.
- Assessment submissions were individual and associated with a `userId`.
- Code execution was handled asynchronously through BullMQ, Judge0, and the evaluation worker.
- Weighted testcase scoring was supported by the underlying evaluation infrastructure.

**Conclusion:** The dedicated Labs subsystem is absent from the audited legacy snapshot. This specification defines the adopted target product baseline; it does not claim that the target subsystem has already been implemented.

## 3. Proposed design principles

The following are **recommendations for review**, not confirmed requirements.

1. Labs should be a distinct academic activity/domain rather than implicitly represented as Rooms or Assessments.
2. Labs should reuse existing Problem, CodeSubmission, Judge0, and Test Case Evaluation capabilities.
3. Lab assignment and Lab grading should be separate from raw code execution.
4. A Lab may be assigned to a Group, but the recommended submission model is individual.
5. Lab score should be calculated from the evaluation percentage and configured maximum score.
6. Submission attempts should be retained as history.
7. Staff score changes should be permission-controlled and auditable.
8. Lab policy should not silently inherit unrelated Assessment or Contest rules.

## 4. Scope

### 4.1 Included
- Lab assignment creation and editing
- Course association and Problem association
- Assignment to eligible Students or Groups
- Publication and availability
- Student My Labs view
- Submission attempts
- Judge0 evaluation integration
- Score conversion
- Attempt-selection policy
- Due dates and late-submission handling
- Lab status and progress
- Staff feedback and manual score adjustment
- Permissions, validation, and audit requirements

### 4.2 Excluded or owned by other modules

| Area | Responsible module |
|---|---|
| Course creation and Student enrollment | Module 4 — Course & Enrollment |
| Problem statement and test cases | Module 5 — Problem Management |
| Code submission intake and Judge0 | Module 6 — Code Submission & Judge0 |
| Testcase scoring and verdict | Module 7 — Test Case Evaluation |
| Group creation and membership | Module 10 — Groups |
| Assessments and assessment attempts | Module 9 — Assessments |
| Contest-specific scoring and ranking | Module 11 — Contests |
| Notifications | Module 13 — Notifications |
| Audit trail | Module 15 — Audit Logging |
| Canonical lifecycle rules | Module 17 — Functional State Machines |
| Cross-module failures and recovery | Module 18 — Edge Cases & Failure Handling |

## 5. Actors and permissions

The role assignments below are reconciled to the supplied RBAC and Module 4 adopted access model. The more restrictive Module 8 decisions apply where source matrices conflict.

| Actor | Proposed capability |
|---|---|
| SUPER_ADMIN | Platform-level administration and authorized troubleshooting |
| HOD | Manage or oversee Labs within authorized College/Department scope |
| MANAGER | Manage Labs within authorized College/Department/Section or Group scope |
| Faculty / Lab creator | Create and manage Labs within assigned academic scope, if this role exists in target RBAC |
| STUDENT | View assigned Labs, submit code, and view permitted results |
| FINANCE_TEAM | No Lab access by default unless another requirement establishes it |

### 5.1 Access rules

**[RECOMMENDATION]**
- Students may view only Labs assigned to their Course, Section, Group, or individual account.
- Students may view only their own submission details and score.
- Staff access must be limited to their authorized academic scope.
- Staff must not assign Labs outside their permitted scope.
- Manual score adjustment requires an approved grading permission.
- Hidden testcase and raw Judge0 access follows Module 7’s confidentiality policy.

**Resolved by LAB-D37:** no separate Faculty/Lab Staff role is introduced in v1; use HOD and scoped MANAGER.

## 6. Lab data model

### 6.1 LabAssignment

The conceptual source model contains:

| Field | Meaning |
|---|---|
| `id` | Unique Lab identifier |
| `courseId` | Associated Course |
| `problemId` | Associated Problem |
| `title` | Lab title |
| `description` | Lab instructions |
| `scope` | Source-defined `ContentScope` |
| `maxScore` | Maximum Lab score |
| `dueDate` | Due date/time |
| `createdAt` | Creation timestamp |
| `updatedAt` | Last update timestamp |

**[RECOMMENDATION]** The target implementation may need additional fields such as publication status, creator/last editor, published timestamp, availability start time, late policy, attempt policy, assignment configuration/version, and score-override policy. These are proposed additions, not confirmed schema fields.

### 6.2 LabSubmission

| Field | Meaning |
|---|---|
| `id` | Unique Lab submission record |
| `labAssignmentId` | Lab being submitted |
| `studentId` | Student submitting |
| `codeSubmissionId` | Associated code submission |
| `score` | Lab score |
| `feedback` | Feedback associated with submission |
| `submittedAt` | Submission timestamp |

**[RECOMMENDATION]** Each submission should support attempt identification and grading provenance, directly or through related records.

### 6.3 GroupLab association

**[SOURCE—SCHEMA]** The conceptual schema describes a GroupLab association between a Group and a LabAssignment.

**[RECOMMENDATION]** Use this association to determine eligibility. The Group should not automatically become the submitter or score owner.

**Resolved by LAB-D05/LAB-D24:** retain GroupLab as the optional many-to-many association; Module 10 owns GroupLab writes.

## 7. Lab creation and assignment

### 7.1 Create Lab

**[RECOMMENDATION]** Authorized staff should be able to:

1. Select a Course.
2. Select a Problem.
3. Enter Lab title and description.
4. Set maximum score.
5. Configure due date and availability.
6. Select target Group(s), Section(s), or eligible Student population.
7. Configure attempts and late-submission rules.
8. Save as draft or publish.

### 7.2 Validation

A Lab should not be published unless:
- Title is present.
- Course is valid and within creator’s authorized scope.
- Problem is valid and available for assignment.
- Maximum score is greater than zero.
- Assignment target is valid.
- Due-date configuration is valid, if configured.
- Required policy settings are valid.

**Resolved by LAB-D03:** v1 supports exactly one Problem per LabAssignment, matching the conceptual `problemId` field.

### 7.3 Assignment eligibility

**[RECOMMENDATION]** A Student is eligible when:
- They are enrolled in the associated Course.
- They belong to an assigned Group or Section, if applicable.
- Their account is active and permitted to access the Course.
- The Lab is published and available to them.

The relationship between Course enrollment, Group membership, and Lab assignment must be defined by Modules 4 and 10.

## 8. Lab lifecycle and availability

### 8.1 Proposed lifecycle

This is a proposed lifecycle, not an audited implementation:

`Draft → Published → Closed → Archived`

| Status | Proposed meaning |
|---|---|
| Draft | Editable by authorized staff; not visible to Students |
| Published | Visible to eligible Students and available according to configured dates |
| Closed | No new submissions accepted under the configured closure policy |
| Archived | Historical Lab retained for reference; not available for normal editing or submission |

### 8.2 Availability rules

**[RECOMMENDATION]**
- Students cannot submit before a configured start time.
- Students cannot submit after closure unless the late policy permits it.
- Staff can inspect a Lab after it closes.
- Publishing or closing a Lab does not delete submissions or scores.

**Resolved by LAB-D09/LAB-D29:** dueDate is an inclusive UTC hard cutoff by default; late acceptance without penalty is an explicit Lab policy option.

## 9. Student Lab workflow

### 9.1 My Labs

The Student My Labs view should list Labs they are eligible to access.

**[RECOMMENDATION]** Display Lab title, Course, Problem title, due date, maximum score, Lab status, submission status, latest score, best score (if applicable), and late indicator (if applicable).

### 9.2 Lab details

Students should be able to view:
- Lab instructions and Problem statement
- Allowed programming languages
- Due date and availability
- Maximum score and attempt policy
- Submission history
- Current score and feedback when available

Hidden testcase input and expected output remain protected under Module 7.

### 9.3 Submit code

**[RECOMMENDATION]**
1. Student opens an eligible Lab.
2. Student selects a supported language.
3. Student submits source code.
4. DEZOLVER creates a CodeSubmission associated with Lab context.
5. Submission is evaluated through Modules 6 and 7.
6. Evaluation percentage is converted into Lab score.
7. Lab submission record is updated.
8. Student sees permitted evaluation details and Lab score.

The Lab module should not implement a second Judge0 execution engine.

## 10. Individual vs Group submission

### 10.1 Source interpretation

The Business Flow v2 includes Groups in the Lab workflow, and the conceptual schema contains a GroupLab association. However, the audited legacy code supports individual assessment submissions and does not establish group-submission behavior for Labs.

### 10.2 Proposed recommendation

**Recommended default: Group assignment, individual submission and grading.**

- A Lab may be assigned to one or more Groups.
- Each eligible Student in an assigned Group can access the Lab.
- Each Student submits their own code.
- Each Student receives an individual score.
- One Student’s submission does not automatically count for another Student.
- Group membership determines eligibility, not score ownership.

### 10.3 Open decision

Confirm whether DEZOLVER also needs **collaborative Group submissions**, where one shared submission or score is associated with multiple Students. The GroupLab association alone does not establish this behavior.

## 11. Lab scoring

### 11.1 Evaluation score

**[AUDIT—LEGACY]** The supplied forensic report describes testcase evaluation as weighted percentage:

`Evaluation Score = round(100 × passed testcase weight / total testcase weight)`

Evaluation and testcase scoring rules are owned by Module 7. This is a legacy-code audit finding, not a confirmed target-release product rule.

### 11.2 Proposed Lab score conversion

**[RECOMMENDATION]**

`Lab Score = (Evaluation Score / 100) × Lab Max Score`

Example:
- Evaluation score: 80%
- Lab maximum score: 20
- Lab score: 16 out of 20

This allows different Labs to use different maximum scores without changing testcase evaluation.

### 11.3 Rounding

**Resolved by LAB-D08:**
- Whether Lab scores may contain decimals.
- Decimal precision (whole numbers, two decimal places, etc.).
- Whether rounding occurs during conversion or only at display/report time.
- Whether rounding follows Module 7.

**Recommendation:** Preserve sufficient internal precision and apply one approved rounding rule when the score is finalized.

### 11.4 Score range and validation

**[RECOMMENDATION]**
- Lab score cannot be negative.
- Lab score cannot exceed maximum score unless an approved bonus policy exists.
- Evaluation failure cannot produce a score greater than the evaluation result.
- Missing/incomplete evaluation is not silently treated as a final grade.
- Score calculation must be reproducible from evaluation result and Lab configuration.

## 12. Multiple attempts and final score

### 12.1 Legacy finding

**[AUDIT—LEGACY]** The forensic report identifies append-only assessment submissions in the old code, but no Lab-specific attempt policy or final Lab grade selector.

The old code does not establish whether highest, latest, first, or first-accepted submission determines the Lab score.

### 12.2 Proposed recommendation

**Recommended default: Multiple attempts, highest valid Lab score.**

- Each submission is retained as an attempt.
- New attempts do not erase previous attempts.
- Final score is the highest valid score among eligible attempts.
- Student may see attempt history and best score, subject to visibility policy.
- Re-evaluation of an attempt does not automatically create a new Student attempt.

### 12.3 Attempt policy

**Resolved by LAB-D18:** attempts are unlimited by default; an optional positive maximum may be configured per published Lab revision.

**[RECOMMENDATION]** If configurable attempts are supported, use highest score as the default unless a Lab explicitly selects another approved policy.

## 13. Due dates and late submissions

### 13.1 Source evidence

The conceptual LabAssignment schema contains `dueDate`. The source material does not define late-submission allowance, grace periods, penalties, hard cutoff behavior, or timezone handling.

### 13.2 Proposed recommendation

Make late-submission policy configurable per Lab.

Recommended initial default:
- Due date is displayed in the Student’s applicable local time.
- Late submissions are **not penalized unless a late policy is explicitly configured**.
- A Lab may permit or prohibit submissions after due date.
- If late submissions are permitted, the Lab records whether each submission is late.
- Staff can distinguish on-time and late attempts.

### 13.3 Possible late policies

These are options, not existing requirements.

| Policy | Meaning |
|---|---|
| No late submissions | Submission blocked after deadline |
| Grace period | Late submission allowed for configured period without penalty |
| Flat penalty | Fixed points deducted for late submission |
| Percentage penalty | Percentage deducted from earned score |
| Penalty per interval | Deduction per hour/day late |
| Late but accepted | Submission accepted with no score penalty |

### 13.4 Open decisions

- Is the deadline inclusive?
- Is a grace period required?
- Is late penalty calculated from earned score or Lab maximum?
- Can late penalty reduce score to zero?
- Can staff grant individual extensions?
- What timezone is authoritative?
- Does rejudge preserve original submission timestamp?

## 14. Feedback and manual score adjustment

### 14.1 Legacy finding

**[AUDIT—LEGACY]** The forensic report identifies no dedicated manual Lab score override or pedagogical feedback workflow in the old code.

Anti-cheat review decisions are not grading overrides and must not be treated as such.

### 14.2 Proposed recommendation

- Automated score is the default.
- Authorized staff may provide feedback.
- Staff may adjust score only if score-override permission is approved.
- Score adjustment requires a reason.
- Original automated score is preserved.
- Final score identifies whether it was automated or manually adjusted.

### 14.3 Proposed score-adjustment record

Retain, directly or through audit history:
- Lab submission
- Original automated score
- Adjusted score
- Staff member who made change
- Reason
- Timestamp
- Feedback
- Previous and new score

### 14.4 Open decisions

- Is manual score adjustment required in first release?
- Which roles may adjust scores?
- Is second-person approval required?
- Can staff adjust above Lab maximum?
- Can an adjustment be reverted?
- Should adjustment notify the Student?

## 15. Lab completion and progress

**[SOURCE—BUSINESS]** Page-to-DB Mapping identifies a Student My Labs workflow. It does not establish a formal Lab completion rule.

### 15.1 Proposed recommendation

Track these concepts separately:
- Not started
- Submitted
- Evaluation pending
- Evaluated
- Completed
- Late
- Closed without submission

These are proposed display/progress concepts, not approved persisted status values.

### 15.2 Completion criteria

**Resolved by LAB-D17/LAB-D33:** Lab completion is derived when an accepted attempt has a complete Module 7 evaluation; no minimum score threshold is required.

**Recommendation:** Use a configurable completion threshold if completion requires more than submission, while keeping submission and completion status distinct.

## 16. Notifications

Notifications are owned by Module 13.

Potential Lab events:
- Lab published
- Lab assignment changed
- Lab due date approaching
- Lab due date changed
- Lab closed
- Submission received
- Submission evaluated
- Score adjusted
- Feedback added
- Lab reopened

**Resolved by LAB-D24/LAB-D35:** Module 8 emits safe committed Lab events through the outbox; Module 13 owns recipient, channel, reminder and delivery configuration.

## 17. Edge cases and failure handling

These cases should be addressed in Modules 17 and 18 and tested as part of Lab implementation.

| Scenario | Expected behavior to define |
|---|---|
| Student removed from assigned Group | Whether access is revoked immediately or historical submissions remain accessible |
| Student joins Group after Lab publication | Whether Lab assignment is inherited |
| Group membership changes before deadline | Eligibility rules for existing/new members |
| Lab unpublished after submission | Existing submission and score remain accessible according to policy |
| Lab due date changes | Whether previous submissions or late status change |
| Problem edited after Lab publication | Whether Lab uses original Problem version |
| Problem deleted or archived | Lab behavior and historical display |
| Evaluation finishes after deadline | Submission timestamp determines on-time status, not grading completion |
| Judge0 fails | Submission remains recoverable; not treated as Student failure |
| Duplicate submission request | Prevent unintended duplicate attempts |
| Staff changes max score after submissions | Whether existing scores are recalculated |
| Staff overrides score | Preserve automated score and audit change |
| Rejudge changes evaluation score | Recalculate Lab score according to approved policy |
| Student loses course enrollment | Access to historical Lab and scores |
| Lab archived | Retain history without normal submissions |

## 18. Acceptance criteria

These are proposed criteria for review.

- [ ] Authorized staff can create a Lab associated with Course and Problem.
- [ ] Lab can be assigned to eligible Students or Groups.
- [ ] Students can view assigned Labs in My Labs.
- [ ] Students cannot access Labs outside authorized academic scope.
- [ ] Lab publication and availability are enforced server-side.
- [ ] Student submissions link to the correct Lab, Student, and CodeSubmission.
- [ ] Code execution reuses Modules 6 and 7.
- [ ] Lab score follows approved evaluation-to-Lab conversion.
- [ ] Maximum score and rounding rules are enforced.
- [ ] Attempt history is retained.
- [ ] Final score follows approved attempt-selection policy.
- [ ] Due-date and late-submission rules are enforced.
- [ ] Students view only permitted submission details and scores.
- [ ] Staff inspect results only within authorized academic scope.
- [ ] Feedback and manual adjustments are permission-controlled if enabled.
- [ ] Score adjustments preserve original score and audit information.
- [ ] Lab progress and completion follow approved rules.
- [ ] Evaluation failures are distinguishable from Student code failures.
- [ ] Lab events integrate with Notifications and Audit Logging where approved.

## 19. QA scenarios

| ID | Scenario | Expected behavior |
|---|---|---|
| LAB-01 | Staff creates valid Lab | Lab saved with correct Course and Problem |
| LAB-02 | Staff publishes without Problem | Publication blocked |
| LAB-03 | Student assigned through Group | Student can see Lab |
| LAB-04 | Student outside assigned Group opens Lab | Access denied |
| LAB-05 | Student submits valid code | CodeSubmission linked to Lab and Student |
| LAB-06 | Submission accepted by Judge0 | Lab score follows approved conversion |
| LAB-07 | Submission receives partial testcase score | Lab score reflects approved partial-credit policy |
| LAB-08 | Student submits multiple attempts | Attempt history retained |
| LAB-09 | Attempts have different scores | Final score follows approved attempt policy |
| LAB-10 | Student submits before deadline | Submission classified as on time |
| LAB-11 | Student submits after deadline | Late policy enforced |
| LAB-12 | Evaluation finishes after deadline | Late status based on submission time |
| LAB-13 | Judge0 infrastructure failure | Not treated as Student failure |
| LAB-14 | Staff adjusts score | Permission, reason, and score history recorded |
| LAB-15 | Staff outside academic scope accesses Lab | Access denied |
| LAB-16 | Lab maximum score changes after grading | Existing scores follow approved recalculation policy |
| LAB-17 | Problem changes after Lab publication | Lab follows approved Problem version policy |
| LAB-18 | Student removed from Group | Access and historical score behavior follow approved rule |
| LAB-19 | Lab closes while submissions are in progress | Behavior follows approved closure policy |
| LAB-20 | Student accesses another Student’s submission API | Access denied |

## 20. Adopted Module 8 product defaults

All product, architecture and security questions from the original draft are resolved by LAB-D01–LAB-D38. The complete register is in ADOPTED-DECISIONS.md.

| Area | Adopted v1 target |
|---|---|
| Assignment model | One Problem per Lab; Group assignment optional/multi-Group; individual Student attempts and scores |
| Scoring | Module 7 evaluation percentage × Lab maxScore / 100; two-decimal half-up rounding; no score for incomplete evaluation |
| Attempts | Unlimited by default; optional maxAttempts; highest valid completed Lab score; latest accepted attempt breaks ties |
| Deadline | Optional UTC dueDate; inclusive hard cutoff by default; optional late acceptance without penalty |
| Lifecycle | DRAFT → PUBLISHED → CLOSED → ARCHIVED; restore to DRAFT; no hard delete |
| Version/history | Published Lab revisions pin immutable ProblemVersion and grading/availability settings; prior submissions remain pinned |
| Eligibility | Active Course enrollment plus Course/Lab/Problem scope intersection; GroupLab is an additional filter, not an enrollment substitute |
| Grading | Manual score override disabled in v1; feedback optional and Student-owner readable |
| Visibility | Student own results; HOD/Manager scoped summaries; Finance metadata-only; raw/hidden artifacts follow Modules 6/7 |

Remaining work is engineering: schema migration, server-side RBAC/scope checks, Module 6/7 integration, GroupLab integration with Module 10, lifecycle/event wiring and QA execution.

## 21. Dependencies

- Module 4 — Course & Enrollment: course association and Student eligibility.
- Module 5 — Problem Management: Problem selection, publication, and versioning.
- Module 6 — Code Submission & Judge0: submission and execution.
- Module 7 — Test Case Evaluation: testcase results and evaluation percentage.
- Module 10 — Groups: Group assignment and membership.
- Module 13 — Notifications: Lab and grading events.
- Module 15 — Audit Logging: score adjustments and staff actions.
- Module 17 — Functional State Machines: Lab and submission lifecycle.
- Module 18 — Edge Cases & Failure Handling: queue, Judge0, and recovery behavior.

## 22. Source references

- Business Flow v2: `docs/source-text/Business Flow v2.md`
- Page-to-DB Mapping v2: `docs/source-text/Page - DB Mapping v2.md`
- Schema reconstruction: `docs/source-text/scheema.md`
- Module 6 — Code Submission & Judge0
- Module 7 — Test Case Evaluation
- User-provided forensic report: `DEZOLVER_LABS_AND_LAB_SCORING_DEEP_DIVE.md`

### Evidence summary

- **Business Flow v2:** Group/Lab/Problem/submission/evaluation/score workflow.
- **Page-to-DB Mapping v2:** HOD/Manager Courses & Labs and Student My Labs.
- **Conceptual schema:** LabAssignment, LabSubmission, GroupLab.
- **Legacy code audit:** Dedicated Lab subsystem absent in investigated snapshot; underlying individual submissions and weighted testcase evaluation exist.

---

**Authoring checkpoint:** Module 8 is a detailed source-grounded draft. Recommended defaults remain unapproved. The user has directed that all modules be completed first and that clarification/decision resolution happen in a later cross-module review. Continue to Module 9 without pausing for LAB-D decisions.


---

# Functional Specification Standard v1.0 — detailed function-level expansion

> This expansion follows the user’s current workflow: complete every module from the supplied source documents first, keep missing/conflicting details explicit, and resolve them after all modules are authored. Repository inclusion is not product approval.

## 23. Evidence labels and source interpretation

| Label | Meaning | Module 8 usage |
|---|---|---|
| SOURCE—BUSINESS FLOW v2 | Updated business flow source | Group → Lab → Problem → Student → Code Submission → Judge0 → Evaluation → Lab Score; HOD/Manager responsibilities |
| SOURCE—BUSINESS FLOW (original) | Original business flow source | Similar Lab flow, but some diagrams omit Judge0 as a named step |
| SOURCE—PAGE MAPPING v2 | Updated page/entity map | HOD/Manager Courses & Labs, Manager Student Performance, Student Dashboard, Assignments, My Labs, My Groups |
| SOURCE—PAGE MAPPING (original) | Original page/entity map | Earlier/lighter entity lists; differences are preserved |
| SOURCE—RBAC | Roles, permissions, scope and ownership rules | Role/action/scope matrix |
| SOURCE—SCHEMA | Conceptual ERD and field list | LabAssignment, LabSubmission, GroupLab, Course, CourseEnrollment, CourseSection, Group, GroupMember, enums |
| AUDIT—LEGACY | Forensic report referenced in the existing Module 8 draft, outside the six primary source files | Kept separate; does not establish target behavior |
| DERIVED | Required to apply source constraints securely/consistently | Server-side scope checks, ownership checks, hidden-test protection |
| RECOMMENDATION | Suggested target behavior | Not approved |
| TBD / OPEN DECISION | Source omission/conflict | Carry forward to later cross-module review |

### 23.1 Source coverage and discrepancies

| Source | Relevant information captured | Limits |
|---|---|---|
| Business Flow v2 | HOD oversees department activities; Manager manages assigned Section; Groups mediate Labs; Lab links Problem → Student submission → Judge0 → Evaluation → Lab Score | Does not define CRUD screens, attempts, deadline policy, score formula, publication or group collaboration |
| Business Flow original | HOD/Manager structure; Group/Lab/Problem/Student submission/evaluation; Student Group → Lab flow | Some steps are less explicit than v2; no detailed lifecycle |
| Page Mapping v2 | HOD and Manager Courses & Labs; Manager Student Performance; Student Dashboard, Assignments, My Labs, My Groups; associated entities | Entity/page mapping is not a detailed function contract |
| Page Mapping original | Original Courses & Labs and My Labs mapping | Lists fewer related entities than v2; omission is not proof that functionality was removed |
| RBAC | Super Admin/HOD/Manager/Finance/Student Lab permissions; College/Department/Section/Group/ownership scope; server-side authorization | Manager permission verbs differ across matrix sections |
| Schema | LabAssignment, LabSubmission, GroupLab; Course, CourseSection, CourseEnrollment, Group, GroupMember; ContentScope and PublishStatus enums | Conceptual schema; LabAssignment lacks a status field despite PublishStatus enum; attempt, late policy, score history and completion not defined |

### 23.2 Page-to-DB Mapping v2 vs original

- HOD Courses & Labs, v2: Course, CourseSection, CourseEnrollment, CourseProblem, GroupCourseProblem, LabAssignment, LabSubmission, CodeSubmission, Problem.
- HOD Courses & Labs, original: Course, CourseSection, CourseEnrollment, LabAssignment, Problem, LabSubmission.
- Manager Courses & Labs, v2: same expanded set as HOD Courses & Labs.
- Manager Courses & Labs, original: Course, CourseSection, CourseEnrollment, LabAssignment, Problem, LabSubmission.
- Manager Student Performance, v2: User, Section, CourseEnrollment, LabSubmission, CodeSubmission, ContestScoreRank.
- Manager Student Performance, original: User, Section, CourseEnrollment, LabSubmission, CodeSubmission, Assessment.
- Student Dashboard, v2: User, College, Department, Section, CourseEnrollment, LabSubmission, ContestParticipant, ContestScoreRank, Notification.
- Student Dashboard, original: User, CourseEnrollment, Course, Assessment, LabSubmission, CodeSubmission, Certificate, ContestScoreRank.
- Student Assignments, v2: LabAssignment, LabSubmission, CodeSubmission, Problem, Course.
- Student My Labs, v2: LabAssignment, LabSubmission, CodeSubmission, Problem, Course.
- Student My Labs, original: LabAssignment, LabSubmission, Problem, Course.
- Student My Groups, v2: Group, GroupMember, GroupLab, GroupContest, User, Department; original lists a smaller subset.

These are page/entity associations, not proof of exact routes, UI controls, APIs, or filter behavior.

## 24. Terminology and field-level source reference

| Term | Source meaning | Evidence |
|---|---|---|
| Lab | Academic activity connecting Group, Problem, Student code submission, evaluation and Lab score | Business Flow |
| LabAssignment | Conceptual record linked to one Course and one Problem | Schema |
| LabSubmission | Student submission record linked to LabAssignment and CodeSubmission | Schema |
| GroupLab | Group ↔ LabAssignment junction | Schema |
| Lab scope | ContentScope: DEPT_WIDE, SECTION_ONLY, GLOBAL | Schema |
| PublishStatus | DRAFT, PUBLISHED, ARCHIVED | Schema enum; LabAssignment does not list a status field |
| Student eligibility | Permitted Course/Section/Group/academic relationship plus RBAC | Derived; precedence TBD |
| Lab score | LabSubmission.score, decimal(5,2) | Schema |
| Lab feedback | LabSubmission.feedback, text | Schema |
| Attempt | CodeSubmission/LabSubmission occurrence; no attempt number or limit field shown | Schema gap |

## 25. Actors, permissions, and scope

### 25.1 RBAC source matrix

| Role | RBAC module matrix: Labs | Page matrix: Courses & Labs | API matrix: Lab API | Data scope |
|---|---|---|---|---|
| SUPER_ADMIN | Full CRUD | V/C/U/D | Full CRUD | Global |
| FINANCE_TEAM | View | V | V* | Assigned College |
| HOD | Full CRUD | V/C/U/D | Full CRUD* | Assigned College + Department |
| MANAGER | Full CRUD* | V/C/U* | Full CRUD* | Assigned College + Department + assigned Section/permitted Group |
| STUDENT | View/Submit | V/SUBMIT | V/SUBMIT | Own records + permitted academic resources |

**Conflict:** the broad role/module matrix grants Manager Full CRUD* for Labs; the page and API matrices show V/C/U* (no explicit Delete). Do not silently grant Delete. Decision LAB-D06 remains open.

### 25.2 Authorization requirements

1. Role determines the action; scope determines which record the action applies to.
2. HOD is limited to assigned College + Department.
3. Manager is limited to assigned College + Department + assigned Section/permitted Group.
4. Student-owned LabSubmission/result access is restricted to the authenticated Student.
5. Backend must validate role, permission, scope and ownership for every protected operation; UI visibility is not security.
6. Student eligibility must be checked server-side for list, detail and submit operations.
7. Finance Team has View in the RBAC source; exact Lab content, student submission, score and feedback visibility remains unresolved.
8. “Lab Staff” appears in page mapping, but Faculty/Lab Staff is not one of the five roles in the RBAC role list. Do not create a new role implicitly.

### 25.3 Action-level access

| Operation | Required action | Scope/ownership |
|---|---|---|
| Staff Lab list | View | College/Department/Section/Group |
| Create Lab | Create | Course and Problem within staff scope |
| Edit Lab | Update | Existing Lab and relations within scope |
| Delete Lab | Delete only if permission resolved | Manager Delete discrepancy |
| Publish/archive | Update/lifecycle action | Scoped Lab; exact lifecycle permission TBD |
| Assign Group | Create/update relation | Group within staff scope |
| Student My Labs/detail | View | Course/Section/Group eligibility |
| Student submit | Submit/Create | Student identity + permitted Lab/Problem |
| Student result/feedback | View own | LabSubmission.studentId = authenticated Student |
| Staff review | View | HOD/Manager authorized academic scope |
| Manual grade override | Explicit grading permission | Not granted explicitly by source RBAC |
| Rejudge | Module 6/7 permission | Scoped submission; target permission TBD |

## 26. Pages and entry points

Exact route strings, control labels, sorting, filters and pagination are not provided by the source files.

| Actor | Page/entry point | Functional role | Source |
|---|---|---|---|
| HOD | Courses & Labs | View/create/update Lab assignments and review related submissions | Page Mapping v2 |
| HOD | Department Analytics | Read analytics involving GroupLab/GroupContest; Lab metrics not specified | Page Mapping v2 |
| MANAGER | Courses & Labs | Scoped Lab management | Page Mapping v2 |
| MANAGER | Student Performance | Review LabSubmission/CodeSubmission within scope | Page Mapping v2 |
| STUDENT | Dashboard | LabSubmission-related overview | Page Mapping v2 |
| STUDENT | Assignments | LabAssignment, LabSubmission, CodeSubmission, Problem, Course | Page Mapping v2 |
| STUDENT | My Labs | Browse, open, submit and view permitted results | Page Mapping + Business Flow |
| STUDENT | My Groups | Group/GroupLab context | Page Mapping v2 |
| SYSTEM | Submission/evaluation processing | Link CodeSubmission, evaluation and LabSubmission/score | Business Flow + Modules 6/7 |

## 27. Functional inventory

| Function ID | Function | Actor | Entry point | Type | Status |
|---|---|---|---|---|---|
| LAB-F001 | List/filter Labs for staff | HOD/Manager/Admin | Courses & Labs | Read | Source page/RBAC |
| LAB-F002 | Create LabAssignment | HOD/Manager/Admin | Courses & Labs | Create | Schema/RBAC |
| LAB-F003 | Edit LabAssignment | HOD/Manager/Admin | Lab detail | Update | Schema/RBAC |
| LAB-F004 | Publish/archive Lab | Authorized staff | Lab lifecycle action | Workflow | Enum source; field missing |
| LAB-F005 | Configure scope and GroupLab | Authorized staff | Lab/Groups | Assignment | Schema/Business Flow |
| LAB-F006 | List eligible Labs for Student | Student | My Labs/Assignments | Read | Mapping/RBAC |
| LAB-F007 | View Lab and Problem details | Student/staff | Lab detail | Read | Business Flow/schema |
| LAB-F008 | Submit code for Lab Problem | Student | Lab detail | Workflow | Business Flow/RBAC |
| LAB-F009 | Create/link LabSubmission | System | Submission intake | Data workflow | Schema |
| LAB-F010 | Evaluate through Judge0 | System | CodeSubmission | Integration | Business Flow/Modules 6–7 |
| LAB-F011 | Convert evaluation to Lab score | System | Evaluation complete | Calculation | Score field + Business Flow |
| LAB-F012 | View Student submission history/result | Student | My Labs/result | Read | Mapping/RBAC |
| LAB-F013 | Staff review Lab submissions | HOD/Manager | Student Performance/Courses & Labs | Read | Mapping/RBAC |
| LAB-F014 | Record/display feedback | Staff/Student | Result review | Update/read | Schema feedback field |
| LAB-F015 | Manual score adjustment | Authorized staff | Grading view | Update | Recommendation only |
| LAB-F016 | Enforce dueDate/late policy | System/Student | Submit | Validation/workflow | dueDate source; policy TBD |
| LAB-F017 | Resolve Course/Section/Group eligibility | System | List/detail/submit | Authorization | Schema/RBAC |
| LAB-F018 | Track Lab progress/completion | Student/staff/system | Dashboard/My Labs | Read/update | Page mapping; completion TBD |
| LAB-F019 | Rejudge/re-evaluate and update score | Staff/system | Rerun | Workflow | Modules 6/7; Lab policy TBD |
| LAB-F020 | Preserve history and integrate audit/events | System | Lifecycle/submission/grading | System | Modules 13/15 |

## 28. Detailed function specifications

### LAB-F001 — List and filter Labs for staff

**Requirement status:** SOURCE—PAGE MAPPING + SOURCE—RBAC. **Specification status:** Ready for review; UI details TBD. **Priority:** TBD.

**Purpose/outcome:** Display Lab assignments within the authenticated staff member’s academic scope.

**Actors:** SUPER_ADMIN, HOD, MANAGER; FINANCE_TEAM has View per RBAC.

**Permissions/scope:** Require View. Super Admin global; HOD College+Department; Manager assigned College+Department+Section/permitted Group; Finance Team assigned College. Apply scope on the server before returning rows.

**Preconditions/entry:** Authenticated account with View permission; HOD/Manager Courses & Labs page. Page Mapping lists related Course, LabAssignment, LabSubmission, CodeSubmission, Problem and enrollment/group entities.

**UI behavior:** Display scoped Labs. Exact columns, search, sort, filters, pagination, empty/loading/error states and UI labels are not defined by source (TBD). Do not expose student result data in list rows unless authorized.

**Inputs:** authenticated identity; system-derived scope; optional query/filter/sort/page parameters (not source-defined).

**Main flow:** (1) Actor opens Courses & Labs. (2) Backend checks role/action. (3) Backend resolves scope. (4) Query scoped LabAssignment/Course/Problem. (5) Include submission summaries only where permitted. (6) Return results and render list.

**Rules:** LAB-R01/R02. UI filters cannot widen scope. Validation LAB-V001.

**Backend/data:** Read LabAssignment and related entities; no mutation. Page mapping is not API specification.

**Success/failure/edge:** Scoped list; empty list is a normal state. Out-of-scope filter must not reveal record existence. Missing scope context → deny or safe empty result (TBD). Large lists/pagination TBD.

**Audit/privacy:** Read audit requirements belong to Module 15. Student/private results must not leak.

**Acceptance:** HOD/Manager only see scoped Labs; Finance Team only approved view; Students cannot use staff page. **QA:** LAB-QA-001–006.

**Unresolved:** LAB-D01, D06, D36.

### LAB-F002 — Create LabAssignment

**Requirement status:** SOURCE—SCHEMA + SOURCE—RBAC. **Specification status:** Decision required.

**Purpose/outcome:** Create LabAssignment linked to a Course and Problem.

**Actors/permissions:** SUPER_ADMIN Full CRUD; HOD Full CRUD within College/Department; Manager scoped Create/Update in page/API matrices (role matrix also says Full CRUD*); Finance View; Student no create.

**Preconditions/entry:** Authenticated actor with Create permission; Courses & Labs create action (exact UI control TBD); valid Course and Problem in scope.

**UI/inputs:**

| Field | Schema type | Required | Validation/notes |
|---|---|---|---|
| courseId | varchar | Required relation | Existing Course in scope |
| problemId | varchar | Required relation | Existing Problem in scope |
| title | varchar(255) | Requiredness not explicitly specified | Max length 255 |
| description | text | Nullable/requiredness TBD | Text limits TBD |
| scope | ContentScope | Requiredness TBD | DEPT_WIDE / SECTION_ONLY / GLOBAL |
| maxScore | int | Score configuration | Range/positive requirement TBD |
| dueDate | timestamp | Nullable status TBD | Timezone/late semantics TBD |
| id/createdAt/updatedAt | System | System-generated | Not user-editable |
| Group assignment | GroupLab relation | Optionality TBD | Separate/inline workflow TBD |

**Main flow:** Open create form → select Course and Problem → enter fields → submit → server validates permission/scope/relations/enum/types → persist LabAssignment → create GroupLab links if selected and valid → show saved Lab.

**Rules:** LAB-R04–R06. Do not assume multi-Problem Labs: schema has one problemId. Do not assume group assignment is mandatory.

**Validation:** LAB-V002–V007. Invalid relation/scope/length/date must block save. maxScore range remains TBD; do not silently clamp.

**Backend/data:** Create schema fields exactly: id, courseId, problemId, title, description, scope, maxScore, dueDate, createdAt, updatedAt. Creator/status/publishedAt/late-policy fields are not listed in LabAssignment schema.

**Success/failure/edge:** Success creates record. Cross-scope relation denied. Duplicate request/idempotency policy TBD. Course/Problem state changing during save must be revalidated. Group link failure after create requires transaction/reconciliation policy.

**Audit/privacy:** Module 15 audit policy; Course schema has createdById but LabAssignment does not.

**Acceptance:** valid scoped record saved; invalid Course/Problem/scope denied; server-side validation; no unsupported fields assumed. **QA:** LAB-QA-007–015.

**Unresolved:** LAB-D02–D05, D07, D10, D24.

### LAB-F003 — Edit LabAssignment

**Requirement status:** SOURCE—SCHEMA + RBAC. **Specification status:** Decision required.

**Purpose:** Update permitted Lab fields without corrupting prior submissions/results.

**Permissions/preconditions:** Update permission and scope to Lab/Course/Problem. Entry from scoped Lab detail/edit. Student/Finance Team cannot update.

**UI/inputs:** Existing source fields courseId, problemId, title, description, scope, maxScore, dueDate. Whether Course/Problem are immutable after submission is TBD. id/createdAt immutable; updatedAt changes after successful update.

**Main flow:** Load scoped Lab → display existing values → actor edits → server validates changes and relation scope → check dependent submissions → apply approved edit policy → update updatedAt → show result.

**Rules:** LAB-R07/R08. Changing Problem, scope, maxScore or dueDate after submission may affect historical results; never silently recalculate grades.

**Validation:** LAB-V008–V010; stale concurrent update must not silently overwrite. Invalid fields/relation rejected. Student edit denied.

**Backend/data:** Update only approved LabAssignment fields; related LabSubmission/CodeSubmission remain intact unless an explicit policy applies.

**Failure/edge:** Archived Lab, deleted/archived Course or Problem, Group membership changes, existing submissions, concurrent staff edits, deadline passed.

**Audit/privacy:** Module 15 should record actor/target/changed fields under approved policy.

**Acceptance:** authorized staff update only scoped Lab; invalid edits rejected; historic submissions preserved. **QA:** LAB-QA-016–023.

**Unresolved:** LAB-D06, D08, D11–D13.

### LAB-F004 — Publish/archive Lab

**Requirement status:** SOURCE—SCHEMA enum; operation details TBD.

**Purpose:** Control availability through Draft/Published/Archived concepts.

**Permissions/preconditions:** Scoped staff lifecycle permission; Student cannot change lifecycle. Exact action permission TBD.

**Inputs:** Lab ID, requested lifecycle value; enum source is DRAFT/PUBLISHED/ARCHIVED. LabAssignment does not list a status field, while Course has status: PublishStatus.

**Main flow:** Actor requests lifecycle action → backend checks role/scope → validate current lifecycle and dependencies → validate requested enum/transition → persist in approved target model → update visibility/submit eligibility.

**Rules:** LAB-R09–R11. “Closed” is not a source enum; it remains recommendation only. Archive does not mean delete submission history.

**Validation:** LAB-V011–V013. Invalid status/unauthorized Student action denied; publishing incomplete Lab behavior TBD.

**Backend/data:** Status field/location must be confirmed; do not assume it exists on LabAssignment.

**Success/failure/edge:** Publish, unpublish, archive, reopen, delete, archive after grading, and active submission behavior remain TBD.

**Audit/privacy:** Lifecycle change event to Module 15; notifications to Module 13 if approved.

**Acceptance:** only authorized lifecycle change; enum enforced; historical submissions retained per policy. **QA:** LAB-QA-024–030.

**Unresolved:** LAB-D10, D11; Module 17.

### LAB-F005 — Configure Lab scope and GroupLab assignment

**Requirement status:** SOURCE—SCHEMA + BUSINESS FLOW. **Specification status:** Decision required.

**Purpose:** Associate a Lab with a Group through GroupLab and enforce ContentScope.

**Actors/permissions:** HOD/Manager scoped create/update; Super Admin global; Student cannot change assignment. Both Lab and Group must be within actor scope.

**Preconditions/entry:** Existing LabAssignment and Group; entry from Lab configuration or Groups page (exact UI TBD).

**Inputs:** LabAssignment ID; Group ID(s); scope enum; system createdAt. Number of groups/required assignment TBD.

**Main flow:** Load Lab and permitted Groups → actor selects target(s) → server checks College/Department/Section/Group scope → create/remove GroupLab relation → recalculate eligibility under approved membership policy.

**Rules:** LAB-R12/R13. GroupLab is assignment relation, not evidence of shared group submission or score ownership.

**Validation:** LAB-V014–V016. Foreign Group denied; duplicate relation behavior TBD; Group/Lab scope mismatch rejected or exception decision.

**Backend/data:** GroupLab fields groupId, labAssignmentId, createdAt. No unique constraint shown.

**Failure/edge:** inactive Group, membership added/removed, Group deleted, Lab scope changed, relation removed after submission.

**Audit/privacy:** Assignment changes are candidates for Module 15; notifications Module 13.

**Acceptance:** valid scoped links; membership is checked separately; no automatic collaborative grading. **QA:** LAB-QA-031–038.

**Unresolved:** LAB-D05, D14–D16; Module 10.

### LAB-F006 — List eligible Labs for Student

**Requirement status:** SOURCE—PAGE MAPPING + RBAC. **Specification status:** Ready for review; eligibility precedence TBD.

**Purpose:** Show permitted Labs in My Labs/Assignments/Dashboard.

**Actor/permissions:** Student View; backend uses authenticated identity, not arbitrary studentId.

**Preconditions/entry:** Authenticated Student; My Labs, Assignments, Dashboard or My Groups.

**UI/input:** Lab list; mapped entities LabAssignment, LabSubmission, CodeSubmission, Problem, Course. Search/sort/filter controls not specified.

**Main flow:** Student opens page → server resolves identity → reads CourseEnrollment, CourseSection, GroupMember, GroupLab and Lab scope → applies eligibility policy → returns allowed Lab and own submission/score summary → UI renders list.

**Rules:** LAB-R14–R16. CourseEnrollment and GroupMember are separate relations. GroupLab alone does not establish enrollment.

**Validation:** LAB-V017/V018. Ineligible Student excluded/denied; client-supplied foreign studentId denied.

**Backend/data:** Read CourseEnrollment, CourseSection, GroupMember, GroupLab, LabAssignment, Course, Problem, LabSubmission. No mutation.

**Success/failure/edge:** Empty state; join Group after publish; change Section; enrollment revoked; group membership removed. Eligibility timing TBD.

**Privacy/acceptance:** Student sees own submissions only. QA LAB-QA-039–046.

**Unresolved:** LAB-D14–D18.

### LAB-F007 — View Lab details and associated Problem

**Requirement status:** SOURCE—BUSINESS FLOW + PAGE MAPPING. **Specification status:** Ready for review.

**Purpose:** Display Lab instructions and Problem linked through LabAssignment.problemId.

**Permissions/preconditions:** Student eligible; staff scoped View. Lab/Problem accessible under lifecycle policy.

**Entry/UI/input:** Lab ID from My Labs/Courses & Labs. Display title, description, Course, Problem, maxScore, dueDate and permitted status. Problem details owned by Module 5. Exact labels/layout TBD.

**Main flow:** Open Lab → server verifies permission/scope/eligibility → load LabAssignment/Course/Problem → apply visibility filtering → return allowed details → render.

**Rules:** LAB-R17/R18; hidden testcases protected by Module 7.

**Validation:** LAB-V019/V020; inaccessible Lab/Problem denied/not-found-safe; no hidden test input/output returned.

**Backend/data:** Read LabAssignment/Course/Problem and assignment relations.

**Failure/edge:** archived Problem, Lab archived, Problem changed after publication; historical snapshot policy TBD.

**Acceptance:** correct Problem relation; only eligible actor; hidden test data protected. QA LAB-QA-047–052.

**Unresolved:** LAB-D11, D13, D20, D23.

### LAB-F008 — Submit code for Lab Problem

**Requirement status:** SOURCE—BUSINESS FLOW + RBAC. **Specification status:** Ready for review; detailed policies TBD.

**Purpose:** Submit code for Lab’s associated Problem through Modules 6/7.

**Permissions/preconditions:** Student Submit; authenticated account; eligible Lab/Problem; Lab accepts submission under approved lifecycle/deadline/attempt policy; supported language.

**Entry/UI:** My Labs → Lab detail → submission editor (route/editor/upload specifics not sourced).

**Inputs:** Lab ID; Problem ID derived from Lab; authenticated Student; source code/language (Module 6); server timestamp. Client cannot select arbitrary Problem or Student identity.

**Main flow:** Student writes/selects code → submit → backend verifies role/scope/eligibility/status/dueDate/Problem relation/language → Module 6 creates CodeSubmission → LAB-F009 creates LabSubmission → evaluation starts → return pending/accepted-for-processing state.

**Rules:** LAB-R19–R21. Unauthorized submission must not create Judge0 execution. Lab rules do not silently inherit Assessment/Contest behavior.

**Validation:** LAB-V021–V025; inaccessible Lab/Problem, closed/deadline policy, unsupported language, duplicate submit, attempt cap.

**Backend/data:** CodeSubmission belongs to Module 6; Lab context must be retained for linkage. Atomicity/idempotency between CodeSubmission and LabSubmission TBD.

**Failure/edge:** Judge0 unavailable, client retry, deadline race, duplicate click, eligibility revoked during request. Operational error is not code failure.

**Audit/privacy:** Code source and private submissions governed by Modules 6/15.

**Acceptance:** correct Student/Lab/Problem linkage; unauthorized request produces no CodeSubmission/Judge0 work; no ID spoofing. QA LAB-QA-053–064.

**Unresolved:** LAB-D14–D20, D22–D23.

### LAB-F009 — Create/link LabSubmission

**Requirement status:** SOURCE—SCHEMA. **Specification status:** Decision required.

**Purpose:** Persist Lab-specific submission linkage.

**Actor/trigger:** System after CodeSubmission is accepted for Lab context; Student cannot directly create arbitrary relations.

**Preconditions:** valid LabAssignment, authenticated Student and CodeSubmission.

**Inputs/schema:** id; labAssignmentId; studentId; codeSubmissionId; score decimal(5,2); feedback text; submittedAt timestamp. Nullability, attempt number, status, verdict, unique constraints and score initialization are not specified.

**Main flow:** accepted CodeSubmission → derive Lab context from trusted server state → validate relations → create LabSubmission → set submittedAt → link to evaluation → return reference.

**Rules:** LAB-R22–R24. studentId from auth; LabSubmission must match CodeSubmission/Lab/Student.

**Validation:** LAB-V026–V028; invalid relation rejected; duplicate callback idempotency/reconciliation required; score initial state TBD.

**Backend/data:** Create LabSubmission using source fields; do not invent status or attempt fields.

**Failure/edge:** CodeSubmission created but LabSubmission write fails; evaluation enqueue failure; callback retries; orphan reconciliation; rejudge.

**Privacy/audit:** correct ownership, source-code privacy.

**Acceptance:** correct relationships, no foreign linkage, no unintended duplicate row. QA LAB-QA-065–071.

**Unresolved:** LAB-D19, D22, D23, D25.

### LAB-F010 — Evaluate Lab submission via Judge0

**Requirement status:** SOURCE—BUSINESS FLOW v2; Modules 6/7 own technical execution/evaluation. **Specification status:** Ready for review.

**Purpose:** Execute code against Problem testcases, evaluate and make result available for Lab score.

**Actor:** system/worker; Judge0 external service. Only authorized submission may enter execution.

**Preconditions/input:** CodeSubmission accepted; Lab/Problem link valid; Module 6/7 input/language/testcase config.

**Main flow:** Module 6 submits to Judge0 → Module 7 collects/evaluates testcase results → evaluation result is correlated to CodeSubmission/LabSubmission → LAB-F011 computes Lab score → Student result view updates.

**Rules:** LAB-R25–R27. Lab must not implement separate Judge0; testcase verdict belongs to Module 7; infrastructure error not automatically Student-code failure.

**Validation:** LAB-V029/V030; incomplete evaluation yields pending/incomplete, not final grade; wrong CodeSubmission correlation rejected.

**Backend/data:** Modules 6/7 own queue, tokens, retries, statuses and testcase processing. Lab consumes final result only.

**Failure/edge:** Judge0 outage, partial results, late result, duplicate event, rejudge generation, Problem changed after submission. Recovery owned by Modules 6/7/18.

**Privacy:** Hidden testcase input/expected output and raw Judge0 details remain filtered.

**Acceptance:** correct execution/testcase context; result correctly correlated; infrastructure errors distinct. QA LAB-QA-072–079.

**Unresolved:** LAB-D20–D23; Modules 6/7/18.

### LAB-F011 — Convert evaluation to Lab score

**Requirement status:** SOURCE—SCHEMA confirms score field; Business Flow confirms Evaluation → Lab Score. Formula is not source-confirmed. **Specification status:** Decision required.

**Purpose:** Calculate/persist Lab score from evaluation and maximum score.

**Actor/trigger:** system after finalized evaluation.

**Inputs:** Module 7 evaluation score/verdict; LabAssignment.maxScore (int); LabSubmission.score (decimal(5,2)); optional late/manual adjustment only if approved.

**Existing draft recommendation:** Lab Score = Evaluation Percentage / 100 × Lab Max Score. This formula is not in the six primary sources and remains unapproved.

**Main flow:** load finalized result and maxScore → verify CodeSubmission correlation → apply approved conversion → apply approved rounding/penalty/override if configured → persist LabSubmission.score → expose to authorized Student/staff.

**Rules:** LAB-R28–R31. Deterministic calculation; no score above approved maximum; incomplete result not final; schema precision does not define rounding.

**Validation:** LAB-V031–V034; invalid evaluation range/maxScore/unknown verdict; no unapproved penalty.

**Backend/data:** Update or derive LabSubmission.score; whether stored or derived is architecture decision.

**Failure/edge:** partial credit, zero maxScore, rounding, rejudge, maxScore change after grading, late penalty, manual adjustment.

**Privacy:** score visibility by ownership/staff scope.

**Acceptance:** formula, precision and max-score rules approved and followed; incomplete evaluation cannot award unintended score. QA LAB-QA-080–090.

**Unresolved:** LAB-D07, D08, D21, D22, D26, D27.

### LAB-F012 — Student view of submission history/result

**Requirement status:** SOURCE—PAGE MAPPING + RBAC + schema. **Specification status:** Ready for review; attempt history TBD.

**Purpose:** Student views own LabSubmission, evaluation state, score and permitted feedback.

**Permissions/preconditions:** View own; enforce LabSubmission.studentId = authenticated Student. Entry from My Labs/Lab result.

**UI/input:** Lab/LabSubmission ID. Display timestamp, current status, score and feedback if available; history ordering/attempt selector TBD.

**Main flow:** Student opens result → backend checks owner → loads LabAssignment/CodeSubmission/evaluation → masks hidden/raw fields → returns safe DTO → UI renders.

**Rules:** LAB-R32–R34. Client ID does not bypass ownership. Pending/incomplete is not a final grade.

**Validation:** LAB-V035/V036; foreign ID denied/not-found-safe; missing result shows pending/incomplete.

**Backend/data:** Read LabSubmission, LabAssignment, CodeSubmission and Module 7 result.

**Failure/edge:** pending, rejudge pending, old attempt, enrollment revoked, archived Lab, result retained/deleted.

**Acceptance:** own record only, safe DTO, hidden details masked. QA LAB-QA-091–097.

**Unresolved:** LAB-D14, D17, D18, D20, D21, D23.

### LAB-F013 — Staff review Lab submissions/performance

**Requirement status:** SOURCE—PAGE MAPPING v2 + RBAC. **Specification status:** Ready for review.

**Purpose:** HOD/Manager inspect scoped LabSubmission/CodeSubmission and student performance.

**Permissions:** HOD College+Department; Manager assigned Section/permitted Group; Super Admin global. Finance Team View field-level scope TBD. Student cannot access staff review.

**Entry/input:** HOD/Manager Courses & Labs or Manager Student Performance. Possible filters (Course, Section, Group, Student, Lab, status) are inferred, not source-defined.

**Main flow:** open page → check role/scope → query User/Section/CourseEnrollment/LabSubmission/CodeSubmission/LabAssignment/Problem → return authorized summary/detail → staff opens permitted submission.

**Rules:** LAB-R35–R37. View does not imply raw Judge0/token/hidden testcase access. Export permission not implied.

**Validation:** LAB-V037/V038; out-of-scope Student/filter denied or safely filtered; export requires explicit permission.

**Backend/data:** Read mapped entities; no mutation.

**Failure/edge:** Section change, Group membership changes, cross-Department enrollment, stale assignment, large lists.

**Audit/privacy:** Staff access to private grading data per Module 15; raw details separately authorized.

**Acceptance:** correct scope for HOD/Manager; no raw result access without permission. QA LAB-QA-098–105.

**Unresolved:** LAB-D06, D14, D17, D23, D36.

### LAB-F014 — Record/display feedback

**Requirement status:** SOURCE—SCHEMA confirms LabSubmission.feedback text; workflow not specified. **Specification status:** Decision required.

**Purpose:** Store staff feedback associated with a LabSubmission and display it to authorized Student.

**Permissions:** Staff update permission in scope; Student View own. Exact role-based grading/feedback permission is not a separate RBAC entry.

**Preconditions/entry:** Staff opens scoped submission; Student opens own result.

**Inputs:** LabSubmission ID; feedback text; actor identity. Length, formatting, nullability and empty/clear behavior TBD.

**Main flow:** authorize target → enter feedback → validate → update LabSubmission.feedback → return saved value → Student reads own feedback.

**Rules:** LAB-R38/R39. Feedback cannot be attached to wrong Student/Lab or exposed cross-owner.

**Validation:** LAB-V039–V041; foreign target denied; text limit TBD; blank semantics TBD.

**Backend/data:** Update feedback field. LabSubmission schema has no updatedAt; audit timestamp/history must be handled centrally or added by decision.

**Failure/edge:** edit after score finalized, feedback per attempt vs final Lab, deleted/archived submission.

**Audit/notification:** Module 15 audit and Module 13 notification if policy requires.

**Acceptance:** authorized staff can save feedback; only Student owner can read; change traceable per policy. QA LAB-QA-106–111.

**Unresolved:** LAB-D24, D28.

### LAB-F015 — Manual score adjustment

**Requirement status:** RECOMMENDATION only; no explicit source requirement. **Specification status:** Decision required; not assumed enabled.

**Purpose:** Permit staff score override only if Product approves.

**Permissions/preconditions:** Source RBAC has Lab CRUD but no explicit grading override permission. Feature must remain disabled until permission/approval policy exists.

**Inputs (if approved):** LabSubmission ID, adjusted score, reason, actor, timestamp, optional feedback. Score history/provenance required if enabled.

**Proposed flow (not approved):** staff opens scoped submission → system shows automated score → actor enters override/reason → backend checks grading permission/range → preserves prior score → records override provenance → updates permitted Student view.

**Rules:** LAB-R40/R41. CRUD/View does not imply override permission. Score limits/bonus rule must be approved.

**Validation:** LAB-V042–V044; no permission denied; required reason missing rejected; concurrent update conflict surfaced.

**Backend/data:** Schema has score/feedback but no score history, reason, actor, override timestamp. AuditLog/dedicated history design TBD.

**Failure/edge:** multiple graders, rejudge after override, maxScore changed, revert, second approval, Student notification.

**Audit/privacy:** mandatory traceability if approved; Module 15 owns canonical audit.

**Acceptance:** not enabled until approved; if enabled, explicit permission/reason/history enforced. QA LAB-QA-112–118.

**Unresolved:** LAB-D22, D28, D34.

### LAB-F016 — Enforce dueDate and late policy

**Requirement status:** SOURCE—SCHEMA confirms dueDate timestamp; late policy absent. **Specification status:** Decision required.

**Purpose:** Use dueDate for submission eligibility/late classification once policy is approved.

**Actor/trigger:** system at submit; staff may update dueDate.

**Inputs:** LabAssignment.dueDate, server submission timestamp, approved late policy fields (not in schema).

**Main flow:** submission request → server timestamp → load dueDate/policy → determine accept/late/block → persist submittedAt → apply penalty only if policy approved.

**Rules:** LAB-R42–R44. dueDate does not imply hard closure or penalty. Submission time differs from Judge0 completion time.

**Validation:** LAB-V045–V047. Invalid date rejected; post-deadline behavior TBD; dueDate edits must not silently rewrite historical late status.

**Backend/data:** dueDate on LabAssignment; submittedAt on LabSubmission. No isLate, grace, penalty or extension field shown.

**Failure/edge:** deadline equality, timezone, DST, grace period, staff extension, request received before deadline but processed after, evaluation after deadline.

**Acceptance:** approved deadline policy enforced; grading completion time does not determine lateness unless explicitly decided. QA LAB-QA-119–126.

**Unresolved:** LAB-D09, D29–D31.

### LAB-F017 — Resolve Course/Section/Group eligibility

**Requirement status:** SOURCE—SCHEMA + RBAC. **Specification status:** Decision required.

**Purpose:** Determine Student access using CourseEnrollment, CourseSection, GroupMember, GroupLab and Lab scope.

**Trigger:** every list/detail/submit request. Student identity from auth; staff assignment uses staff scope.

**Inputs/entities:** CourseEnrollment(courseId/studentId/enrolledAt/completedAt); CourseSection(courseId/sectionId); GroupMember(groupId/studentId/joinedAt); GroupLab(groupId/labAssignmentId/createdAt); LabAssignment(courseId/scope); Group(collegeId/departmentId/isActive etc.).

**Main flow:** request Lab → resolve authenticated Student → load Course/scope → check enrollment → check relevant Section relationship → check GroupLab and membership if used → apply approved precedence → allow/deny.

**Rules:** LAB-R45–R47. Course enrollment and Group membership are distinct; ContentScope enum semantics are not defined by enum names alone.

**Validation:** LAB-V048–V051. inactive Group, missing enrollment, revoked membership, cross-scope association must follow approved rule; cross-tenant access denied unless an explicit permitted global rule applies.

**Backend/data:** read User/Course/CourseEnrollment/CourseSection/Section/Group/GroupMember/GroupLab/LabAssignment; no mutation.

**Failure/edge:** Student moves Section, joins after publication, leaves Group, enrollment revoked, scope changed, Group membership inconsistent with Course.

**Acceptance:** server rechecks eligibility for all requests; client IDs do not bypass. QA LAB-QA-127–137.

**Unresolved:** LAB-D14–D18, D32.

### LAB-F018 — Track Lab progress/completion

**Requirement status:** SOURCE—PAGE MAPPING indicates Student Dashboard/LabSubmission; completion semantics absent. **Specification status:** Decision required.

**Purpose:** Display progress while distinguishing submission, evaluation and completion.

**Actors/entry:** Student reads own progress; staff reads scoped progress; Dashboard/My Labs/Student Performance.

**Inputs:** LabAssignment, LabSubmission, CodeSubmission/evaluation, CourseEnrollment. CourseEnrollment.completedAt is Course completion, not Lab completion.

**Main flow:** list eligible Labs → load Student’s LabSubmission → derive approved progress state → display status/score.

**Rules:** LAB-R48–R50. Submission and completion are separate unless approved. Pending evaluation is not completed.

**Validation:** LAB-V052/V053; no submission/completion threshold is inferred from schema.

**Backend/data:** Schema has no Lab progress/status entity. Persisted vs derived progress requires decision.

**Failure/edge:** multiple attempts, pending result, partial score, manual override, Lab archived, eligibility lost.

**Acceptance:** distinct states and no unapproved completion threshold. QA LAB-QA-138–144.

**Unresolved:** LAB-D17, D18, D33.

### LAB-F019 — Rejudge/re-evaluate and refresh Lab score

**Requirement status:** Cross-module Modules 6/7; Lab-specific policy not specified. **Specification status:** Decision required.

**Purpose:** Re-evaluate a Lab CodeSubmission and reconcile score.

**Permissions:** use Module 6/7 rerun permissions; Student cannot invoke staff rejudge by default. RBAC source does not explicitly grant HOD/Manager rejudge.

**Main flow:** authorized request → Module 6 re-executes → Module 7 evaluates → correlate result to LabSubmission → recompute score under approved rule → preserve prior run/score history per policy.

**Rules:** LAB-R51–R53. Rejudge is not automatically a new Student attempt; stale prior result cannot overwrite current run; manual override precedence must be approved.

**Validation:** LAB-V054–V056. Unauthorized rerun denied; active run serialized/rejected; incomplete result not final.

**Backend/data:** CodeSubmission/evaluation history owned by Modules 6/7; LabSubmission.score may be updated. Score history/event/notification TBD.

**Failure/edge:** old result arrives late, multiple rejudge requests, manual override, changed Problem/testcases, infrastructure failure.

**Acceptance:** explicit permission, correct association, preserved history, no duplicate attempt. QA LAB-QA-145–151.

**Unresolved:** LAB-D20–D21, D26, D34; Modules 6/7/13/15/17/18.

### LAB-F020 — Preserve history and integrate audit/notifications

**Requirement status:** SOURCE—PAGE MAPPING/RBAC + DERIVED. **Specification status:** Review required.

**Purpose:** Keep Lab events consistent and integrate centralized audit/notification contracts.

**Actors/trigger:** system after Lab create/edit/publish/archive/assign/submit/evaluate/feedback/override/rejudge.

**Inputs:** event type, actor, target Lab/LabSubmission, scope, timestamp, before/after fields (only approved payload data).

**Main flow:** business operation commits → determine event → record audit if required → emit notification if configured → consumers process idempotently.

**Rules:** LAB-R54–R56. Module 13 owns notifications; Module 15 owns audit. Do not include hidden testcase input/output, source code, raw Judge0 tokens or secrets in ordinary event payloads.

**Validation:** LAB-V057/V058. Failed/uncommitted operation must not emit success; duplicate event must not duplicate score/progress.

**Backend/data:** no separate notification/audit schema is defined here; central contracts apply.

**Failure/edge:** audit/notification service outage, duplicate event, score update during delivery, archived Lab.

**Acceptance:** event contract is centrally defined; sensitive payloads filtered; duplicate processing safe. QA LAB-QA-152–158.

**Unresolved:** LAB-D24, D35; Modules 13/15/18.

## 29. Business rules and validation catalogue

### 29.1 Business rules

| Rule ID | Rule | Evidence |
|---|---|---|
| LAB-R01 | Staff list is scope-filtered | RBAC-derived |
| LAB-R02 | UI filters cannot bypass server scope | RBAC |
| LAB-R03 | Student-private LabSubmission is owner-restricted | RBAC |
| LAB-R04 | LabAssignment links Course and Problem | Schema |
| LAB-R05 | ContentScope values are DEPT_WIDE, SECTION_ONLY, GLOBAL | Schema |
| LAB-R06 | maxScore is int in conceptual schema | Schema |
| LAB-R07 | Course/Problem changes require scope validation | Derived |
| LAB-R08 | Changes after submissions require historic-score policy | Derived/TBD |
| LAB-R09 | PublishStatus enum is DRAFT/PUBLISHED/ARCHIVED | Schema |
| LAB-R10 | Closed is not a source enum | Source comparison |
| LAB-R11 | Archive does not mean delete submissions | Derived |
| LAB-R12 | GroupLab is Group↔LabAssignment relation | Schema |
| LAB-R13 | GroupLab and ContentScope are separate | Schema |
| LAB-R14 | Student access requires permitted academic relationship | RBAC |
| LAB-R15 | CourseEnrollment is course enrollment, not necessarily Lab assignment | Schema |
| LAB-R16 | Group membership may inform eligibility; precedence TBD | Derived/TBD |
| LAB-R17 | Lab references Problem via problemId | Schema |
| LAB-R18 | Lab/Problem visibility follows approved lifecycle/assignment policy | TBD |
| LAB-R19 | Submitted Problem must match Lab problemId | Derived |
| LAB-R20 | Unauthorized submit must not create CodeSubmission/Judge0 work | RBAC |
| LAB-R21 | Lab must not silently inherit Assessment/Contest policies | Derived |
| LAB-R22 | LabSubmission links LabAssignment, Student, CodeSubmission | Schema |
| LAB-R23 | studentId comes from authenticated identity | RBAC |
| LAB-R24 | Submission relationships must be mutually consistent | Derived |
| LAB-R25 | Lab reuses Modules 6/7; no second Judge0 implementation | Cross-module |
| LAB-R26 | Testcase verdict/score is owned by Module 7 | Cross-module |
| LAB-R27 | Infrastructure failure is not automatically Student failure | Cross-module |
| LAB-R28 | Lab score respects approved max-score policy | TBD |
| LAB-R29 | Score calculation is deterministic | Derived |
| LAB-R30 | Incomplete evaluation is not silently final | Derived |
| LAB-R31 | LabSubmission.score is decimal(5,2) | Schema |
| LAB-R32 | Student views own submission/result only | RBAC |
| LAB-R33 | Client IDs cannot bypass authorization | RBAC |
| LAB-R34 | Hidden testcase/raw Judge0 fields remain protected | Modules 6/7 |
| LAB-R35 | Manager performance access is Section/Group scoped | RBAC |
| LAB-R36 | HOD performance access is College+Department scoped | RBAC |
| LAB-R37 | Staff View does not automatically grant raw JudgeRun access | Derived |
| LAB-R38 | Feedback attaches to LabSubmission | Schema |
| LAB-R39 | Feedback is Student-owner scoped | Derived/RBAC |
| LAB-R40 | Manual override remains disabled until approved | Recommendation |
| LAB-R41 | Score range/bonus behavior must be approved | Recommendation |
| LAB-R42 | dueDate is timestamp, not a full late policy | Schema |
| LAB-R43 | Submission time differs from grading completion time | Derived |
| LAB-R44 | Late penalty is not applied without approval | Derived |
| LAB-R45 | CourseEnrollment and GroupMember are distinct | Schema |
| LAB-R46 | ContentScope semantics/inheritance require definition | Schema/TBD |
| LAB-R47 | GroupLab does not grant access to all College students | RBAC-derived |
| LAB-R48 | CourseEnrollment.completedAt is not Lab completion | Schema |
| LAB-R49 | Submission and completion are distinct unless approved | TBD |
| LAB-R50 | Pending evaluation is not completion | Derived |
| LAB-R51 | Rejudge is not automatically a new attempt | Derived |
| LAB-R52 | Stale rejudge result cannot overwrite newer run | Derived |
| LAB-R53 | Rejudge/manual override precedence is TBD | TBD |
| LAB-R54 | Notifications belong to Module 13 | Module ownership |
| LAB-R55 | Audit policy belongs to Module 15 | Module ownership |
| LAB-R56 | Hidden-test/source/token data is not exposed in normal events | Security-derived |

### 29.2 Validation matrix

| Validation ID | Condition | Validation point | Expected behavior | Feedback |
|---|---|---|---|---|
| LAB-V001 | Staff query outside scope | List | Scope/deny; no existence leak | Exact message TBD |
| LAB-V002 | Missing/invalid Course | Create | Block save | Message intent TBD |
| LAB-V003 | Missing/invalid Problem | Create | Block save | Message intent TBD |
| LAB-V004 | Title >255 chars | Create/edit | Reject | Length message TBD |
| LAB-V005 | Invalid ContentScope | Create/edit | Reject enum | Allowed values |
| LAB-V006 | Invalid maxScore | Create/edit | Reject per approved range | Range TBD |
| LAB-V007 | Invalid dueDate | Create/edit | Reject | Date message TBD |
| LAB-V008 | Stale concurrent edit | Edit | Conflict/reload | TBD |
| LAB-V009 | Invalid relation/scope | Edit | Reject | TBD |
| LAB-V010 | Student edits Lab | API | Deny | Access denied |
| LAB-V011 | Invalid lifecycle enum | Lifecycle | Reject | Status validation |
| LAB-V012 | Publish incomplete Lab | Publish | Block under approved rule | Required fields TBD |
| LAB-V013 | Student lifecycle change | API | Deny | Access denied |
| LAB-V014 | Group outside scope | Assignment | Deny | Scope message |
| LAB-V015 | Group/Lab incompatible scope | Assignment | Reject or explicit exception | TBD |
| LAB-V016 | Duplicate GroupLab | Assignment | Idempotent/reject per policy | TBD |
| LAB-V017 | Student ineligible | My Labs | Exclude/deny | No access |
| LAB-V018 | Client studentId != authenticated | API | Deny | Access denied |
| LAB-V019 | Lab/Problem inaccessible | Detail | Deny/not-found-safe | TBD |
| LAB-V020 | Hidden testcase requested | Detail | Mask/omit | No hidden content |
| LAB-V021 | Lab/Problem not accessible for submit | Submit | Deny before execution | Submission rejected |
| LAB-V022 | Deadline/closed | Submit | Approved policy | TBD |
| LAB-V023 | Unsupported language/source | Submit | Module 6 validation | Module 6 message |
| LAB-V024 | Duplicate submit | Submit | Avoid unintended duplicate attempt | TBD |
| LAB-V025 | Attempt limit reached | Submit | Block per policy | TBD |
| LAB-V026 | Invalid LabSubmission relation | Persist | Reject/reconcile | Operational error |
| LAB-V027 | Duplicate create/callback | Persist | Idempotent/reconcile | No duplicate |
| LAB-V028 | Invalid initial score | Persist | Await result/reject per policy | TBD |
| LAB-V029 | Incomplete evaluation | Evaluate | No final score | Pending/incomplete |
| LAB-V030 | Evaluation for wrong CodeSubmission | Evaluate | Reject correlation | Integrity error |
| LAB-V031 | Evaluation score out of range | Calculate | Reject/flag | Validation error |
| LAB-V032 | maxScore invalid/zero | Calculate | No final score until resolved | TBD |
| LAB-V033 | Unknown verdict/infra failure | Calculate | No fabricated score | Operational state |
| LAB-V034 | Unconfigured late/override penalty | Calculate | Do not apply | None |
| LAB-V035 | Foreign LabSubmission | Student read | Deny/not-found-safe | Access denied |
| LAB-V036 | Pending/missing result | Student read | Show pending/incomplete | TBD |
| LAB-V037 | Staff filter foreign Student | Staff read | Deny/scope-filter | Access denied |
| LAB-V038 | Export without permission | Staff read | Deny | TBD |
| LAB-V039 | Feedback target foreign | Feedback | Deny | Access denied |
| LAB-V040 | Feedback too long | Feedback | Reject | Limit TBD |
| LAB-V041 | Feedback blank/clear | Feedback | Explicit clear/no-op policy | TBD |
| LAB-V042 | Override reason missing | Override | Reject if required | TBD |
| LAB-V043 | No override permission | Override | Deny | Access denied |
| LAB-V044 | Concurrent score update | Override | Conflict/review | TBD |
| LAB-V045 | Invalid deadline timestamp | Date | Reject | Date validation |
| LAB-V046 | Submit after deadline | Submit | Approved late policy | TBD |
| LAB-V047 | dueDate edited after submission | Recompute | Preserve historical policy | TBD |
| LAB-V048 | Group inactive | Eligibility | Deny/policy-defined | TBD |
| LAB-V049 | Student not enrolled | Eligibility | Deny/policy-defined | TBD |
| LAB-V050 | Group membership revoked | Eligibility | Recheck access | TBD |
| LAB-V051 | Cross-College/Department | Eligibility | Deny unless explicitly allowed | Access denied |
| LAB-V052 | No LabSubmission | Progress | Approved not-submitted state | TBD |
| LAB-V053 | Completion threshold unknown | Progress | Do not infer completion | TBD |
| LAB-V054 | Unauthorized rejudge | Rejudge | Deny | Access denied |
| LAB-V055 | Rejudge already active | Rejudge | Serialize/reject | TBD |
| LAB-V056 | Incomplete rejudge | Rejudge | No false final score | Incomplete |
| LAB-V057 | Event from failed operation | Event | No success event | Internal |
| LAB-V058 | Duplicate event | Event | Idempotent | Internal |

## 30. Data and state reference

### 30.1 Conceptual source fields

**Course:** id, collegeId, departmentId, createdById, title varchar(255), description text, code varchar(50), scope ContentScope, status PublishStatus, createdAt, updatedAt.

**CourseSection:** courseId, sectionId, createdAt.

**CourseEnrollment:** id, courseId, studentId, enrolledAt, completedAt.

**LabAssignment:** id, courseId, problemId, title varchar(255), description text, scope ContentScope, maxScore int, dueDate timestamp, createdAt, updatedAt.

**LabSubmission:** id, labAssignmentId, studentId, codeSubmissionId, score decimal(5,2), feedback text, submittedAt.

**Group:** id, collegeId, departmentId, createdById, name varchar(255), description, inviteCode varchar(20), isActive, createdAt, updatedAt.

**GroupMember:** id, groupId, studentId, joinedAt.

**GroupLab:** groupId, labAssignmentId, createdAt.

### 30.2 Schema omissions relevant to Lab

The conceptual source does not list on LabAssignment/LabSubmission: Lab status/publishStatus, creator/publishedAt/archiveAt, availability start/close, late policy/grace/penalty/extension, attempt limit/number/final selector, Problem/testcase snapshot, LabSubmission status/verdict/isLate/evaluation generation, score override provenance/history, feedback updatedAt, Lab completion record, score formula version.

Absence from the conceptual source does not prove absence from implementation. Any target addition must be confirmed later.

### 30.3 Relationships and cardinality

- Course 1 → many LabAssignment.
- Problem 1 → many LabAssignment (schema ERD).
- LabAssignment 1 → many LabSubmission.
- User/Student 1 → many LabSubmission.
- Group ↔ LabAssignment via GroupLab.
- Course ↔ Section via CourseSection.
- Course ↔ Student via CourseEnrollment.
- CodeSubmission ↔ LabSubmission relationship is shown in the ERD and LabSubmission has codeSubmissionId. Exact cardinality (1:1, 1:N, N:1) is unclear and remains LAB-D25.

### 30.4 Enums and state caveat

ContentScope: DEPT_WIDE, SECTION_ONLY, GLOBAL.

PublishStatus: DRAFT, PUBLISHED, ARCHIVED.

Course has status: PublishStatus. LabAssignment does not list a status field. The earlier proposed lifecycle Draft → Published → Closed → Archived is a recommendation; Closed is not a source enum. State transitions/reopening/deletion belong in Module 17 after review.

## 31. Cross-module dependencies and integration boundaries

| Module | Lab dependency | Lab responsibility | Not owned here |
|---|---|---|---|
| 4 Course & Enrollment | Course/CourseSection/CourseEnrollment/eligibility | Validate Course and use eligibility context | Enrollment lifecycle |
| 5 Problem Management | Problem/content/testcases | Associate Problem | Problem CRUD/version |
| 6 Code Submission & Judge0 | CodeSubmission, language, execution | Carry Lab context and link CodeSubmission | Execution worker/API/token lifecycle |
| 7 Test Case Evaluation | Verdict, testcase score, hidden-case handling | Consume finalized evaluation for Lab scoring | Evaluation verdict/score semantics |
| 10 Groups | Group/GroupMember/GroupLab | Assignment/membership lookup | Group lifecycle/membership changes |
| 13 Notifications | Lab lifecycle/grading events | Identify candidate events | Delivery/template/recipient policy |
| 15 Audit Logging | Lab CRUD/score/feedback actions | Emit relevant context | Canonical audit schema/retention |
| 17 State Machines | Lab lifecycle/submission/evaluation | Provide Lab states/transitions | Canonical state diagrams |
| 18 Failure Handling | Judge0/persistence/event recovery | Lab consequences | Platform recovery framework |

Candidate notification events (not approved): Lab published, assignment changed, dueDate changed, Lab archived, submission received, evaluation completed, score available/changed, feedback added, override, rejudge. Recipient/channel/timing/dedup belongs to Module 13.

Candidate audit actions (not approved): create/update/publish/archive, GroupLab assignment, feedback edit, manual override if enabled, rejudge, sensitive staff grading access. Exact names/retention/payload belong to Module 15.

## 32. Edge cases and failure handling

| ID | Scenario | Required treatment / unresolved behavior |
|---|---|---|
| LAB-E01 | Student removed from Group | Recheck current access; preserve historical submission; access policy TBD |
| LAB-E02 | Student joins Group after Lab publication | Eligibility inheritance TBD |
| LAB-E03 | Group membership changes before deadline | Current vs snapshot membership TBD |
| LAB-E04 | Lab unpublished after submission | Preserve submission/score; access policy TBD |
| LAB-E05 | dueDate changes after submission | Do not silently rewrite historical late status |
| LAB-E06 | Problem edited after publication | Problem version/snapshot TBD |
| LAB-E07 | Problem deleted/archived | Preserve historical result; display policy TBD |
| LAB-E08 | Judge0 finishes after deadline | Late status should use accepted submission timestamp under derived rule |
| LAB-E09 | Judge0 unavailable | Operational recovery; not Student failure |
| LAB-E10 | Duplicate submit/network retry | Avoid unintended duplicate attempt |
| LAB-E11 | maxScore changes after grading | Recalculation/history policy TBD |
| LAB-E12 | Manual override | Preserve automated score/provenance if feature approved |
| LAB-E13 | Rejudge changes evaluation | Recompute only under approved policy |
| LAB-E14 | Student loses Course enrollment | Historical access vs eligibility TBD |
| LAB-E15 | Lab archived | Block normal submission; retain history |
| LAB-E16 | CodeSubmission created but LabSubmission fails | Reconcile/retry; prevent orphan/duplicate |
| LAB-E17 | LabSubmission exists but evaluation enqueue fails | Retry evaluation without new Student attempt |
| LAB-E18 | Duplicate grading event | Idempotent score/progress update |
| LAB-E19 | Group outside staff scope | Deny assignment |
| LAB-E20 | Group members outside Course/Section | Eligibility precedence TBD |
| LAB-E21 | Student requests foreign LabSubmission | Deny/not-found-safe |
| LAB-E22 | Hidden testcase/diagnostic endpoint bypass | Server-side mask/deny |
| LAB-E23 | Manager Full CRUD vs V/C/U* | Preserve permission conflict; no implied Delete |
| LAB-E24 | Lab has no GroupLab link | Course/scope eligibility TBD |
| LAB-E25 | GLOBAL Lab scope with scoped Course | Scope inheritance conflict TBD |
| LAB-E26 | maxScore zero/negative | Validation decision; avoid divide-by-zero |
| LAB-E27 | Evaluation partial/incomplete | No final grade until approved policy |
| LAB-E28 | Feedback changed after final score | Audit/notification policy TBD |
| LAB-E29 | Problem changed after submission | Historic traceability required |
| LAB-E30 | Rejudge while manual override exists | Precedence/reconciliation TBD |

## 33. Decision log status

**LAB-D01–LAB-D38 are adopted.** No Module 8 product/architecture/security decision remains open. The complete decision register is ADOPTED-DECISIONS.md.

| Workstream | Status | Remaining work |
|---|---|---|
| Assignment/product model | Adopted | None |
| RBAC and academic scope | Adopted | Implement server enforcement |
| Group assignment | Adopted | Integrate Module 10 GroupLab writes/membership |
| Problem versioning | Adopted | Implement revision pinning/migration |
| Submission/evaluation | Adopted | Integrate Modules 6/7 and outbox |
| Score/attempt policy | Adopted | Implement calculation/history/selection |
| Lifecycle/deadline/progress | Adopted | Integrate Module 17; implement derived views |
| Feedback/manual grading | Feedback adopted; manual override disabled | Implement feedback; no score override |
| Audit/notifications/recovery | Module ownership adopted | Integrate Modules 13/15/18 |
| QA | 188 cases defined | Execute and record evidence |

## 34. Module-level acceptance criteria

- [ ] LabAssignment is associated with valid Course and Problem.
- [ ] Source fields/enums are captured without assuming undocumented fields.
- [ ] Staff Lab list/create/update operations enforce role and scope on server.
- [ ] Manager Delete discrepancy is not silently resolved.
- [ ] GroupLab assignment validates both Lab and Group scope.
- [ ] Student My Labs/Assignments shows only eligible Labs.
- [ ] Student detail shows correct Problem and protects hidden testcase data.
- [ ] Student submission links to correct Lab/Student/Problem.
- [ ] Unauthorized submit does not create CodeSubmission/Judge0 execution.
- [ ] Lab evaluation reuses Modules 6/7 and distinguishes infrastructure errors.
- [x] Lab score formula, rounding, max score, attempt policy and late policy are adopted in LAB-D07–D09/D18/D29.
- [ ] Student LabSubmission access is owner-restricted.
- [ ] Staff performance access is College/Department/Section/Group scoped.
- [x] Manual score override is disabled in v1 (LAB-D22/D34).
- [ ] Rejudge preserves traceability and does not unintentionally create a Student attempt.
- [ ] Course completion is not conflated with Lab completion.
- [ ] Notification/audit contracts follow Modules 13/15 and avoid sensitive payloads.
- [ ] Decision-blocked QA is revisited in the all-module review.

## 35. Review/readiness checklist

**Source/product review — complete**
- [x] Business Flow v2 and original reviewed.
- [x] Page-to-DB Mapping v2 and original reviewed.
- [x] RBAC module/page/API matrices reviewed; Manager Delete and Finance broad View conflicts resolved conservatively.
- [x] Conceptual schema fields, relations and enums captured.
- [x] LAB-F001–F020 reviewed and traced to QA.
- [x] Course/Section/Group eligibility reconciled with Modules 4/10.
- [x] CodeSubmission/evaluation/scoring handoff aligned to Modules 6/7.
- [x] All LAB-D01–D38 adopted and recorded.

**Engineering/release — pending**
- [ ] Implement LabStatus, LabAssignmentRevision, LabSection and LabScoreHistory migrations.
- [ ] Implement scoped HOD/Manager/Finance/Student APIs and GroupLab integration.
- [ ] Integrate Module 6 LAB context/idempotency and one-to-one LabSubmission linkage.
- [ ] Integrate Module 7 final evaluation event and score conversion/rejudge reconciliation.
- [ ] Implement attempt allocation, best-score query, deadline policy, feedback and derived progress.
- [ ] Integrate Modules 13, 15, 17 and 18.
- [ ] Execute all 188 QA cases plus migration, security, concurrency and outbox recovery tests.

**Approval boundary:** product decisions are adopted; the target Lab subsystem, schema changes, UI, integrations and QA execution are not represented as implemented.

## 37. Adopted target contract and authority

Earlier source and legacy-audit sections preserve the evidence used during review. Where those sections contain RECOMMENDATION, TBD or OPEN labels, Sections 37–42 and ADOPTED-DECISIONS.md are the adopted v1 target and take precedence. The legacy audit is not evidence that the target Lab subsystem exists.

## 38. Normative Lab domain and lifecycle

### Assignment and version model
- A LabAssignment belongs to exactly one Course and one Problem. Group assignment is optional and may reference multiple Groups through GroupLab.
- A published LabAssignment creates an immutable LabAssignmentRevision. Each revision pins one published ProblemVersion and snapshots Course/Problem association, title/description, scope, maxScore, availability/dueDate, late policy, attempt policy and publication time.
- Editing a published Lab creates a new revision. Existing LabSubmissions remain associated with their original revision and ProblemVersion. Changing maxScore, dueDate, scope, Problem or Group target never silently recalculates old scores.
- Problem and Lab must belong to the same College. The effective Student audience is the intersection of Course scope, Lab scope, Problem scope, Section/Group assignment and active enrollment.
- Add LabStatus: DRAFT, PUBLISHED, CLOSED, ARCHIVED. DRAFT is staff-only; PUBLISHED is visible to eligible Students; CLOSED is read-only for new attempts; ARCHIVED is historical-only. Restore ARCHIVED to DRAFT. Never hard-delete a Lab or cascade-delete academic history.

### Assignment and eligibility
- New submissions require an active Student account, active College/entitlement, PUBLISHED Course, CourseEnrollment.status=ENROLLED, active included Section, published Lab revision and published ProblemVersion.
- CourseEnrollment.COMPLETED and WITHDRAWN do not allow new attempts; own historical Lab records remain viewable subject to account/privacy policy. CourseEnrollment.completedAt is not Lab completion.
- If GroupLab rows exist for the current assignment, Student must be a member of at least one linked active Group. If none exist, eligible Course audience applies. Group membership never replaces Course enrollment.
- Joining an assigned Group after publication grants eligibility once all other conditions pass. Group removal/inactivation blocks new attempts immediately but does not delete the Student's prior submissions or scores.
- SECTION_ONLY uses explicit Section targets (LabSection where required); DEPT_WIDE includes active Sections in the owning Department; GLOBAL includes active Sections in the owning College and is SUPER_ADMIN-only. GLOBAL never means public or cross-College access.
- Manager can create/update only SECTION_ONLY Labs in assigned Section/permitted Group scope. HOD is limited to assigned College+Department. Finance is metadata-only within its College and cannot inspect Student academic records.

### Submission and attempts
- Lab submission is a Module 6 CodeSubmission with context=LAB and contextId=LabAssignment ID. Module 6 owns source, attempt intake, idempotency, SubmissionRun and JudgeRun; Module 8 owns the LabSubmission link and Lab score.
- One accepted CodeSubmission links to one LabSubmission (`codeSubmissionId` unique). Each Student attempt is a new CodeSubmission/LabSubmission pair. This resolves the conceptual ERD's CodeSubmission→many LabSubmission relation in favor of Module 6's one-context-per-CodeSubmission contract.
- Attempt numbers are allocated atomically per Student+LabAssignment. Attempts are unlimited by default; an optional positive maxAttempts applies to a published revision. Only accepted CodeSubmissions count. Same Idempotency-Key replay does not create a new attempt. Only one in-flight attempt per Student+Lab is allowed.
- Rejudge creates a new SubmissionRun only; it is not a Student attempt and does not create a new LabSubmission.

### Lifecycle transitions

| From | To | Guard/meaning |
|---|---|---|
| Create | DRAFT | Staff-only, required fields and scoped relations valid |
| DRAFT | PUBLISHED | HOD/SUPER_ADMIN; complete revision, ProblemVersion, audience and scoring config |
| PUBLISHED | CLOSED | HOD/SUPER_ADMIN; stop new submissions, preserve history |
| CLOSED | PUBLISHED | HOD/SUPER_ADMIN; reopen under a new/selected valid revision |
| DRAFT/PUBLISHED/CLOSED | ARCHIVED | HOD/SUPER_ADMIN; soft retirement, no new submissions |
| ARCHIVED | DRAFT | HOD/SUPER_ADMIN restore; new publish required |

## 39. Normative scoring, late policy and completion

### Score calculation

\`LabScore = roundHalfUp(EvaluationScorePercent × Revision.maxScore / 100, 2)\`

- EvaluationScorePercent and verdict come only from a complete Module 7 result. Module 8 does not re-run Judge0 or re-interpret verdicts.
- LabSubmission.score is nullable until final evaluation completes. A complete evaluation with 0% is a valid final score of 0. Infrastructure/unknown/incomplete evaluation has no Lab score and is not treated as zero.
- Persist the current attempt score in LabSubmission.score (DECIMAL(12,2)); append score changes to LabScoreHistory. Derive best/final Student score from finalized attempts.
- Highest valid finalized attempt is the best score. A tie selects the latest accepted attempt. Keep latest attempt and best attempt separately visible to authorized Student/staff.

### Deadline and late behavior
- dueDate is optional and stored/compared in UTC. If absent, deadline enforcement is not applied.
- Default is BLOCK_AFTER_DUE. Submission is on time when trusted server receivedAt <= dueDate; later submissions are rejected.
- Optional ACCEPT_NO_PENALTY accepts late submissions and stores isLate=true. No numeric penalty, grace period or individual extension exists in v1.
- Student-facing dates may be localized for display; client clock and Judge0 completion timestamp are never authoritative for lateness.
- A global dueDate change creates a new LabAssignmentRevision; existing attempts preserve their original timestamp/revision. Per-Student extensions are deferred.

### Completion and progress
- Derive NOT_STARTED, SUBMITTED, EVALUATION_PENDING, COMPLETED and EVALUATION_FAILED from LabSubmission/CodeSubmission/SubmissionRun. Do not create a Lab progress percentage or infer Lab completion from CourseEnrollment.completedAt.
- COMPLETED means at least one accepted attempt has a complete Module 7 evaluation, regardless of percentage. There is no passing threshold or manual completion override in v1.
- A pending or infrastructure-failed evaluation is not completion and has no final Lab score. A completed 0% result is still evaluated/completed.

## 40. Normative roles, feedback and privacy

| Role | Lab permissions and data |
|---|---|
| SUPER_ADMIN | Global Lab administration, lifecycle, scoped data inspection and privileged Module 6/7 rejudge/raw access subject to audit |
| HOD | Create/view/update/lifecycle control within College+Department; inspect scoped Student Lab performance and provide feedback |
| MANAGER | View/create/update within assigned Section/permitted Group, SECTION_ONLY only; cannot Delete/publish/close/archive/reopen; inspect scoped summaries and provide feedback |
| FINANCE_TEAM | College-scoped metadata-only View; no LabSubmission, Student identity/performance, score, feedback, source or raw evaluation |
| STUDENT | View eligible Labs, submit own attempts, view own history/score/feedback and Module 7 safe visible-test feedback only |

- No separate Faculty/Lab Staff role is introduced in v1. Page labels do not create RBAC roles.
- Manual score override is disabled. Lab CRUD/View does not imply score editing.
- Feedback is optional plain text up to 10,000 characters after trimming. HOD/Manager may add/edit within scope; only the Student owner may read it. Blank input clears feedback; changes are audited.
- Hidden testcase data, raw Judge0 transport/tokens, Student source code and other Students' private records never appear in ordinary Lab APIs/events. Raw inspection follows Module 6/7 audited SUPER_ADMIN policy.

## 41. Cross-module integration and ownership

| Module | Contract |
|---|---|
| 3 — Subscription & Seats | Owns entitlements/seats; Lab activity does not alter usedSeats or purchasedSeats |
| 4 — Course & Enrollment | Owns Course, CourseSection, CourseEnrollment, active Student/Section audience and Course status |
| 5 — Problem Management | Owns Problem CRUD, published ProblemVersion, content scope, language and resource limits |
| 6 — Code Submission & Judge0 | Owns CodeSubmission, LAB context, idempotency, attempts intake, SubmissionRun/JudgeRun and rejudge execution |
| 7 — Test Case Evaluation | Owns testcase execution results, verdict, evaluation percentage and hidden-data masking |
| 10 — Groups & Membership | Owns Group, GroupMember and GroupLab association writes; Module 8 consumes authorized links/membership |
| 13 — Notifications | Owns delivery, recipients, channels and reminder configuration for safe committed Lab events |
| 15 — Audit Logging | Owns canonical audit schema/retention for Lab lifecycle, feedback and score-history events |
| 17 — Functional State Machines | Owns canonical lifecycle transitions; Module 8 supplies Lab state requirements |
| 18 — Edge Cases & Failure Handling | Owns idempotent retry, partial failure, orphan reconciliation and outbox recovery |

### Submission/score event sequence
1. Module 8 validates Lab revision, Course enrollment, Group/Section scope, deadline, attempt policy and ProblemVersion.
2. Module 6 accepts the CodeSubmission (LAB context), atomically allocates the attempt and writes its SubmissionRun/outbox using Idempotency-Key.
3. Module 8 creates one LabSubmission link for the CodeSubmission in the shared transaction where possible; otherwise a durable accepted-submission outbox consumer creates it idempotently and Module 18 reconciles missing links.
4. Module 6/7 complete evaluation and commit SubmissionGraded only after final result commit.
5. Module 8 consumes the event idempotently, calculates Lab score from the pinned revision, updates LabSubmission.score and appends LabScoreHistory when a score changes.
6. Module 13 delivers only configured notifications; Module 15 audits required actions. Event payloads never include source, hidden testcase data or Judge0 tokens.

## 42. Schema delta, business rules and readiness

### Target schema delta
- LabAssignment: add LabStatus, creator/lifecycle timestamps, current revision pointer and optional availableFrom.
- LabAssignmentRevision: immutable revision number, LabAssignment ID, Course/Problem IDs, ProblemVersion ID, title/description, scope, maxScore, availableFrom, dueDate, latePolicy, maxAttempts and publication metadata.
- LabSection: Lab revision/assignment to explicit Section junction for SECTION_ONLY targeting where CourseSection alone is too broad.
- LabSubmission: add labAssignmentRevisionId, attemptNumber, isLate and trusted server submittedAt/acceptedAt; make score nullable DECIMAL(12,2); unique codeSubmissionId. Keep feedback as optional text.
- LabScoreHistory: append-only previous/new score, LabSubmission, evaluationRunId/sourceEventId, formula version, actor/service and timestamp.
- GroupLab remains a Module 10-owned junction; LabAssignmentRevision changes must not mutate prior LabSubmission revision references.

### Core validation rules
- Reject invalid/missing Course, Problem, title, positive maxScore, ContentScope, Group/Section or published ProblemVersion before Lab publish.
- Reject Student requests with foreign Student/Lab IDs, foreign College/Department/Section/Group, inactive Group, inactive/non-enrolled CourseEnrollment, unpublished Lab or ProblemVersion mismatch.
- Reject submission before availableFrom, after hard dueDate cutoff, after CLOSED/ARCHIVED, when maxAttempts reached, or while another attempt for the same Student/Lab is in flight.
- Return pending/incomplete for processing or infrastructure failures; never create a Lab score from null/unknown Module 7 results.
- Idempotent CodeSubmission/LabSubmission callbacks and SubmissionGraded consumption must not duplicate attempt, score history or notification side effects.
- Scope changes, Group removal, Course withdrawal and archive preserve historical submission/score records; they do not confer new submission eligibility.

### Release checklist
- [x] Source review and LAB-D01–LAB-D38 adoption complete.
- [x] 20 function IDs retained and aligned to the normative contract.
- [x] 188 target QA cases defined and uniquely identified.
- [ ] Schema migrations and indexes implemented.
- [ ] Scoped APIs/UI and lifecycle implemented.
- [ ] Modules 4–7 and 10 integration complete.
- [ ] Module 13/15/17/18 contracts implemented.
- [ ] Execute QA, migration, authorization, concurrency and recovery tests.