# 6. Code Submission & Judge0

**Document type:** Functional Specification  
**Version:** 0.2 — Detailed draft aligned to Functional Specification Standard v1.0  
**Status:** Detailed draft — ready for product/architecture/security review; target behavior remains unapproved  
**Scope:** Student code submission, DEZOLVER backend submission handling, Judge0 integration boundary, submission records, and result retrieval. Detailed test-case evaluation and scoring belong to Module 7 and relevant academic modules.

> This draft uses Business Flow v2, Page-to-DB Mapping v2, and the schema reconstruction. Source-confirmed fields and flow are distinguished from derived safeguards and unresolved decisions. Judge0 API details are not assumed where the source does not define them.

## 1. Purpose

Define how DEZOLVER receives a student's code submission, records it, hands it to Judge0 through the DEZOLVER backend, and makes the resulting submission status and execution result available to authorized users/workflows.

The source business flow establishes:

**Code Submission → DEZOLVER Backend → Judge0 → Evaluation → Result → Score**

This module focuses on submission handling and the Judge0 integration boundary. Evaluation logic, test-case comparison, score calculation, and academic ranking are specified in downstream modules.

## 2. Scope

### Included
- Student submission initiation
- Submission context and Problem reference
- CodeSubmission record creation and fields
- Backend-to-Judge0 integration boundary
- Judge0 token storage/reference
- Submission status/verdict representation
- Execution metadata and output/error fields
- Submission history and result retrieval
- Role and activity authorization safeguards
- Failure handling requirements and unresolved decisions
- Interfaces with Problems, Labs, Assessments, Contests, and Evaluation

### Excluded / owned by other modules

| Topic | Specification owner |
|---|---|
| Problem metadata, allowed languages, time/memory limits | Module 5 — Problem Management |
| Test-case execution, output comparison, final evaluation logic | Module 7 — Test Case Evaluation |
| Lab assignment, lab score aggregation | Module 8 — Labs & Lab Scoring |
| Assessment availability, timing, attempt rules, scoring | Module 9 — Assessments |
| Group membership and group-based eligibility | Module 10 — Groups |
| Contest eligibility, submission windows, penalty, score, rank | Module 11 — Contests |
| Notifications for submission/result events | Module 13 — Notifications |
| Audit event catalogue | Module 15 — Audit Logging |
| Canonical submission state machine | Module 17 — Functional State Machines |
| Detailed retry, timeout, and failure edge cases | Module 18 — Edge Cases & Failure Handling |

## 3. Source documents and interpretation

### 3.1 Business Flow v2

The overall flow shows:
- Student accesses a Course and Problem.
- Student writes code and submits.
- DEZOLVER handles the Code Submission.
- The DEZOLVER backend sends work to Judge0.
- Judge0 leads into Evaluation.
- Evaluation produces a Result and Score.

The Student flow also shows submissions in:
- Course/Problem workflow
- Group → Lab → Problem workflow
- Contest → Contest Problem → Participation workflow

The source diagram does not specify Judge0 API endpoints, request/response schema, synchronous/asynchronous behavior, retry strategy, or error mapping.

### 3.2 Page-to-DB Mapping v2

| Role | Page/workflow | Mapped entities include |
|---|---|---|
| HOD | Courses & Labs | LabSubmission, CodeSubmission, Problem, Course-related entities |
| HOD | Assessments | Assessment, AssessmentProblem, Problem, CodeSubmission, User |
| HOD | Problems | Problem, TestCase, CodeSubmission, User |
| HOD | Contests | Contest, ContestProblem, participation/ranking entities, Problem, CodeSubmission |
| Manager | Courses & Labs | LabSubmission, CodeSubmission, Problem, Course-related entities |
| Manager | Assessments | Assessment, AssessmentProblem, Problem, CodeSubmission, User |
| Manager | Problems | Problem, TestCase, CodeSubmission, User |
| Manager | Contests | Contest, ContestProblem, participation/ranking entities, Problem, CodeSubmission |
| Student | Assignments | LabAssignment, LabSubmission, CodeSubmission, Problem, Course |
| Student | My Labs | LabAssignment, LabSubmission, CodeSubmission, Problem, Course |
| Student | Practice Problems | Problem, TestCase, CodeSubmission, User |
| Student | Contests | Contest, ContestProblem, ContestParticipant, ContestScoreRank, Problem, CodeSubmission |
| Student | My Courses | Course, CourseProblem, Problem and enrollment entities |

The mapping confirms CodeSubmission is used in several academic contexts. It does not fully specify submission screen behavior or per-context restrictions.

### 3.3 Schema reconstruction

The schema defines CodeSubmission as:

| Field | Type |
|---|---|
| id | varchar |
| problemId | varchar |
| studentId | varchar |
| language | varchar(50) |
| code | text |
| judge0Token | varchar(255) |
| verdict | JudgeVerdict |
| executionTimeMs | int |
| memoryUsageKb | int |
| failedTestCaseIndex | int |
| stdout | text |
| stderr | text |
| submittedAt | timestamp |
| startedAt | timestamp |
| completedAt | timestamp |

The schema defines JudgeVerdict:
- PENDING
- RUNNING
- ACCEPTED
- WRONG_ANSWER
- TIME_LIMIT_EXCEEDED
- MEMORY_LIMIT_EXCEEDED
- RUNTIME_ERROR
- COMPILATION_ERROR

No additional verdicts or fields are assumed in this draft.

## 4. Roles and submission access

| Role | Source-described relationship | Submission access |
|---|---|---|
| SUPER_ADMIN | Platform administration | Submission inspection/administration permissions are not specified |
| FINANCE_TEAM | Billing/subscription workflows | No CodeSubmission responsibility is mapped |
| HOD | Courses & Labs, Problems, Assessments, Contests pages include CodeSubmission | Access to authorized academic scope; exact inspection/export rights TBD |
| MANAGER | Courses & Labs, Problems, Assessments, Contests pages include CodeSubmission | Must remain within assigned Section/activity scope; exact inspection rights TBD |
| STUDENT | Submits code for Problems in Courses, Labs, Practice, and Contests | May create and view own submissions in eligible contexts |

### 4.1 Derived authorization safeguards

- Server must derive authenticated Student identity; client-provided studentId must not be trusted.
- Submission must reference a valid Problem the Student can access in the selected activity context.
- Course, Lab, Assessment, and Contest eligibility must be validated server-side before accepting a submission.
- Student must not view another Student's source code, raw output, error output, or submission details unless an approved feature permits it.
- HOD/Manager submission visibility must be constrained by College/Department/Section and activity authorization.
- Client must not alter another user's submission by changing a submission ID.
- Judge0 credentials/configuration must remain server-side; the client must not directly invoke Judge0 with privileged credentials.

**TBD:** whether Students may resubmit freely, whether staff may run submissions on behalf of Students, and exact staff rights to source code and execution output.

## 5. Functional requirements

### SUB-001 — Create a CodeSubmission record

**Source-confirmed schema.**

A submission record includes:
- id
- problemId
- studentId
- language
- code
- judge0Token
- verdict
- executionTimeMs
- memoryUsageKb
- failedTestCaseIndex
- stdout
- stderr
- submittedAt
- startedAt
- completedAt

**Derived requirement:** a submission must be persisted and associated with its authenticated Student and Problem so its lifecycle/result can be tracked.

**TBD:**
- Whether record creation occurs before Judge0 handoff or only after successful handoff.
- Whether judge0Token is nullable before Judge0 accepts the request.
- Initial field values/defaults.
- Whether submission metadata includes activity/context IDs beyond problemId.
- Source-code retention rules.

### SUB-002 — Capture submitted source code

**Source-confirmed schema:** code is text.

Submitted source code is stored in CodeSubmission.

**TBD:** maximum code length/request payload size; normalization before storage/execution; encryption/retention policy; whether a submission is immutable after creation; and whether HOD/Manager can view code and under what conditions.

**Derived safeguard:** source code is untrusted input and must never be executed directly on the DEZOLVER application server.

### SUB-003 — Capture programming language

**Source-confirmed schema:** language is varchar(50).

The submission records the language used by the Student.

**Derived requirement:** submitted language must be compatible with Problem.allowedLanguages (Module 5) and platform Judge0 mapping.

**TBD:**
- Canonical stored values: language names, internal IDs, or Judge0 IDs.
- Language version selection.
- Behavior if a configured Judge0 language becomes unavailable.
- Normalization rules.

The language catalogue and Judge0 language mapping must be finalized in implementation design. Problem's allowed-language policy is defined in Module 5.

### SUB-004 — Validate submission eligibility

**Derived requirement from business context and access controls.**

Before accepting a submission, DEZOLVER should verify:
- Student is authenticated and active.
- Problem exists and is accessible.
- Student is permitted to submit in the selected Course/Lab/Assessment/Contest/Practice context.
- Language is permitted for the Problem and activity.
- Activity-level submission restrictions are satisfied.
- Source code and request payload satisfy approved size/format limits.

**TBD:** exact checks, attempt limits, deadlines, cooldowns, and context-specific eligibility rules. These are primarily owned by Modules 8–11.

### SUB-005 — Initialize submission verdict

**Source-confirmed enum:** JudgeVerdict includes PENDING and RUNNING.

**Proposed/derived behavior:** a newly accepted submission begins in a pending/queued state, then enters running when execution starts.

**TBD:**
- Whether initial persisted verdict is always PENDING.
- Whether RUNNING is set by DEZOLVER or inferred from Judge0.
- Whether a separate queue state is required.
- Whether record creation and state transition are atomic.
- Whether failed handoff leaves a record PENDING or requires an operational error representation.

Do not add enum values without schema/product approval.

### SUB-006 — Submit code from DEZOLVER backend to Judge0

**Source-confirmed flow:** DEZOLVER backend sends Code Submission to Judge0.

The backend is the integration boundary.

**Derived integration safeguards:**
- Judge0 requests are initiated server-side.
- Judge0's returned token/reference is associated with the relevant CodeSubmission.
- Submission ownership and Problem context are retained across the external execution request.
- Backend maps DEZOLVER language/resource settings to Judge0's accepted format.
- Backend does not expose privileged Judge0 credentials to Students.

**TBD:**
- Judge0 deployment type (hosted vs self-hosted).
- API endpoint(s), request body, headers, authentication.
- Synchronous vs asynchronous submission.
- Callback/webhook, polling, or other result retrieval mechanism.
- Base URL, credentials, environment configuration.
- Batch submission vs one request per CodeSubmission.
- Whether Judge0 receives DEZOLVER test cases directly or evaluation is orchestrated separately.
- Callback URL or integration metadata, if applicable.

### SUB-007 — Store Judge0 reference

**Source-confirmed schema:** judge0Token varchar(255).

The CodeSubmission record stores a Judge0 token/reference.

**Derived requirement:** token/reference must be associated with the correct submission and used to correlate later execution/result retrieval.

**TBD:** token uniqueness/database constraints, nullability before handoff, expiration/result retention, sensitivity, reuse rules, and duplicate-job prevention.

### SUB-008 — Track execution timestamps

**Source-confirmed schema:** submittedAt, startedAt, completedAt.

Interpretation:
- submittedAt records submission time.
- startedAt records execution start.
- completedAt records completion.

**TBD:** timestamp authority (backend vs Judge0-derived); nullability; whether submittedAt means DEZOLVER receipt or Judge0 acceptance; timezone convention; and behavior when execution never starts or no completion result is received.

### SUB-009 — Store execution result metadata

**Source-confirmed schema:** verdict, executionTimeMs, memoryUsageKb, failedTestCaseIndex, stdout, stderr.

These fields capture verdict and execution-related information.

**TBD:**
- Which values come from Judge0 vs DEZOLVER Evaluation.
- Whether executionTimeMs and memoryUsageKb are totals, maxima, or per-test values.
- Units and rounding conventions.
- Whether stdout/stderr are complete, truncated, filtered, or stored only for selected verdicts.
- Whether failedTestCaseIndex is zero-based or one-based.
- Whether it refers to the first failed test or another failure.
- Whether Student response exposes raw stderr/stdout or sanitized/limited output.
- Whether failedTestCaseIndex applies to compilation/runtime errors.

Detailed test-by-test semantics belong to Module 7.

### SUB-010 — Support submissions from multiple activity contexts

**Source-confirmed business flow/page mapping:** CodeSubmission is used in Courses, Labs, Practice Problems, and Contests. Page-to-DB Mapping also associates CodeSubmission with HOD/Manager Assessments pages.

**Derived requirement:** submission service validates activity context before accepting a submission.

**TBD:**
- Whether CodeSubmission needs explicit courseId, labAssignmentId, assessmentId, contestId, or other context fields.
- Whether context is stored on CodeSubmission or in a separate association.
- Whether one submission can count toward more than one activity.
- Whether activity-specific rules are stored with the submission.
- Whether assessment submission context is supported by the current schema.
- Whether contest submissions share the same CodeSubmission record/lifecycle as practice submissions.

The schema reconstruction shows problemId and studentId as direct academic references; context linkage is not explicitly shown in those fields.

### SUB-011 — Retrieve submission status and result

**Derived requirement:** Student can retrieve status/result for own submissions; authorized staff can inspect submissions within their permitted academic scope.

**TBD:**
- Automatic refresh vs manual refresh.
- Detail page vs inline result display.
- Fields shown for each verdict.
- Historical submission sorting/pagination.
- Submission comparison or code-diff features.
- Staff filtering by Student, Problem, verdict, language, and date.
- Export/download.
- Immediate result visibility vs activity-close visibility.

### SUB-012 — Preserve submission ownership and history

**Source-confirmed schema:** submission stores studentId and submittedAt.

**Derived requirement:** each submission retains its Student association and unique identity.

**TBD:** Student/staff deletion rights; retention after Student deactivation; history after Problem archival/deletion; rejudge updates existing vs creates new record; immutability; and staff-action audit behavior.

### SUB-013 — Handle duplicate/repeated Submit requests

**Decision required / not source-defined.**

Confirm:
- Whether repeated clicks/retries create separate submissions.
- Whether API supports an idempotency key.
- Whether client request identifier is persisted.
- Whether duplicate Judge0 jobs are prevented if network response is lost.
- Whether identical code submitted multiple times counts as separate attempts.

Do not assume idempotency behavior without confirmation.

### SUB-014 — Submission concurrency and rate controls

**Decision required / not source-defined.**

Confirm:
- Maximum concurrent submissions per Student.
- Rate limits per Student, College, Problem, or activity.
- Queue priority across Practice, Labs, Assessments, and Contests.
- Cancellation of pending/running submissions.
- Execution quota/Judge0 capacity limits.
- Retry-after or other rate-limit response behavior.

### SUB-015 — Restrict result/output disclosure

**Derived security safeguard.**
- Student result APIs enforce submission ownership.
- Hidden TestCase input/expectedOutput is not returned through submission detail/result endpoints.
- Raw Judge0 output/stderr is filtered according to approved feedback policy.
- Staff access to source code/output follows academic scope.
- Judge0 integration errors do not disclose credentials, internal hostnames, or infrastructure secrets.

**TBD:** exact result payload per verdict and whether source code/stdout/stderr are visible to Student, HOD, Manager, or platform administrators.

---

## 6. High-level workflows

### 6.1 Student submits code — source-grounded outline

1. Student opens a Problem in an eligible Course, Practice Problems page, Lab, Assessment, or Contest.
2. Student writes/selects code and language.
3. Student triggers Submit.
4. DEZOLVER validates Student identity, Problem access, language, and activity eligibility.
5. DEZOLVER creates/persists a CodeSubmission according to the approved creation strategy.
6. DEZOLVER backend sends the submission to Judge0.
7. DEZOLVER stores Judge0 token/reference.
8. Judge0 processes code.
9. Evaluation determines result.
10. Submission status/result is persisted and made available to authorized users.
11. Activity-specific modules consume result data for Lab score, Assessment result, Contest score/rank, or Course/practice history.

Steps 4–11 are a detailed proposed decomposition of the source diagram; transaction boundaries and integration mechanics are not source-defined.

### 6.2 Result retrieval — conceptual outline

1. Student requests a submission or refreshes status.
2. Backend verifies ownership and activity access.
3. Backend retrieves stored status/result, or synchronizes with Judge0 if that mechanism is approved.
4. Backend returns only fields authorized for the caller.
5. Student UI displays current verdict and permitted execution details.

**TBD:** whether result retrieval triggers Judge0 polling or a background worker updates CodeSubmission.

### 6.3 Staff submission inspection — conceptual outline

1. HOD/Manager opens a mapped page such as Problems, Courses & Labs, Assessments, or Contests.
2. Backend validates College/Department/Section and activity scope.
3. Authorized CodeSubmission records are retrieved.
4. Staff sees permitted status/result/source details.

Exact inspection capabilities and fields are not defined in source mapping.

---

## 7. Data model reference

### 7.1 CodeSubmission

| Field | Schema type | Meaning |
|---|---|---|
| id | varchar | Submission identifier |
| problemId | varchar | Problem reference |
| studentId | varchar | Student reference |
| language | varchar(50) | Submitted programming language |
| code | text | Source code |
| judge0Token | varchar(255) | Judge0 token/reference |
| verdict | JudgeVerdict | Submission verdict/state |
| executionTimeMs | int | Execution time in milliseconds |
| memoryUsageKb | int | Memory usage in KB |
| failedTestCaseIndex | int | Failed test-case index |
| stdout | text | Standard output |
| stderr | text | Standard error |
| submittedAt | timestamp | Submission time |
| startedAt | timestamp | Execution start time |
| completedAt | timestamp | Completion time |

### 7.2 JudgeVerdict

Source-confirmed values:

| Value | Meaning |
|---|---|
| PENDING | Initial/waiting state; exact semantics TBD |
| RUNNING | Execution in progress |
| ACCEPTED | Accepted verdict |
| WRONG_ANSWER | Wrong-answer verdict |
| TIME_LIMIT_EXCEEDED | Time-limit verdict |
| MEMORY_LIMIT_EXCEEDED | Memory-limit verdict |
| RUNTIME_ERROR | Runtime-error verdict |
| COMPILATION_ERROR | Compilation-error verdict |

Enum values are source-confirmed. Transition rules, terminal-state behavior, and mapping from Judge0 native statuses are TBD.

### 7.3 Related entities

- Problem — Problem definition and constraints (Module 5)
- TestCase — input/expected output and hidden/visible classification (Module 5)
- LabSubmission — Lab-specific submission tracking (Module 8)
- AssessmentProblem / Assessment — Assessment context (Module 9)
- ContestProblem / ContestParticipant / ContestScoreRank — Contest context (Module 11)
- User — Student identity and authorization (Module 1)

The schema reconstruction does not show direct CodeSubmission foreign-key fields to LabSubmission, Assessment, Contest, Course, or Group. Context association must be confirmed.

---

## 8. Integration and validation requirements

These are derived requirements; implementation details remain TBD.

| Rule ID | Requirement | Status |
|---|---|---|
| SUB-V01 | Submission references a valid Problem. | Derived |
| SUB-V02 | studentId comes from authenticated identity. | Derived security safeguard |
| SUB-V03 | Submission is permitted in selected academic context. | Derived; activity rules TBD |
| SUB-V04 | Submitted language is allowed for Problem/activity. | Derived; mapping TBD |
| SUB-V05 | Source code/request size satisfies approved limits. | Limit TBD |
| SUB-V06 | Judge0 credentials are never exposed to client. | Security safeguard |
| SUB-V07 | Judge0 reference is correlated to correct CodeSubmission. | Derived |
| SUB-V08 | Judge0 result updates cannot modify another submission. | Derived |
| SUB-V09 | Verdict uses schema JudgeVerdict enum unless change approved. | Source enum |
| SUB-V10 | Execution metadata uses agreed units/interpretation. | Exact behavior TBD |
| SUB-V11 | Student result access enforces submission ownership. | Derived |
| SUB-V12 | Hidden TestCase data is not exposed through result APIs. | Security safeguard |
| SUB-V13 | Staff submission access enforces RBAC and academic scope. | Derived |
| SUB-V14 | Duplicate request/retry behavior is explicitly defined. | TBD |
| SUB-V15 | Submission context is retained for activity-specific scoring/reporting. | Context linkage TBD |

---

## 9. Source gaps, conflicts, and risks

### 9.1 Judge0 integration contract is absent

Business Flow shows backend-to-Judge0 handoff but does not define API endpoint, authentication, payload, callback/polling, or deployment model.

**Decision:** establish Judge0 integration contract, environment configuration, and response mapping.

### 9.2 CodeSubmission context fields may be incomplete

Schema shows problemId and studentId but no explicit Course, Lab, Assessment, Contest, Group, or assignment context fields.

**Risk:** score aggregation and activity history may not identify the activity without an additional association.

**Decision:** confirm whether context is stored on CodeSubmission, in link tables, or inferred from existing relationships.

### 9.3 JudgeVerdict lacks infrastructure/queue failure states

Source enum includes PENDING, RUNNING, ACCEPTED, WRONG_ANSWER, TIME_LIMIT_EXCEEDED, MEMORY_LIMIT_EXCEEDED, RUNTIME_ERROR, and COMPILATION_ERROR.

**Risk:** Judge0 outage, submission rejection, queue timeout, or lost result may not map cleanly.

**Decision:** add operational states only if approved, or represent infrastructure failure separately. Do not silently expand the enum.

### 9.4 PENDING/RUNNING lifecycle is unspecified

Schema has verdict and timestamps but no transition rules.

**Decision:** define state/timestamp assignment and authoritative system.

### 9.5 Result feedback may expose hidden information

Schema stores stdout, stderr, and failedTestCaseIndex.

**Risk:** detailed output could leak hidden TestCase information or infrastructure details.

**Decision:** define verdict-specific Student/staff payloads and output sanitization.

### 9.6 Submission retries/idempotency are unspecified

**Risk:** network retry or repeated Submit could create multiple CodeSubmission records and Judge0 jobs.

**Decision:** define idempotency, retry policy, and duplicate handling.

### 9.7 Historical submissions and rejudging are unspecified

**Risk:** Problem/TestCase or Judge0 configuration changes could affect historical result interpretation.

**Decision:** define rejudge behavior, record versioning, and result retention with Modules 5, 7, and 15.

---

## 10. Decisions required before finalization

| ID | Decision | Why it matters |
|---|---|---|
| SUB-D01 | Confirm Judge0 deployment (hosted/self-hosted) and environments. | Infrastructure |
| SUB-D02 | Specify Judge0 API endpoints, authentication, request/response schema. | Integration |
| SUB-D03 | Choose synchronous/asynchronous execution and polling/callback/result-update mechanism. | Lifecycle |
| SUB-D04 | Define DEZOLVER language identifiers and mapping to Judge0 language/version IDs. | Compatibility |
| SUB-D05 | Define CodeSubmission creation timing and initial verdict. | Persistence |
| SUB-D06 | Define judge0Token nullability, uniqueness, expiration, and sensitivity. | Correlation |
| SUB-D07 | Define context association fields/tables for Course, Lab, Assessment, Contest, Practice. | Attribution |
| SUB-D08 | Define submission validation and code/request size limits. | API/security |
| SUB-D09 | Define per-Student/activity submission limits, rate limits, and concurrency. | Capacity/fairness |
| SUB-D10 | Define duplicate submission and idempotency behavior. | Reliability |
| SUB-D11 | Define retry policy for transient backend/Judge0 errors. | Reliability |
| SUB-D12 | Define Judge0 timeout, queue timeout, and result retrieval timeout. | Reliability |
| SUB-D13 | Define mapping of Judge0 statuses/verdicts into JudgeVerdict. | Verdict semantics |
| SUB-D14 | Decide how infrastructure failures are represented without penalizing Student code. | Correctness |
| SUB-D15 | Define executionTimeMs, memoryUsageKb, failedTestCaseIndex semantics/units. | Result consistency |
| SUB-D16 | Define stdout/stderr visibility, truncation, sanitization, and retention. | Privacy/security |
| SUB-D17 | Define Student/HOD/Manager source-code and result access rights. | RBAC |
| SUB-D18 | Define whether Students can cancel pending/running submissions. | UX/processing |
| SUB-D19 | Define submission deletion/retention and behavior after Student deactivation. | Data lifecycle |
| SUB-D20 | Define rejudge behavior and overwrite vs new record. | History |
| SUB-D21 | Define one Judge0 job per submission vs evaluation batch. | Architecture |
| SUB-D22 | Define whether Judge0 runs individual tests or DEZOLVER orchestrates them. | Module 7 boundary |
| SUB-D23 | Define source code/output encryption, redaction, and retention. | Data security |
| SUB-D24 | Define audit events for submit, handoff, result update, rejudge, and staff access (Module 15). | Audit |
| SUB-D25 | Define notification/status refresh behavior for completed submissions (Module 13). | UX/integration |

---

## 11. Acceptance checklist

Code Submission & Judge0 is implementation-ready when:
- [ ] Judge0 deployment and API contract are documented.
- [ ] Submission creation timing and initial verdict are confirmed.
- [ ] Language identifiers and Judge0 mapping are finalized.
- [ ] Submission context is linked to Course/Lab/Assessment/Contest/Practice.
- [ ] Eligibility checks are defined per activity type.
- [ ] Judge0 token/reference correlation is reliable.
- [ ] Pending/running/completed lifecycle and timestamps are agreed.
- [ ] Judge0 status/verdict mapping is approved.
- [ ] Infrastructure failures do not incorrectly penalize Student code.
- [ ] Retry, idempotency, rate-limit, and concurrency behavior is defined.
- [ ] Execution metadata units and failedTestCaseIndex semantics are agreed.
- [ ] Student/staff result visibility and source-code permissions are defined.
- [ ] Hidden TestCase information cannot leak through result APIs.
- [ ] Rejudge and historical submission behavior is approved.
- [ ] Module 7 evaluation and Modules 8–11 academic context interfaces are agreed.
- [ ] Audit and notification requirements are coordinated.

---

## 12. Dependencies and interfaces

- **Module 1 — Authentication & User Lifecycle:** authenticated Student identity, active status, role, sessions.
- **Module 2 — College / Department / Section Management:** organizational scope and academic eligibility.
- **Module 4 — Course & Enrollment:** Course access and Course-related Problem submission context.
- **Module 5 — Problem Management:** Problem metadata, allowed languages, resource limits, visibility.
- **Module 7 — Test Case Evaluation:** test execution, expected-output comparison, verdict determination, result interpretation.
- **Module 8 — Labs & Lab Scoring:** Lab submission association and score aggregation.
- **Module 9 — Assessments:** assessment eligibility, deadlines, attempt rules, result integration.
- **Module 10 — Groups:** group membership and group-specific eligibility/context.
- **Module 11 — Contests:** contest submission window, eligibility, scoring, penalties, ranking.
- **Module 13 — Notifications:** submission/result notifications if required.
- **Module 15 — Audit Logging:** submission and judge lifecycle events.
- **Module 17 — Functional State Machines:** canonical CodeSubmission lifecycle.
- **Module 18 — Edge Cases & Failure Handling:** retries, timeouts, Judge0 outage, duplicate requests, stale results, concurrency.

---

## 13. Source references

Repository source-text documents:
- [Business Flow v2.md](../../docs/source-text/Business%20Flow%20v2.md)
- [Page - DB Mapping v2.md](../../docs/source-text/Page%20-%20DB%20Mapping%20v2.md)
- [scheema.md](../../docs/source-text/scheema.md)

Original Drive PDFs:
- [Business Flow v2.pdf](https://drive.google.com/file/d/1rE9q1h0u9FTVLJmlP2Wl7WWfKFqsETgL/view)
- [Page - DB Mapping v2.pdf](https://drive.google.com/file/d/1UdzowkDPP85PNI0-GQ2k4cEmwIIAZYcw/view)
- [scheema.pdf](https://drive.google.com/file/d/10_LGfTsQ2eGmbJM5tQE9Pkmlwhft1lAI/view)

**Review gate (superseded by the later expansion):** The original source-based draft paused before Module 7. The user has since explicitly authorized Modules 6 and 7 to be upgraded together. Both detailed drafts are now repository-included and await Product/Architecture/Security review; unresolved target decisions remain open.


---

# Functional Specification Standard v1.0 compliance expansion

> This expansion supplements SUB-001–SUB-015 above. The original requirements remain source anchors. Stable function IDs below use SUBMIT-Fxxx. This document distinguishes the conceptual schema, business/RBAC sources, and supplied legacy code forensic report. Legacy audit findings are not target requirements unless separately approved.

## 14. Evidence and specification status

- **Source-confirmed:** stated in Business Flow, Page-to-DB Mapping, RBAC, or conceptual schema.
- **Legacy-verified (audit-reported):** described by the supplied source-code forensic investigation. The implementation is not re-inspected as part of this documentation task.
- **Derived:** required safeguard or workflow implication from confirmed source facts.
- **Target requirement:** not approved unless explicitly noted as such by the product owner.
- **Recommendation:** proposal only.
- **TBD / Decision required:** source gaps or conflicts requiring product/architecture/security approval.

The forensic report covers a specific legacy v2 code snapshot. Its findings may not describe the intended DEZOLVER target release. Do not equate legacy code behavior with target requirements.

## 15. Terminology and ownership boundaries

| Term | Meaning | Evidence/status |
|---|---|---|
| Code Submission | Conceptual workflow/entity representing submitted code associated with Problem and Student. | Business Flow + schema |
| Legacy Submission | Actual legacy model reported as assessment.submissions, not the conceptual CodeSubmission entity. | Audit-reported |
| Submission attempt | One formal Student submission. Legacy code increments attemptNumber inline; conceptual schema has no attempt number. | Conflict |
| Activity context | Course, Practice Problem, Lab, Assessment, Contest, or Group-related workflow that permits submission. | Business Flow/Page mapping |
| Judge0 token | External execution correlation reference. Conceptual schema has one judge0Token; legacy evaluation report says a separate token is returned per testcase. | Conflict |
| Formal Submit | Persisted submission with eligibility/attempt handling and evaluation. | Legacy audited flow |
| Run Code / scratchpad | Separate custom-input execution without formal submission/attempt side effects. | Reported absent in legacy code; target TBD |
| Evaluation | Testcase execution, comparison, verdict and score aggregation. | Module 7 |
| Activity score | Lab/Assessment/Contest/Course progress derived from a submission result. | Owning academic module |

### Module ownership
- Module 6 owns submission intake, actor/context validation boundary, persisted submission identity, execution job handoff/correlation, submission status/result transport, and submission history access.
- Module 7 owns testcase set selection/execution semantics, testcase result mapping, comparator, scoring, verdict aggregation, and evaluation result integrity.
- Modules 8–11 own activity eligibility and activity-specific score/attempt/deadline/contest rules.
- Module 17 owns the canonical lifecycle/state machine; Module 18 owns detailed recovery and edge-case policies.

## 16. Role, action, and scope matrix

| Role | Conceptual RBAC permission | Scope | Submission behavior |
|---|---|---|---|
| SUPER_ADMIN | View/Manage Code Submissions | Global | Administrative inspection/manage; precise source/raw execution access TBD |
| FINANCE_TEAM | No Code Submission grant in platform/API matrix | Assigned College for Finance duties, but no submission permission | Deny unless a new explicit grant is approved |
| HOD | View | College + Department | Inspect submissions in authorized academic scope; raw code/output access needs field-level decision |
| MANAGER | View* | College + Department + assigned Section/permitted Group | Scope-limited inspection; field-level access TBD |
| STUDENT | Create / View own | Own submissions + permitted academic resources | Submit only when Problem/activity eligibility passes; view own records |

**Source conflict/interpretation**
- RBAC platform matrix: SUPER_ADMIN View/Manage; HOD View; MANAGER View*; STUDENT Create/View own; FINANCE_TEAM —.
- Page/API matrices also show HOD V*, Manager V*, Student CREATE/V OWN.
- The RBAC *scope marker* restricts data to the actor’s allowed academic scope.
- Page visibility does not authorize backend access. Every API must enforce role, scope, and ownership.
- Staff access to source code, stdout, stderr, compiler output, hidden-test metadata, raw Judge0 response, or rejudge is not fully specified by the conceptual RBAC matrix.

## 17. Functional inventory

| Function ID | Function | Primary actor | Entry point / trigger | Type | Status |
|---|---|---|---|---|---|
| SUBMIT-F001 | Initiate formal code submission | Student | Coding workspace / Submit | Workflow | Draft |
| SUBMIT-F002 | Authenticate and authorize submission | System | Submit request | Security | Derived |
| SUBMIT-F003 | Validate activity and Problem eligibility | System + activity owner | Submit request | Eligibility | Draft |
| SUBMIT-F004 | Validate code and language | System | Submit request | Validation | Draft |
| SUBMIT-F005 | Enforce attempt/resubmission rules | System | Submit request | Limit | Legacy behavior + TBD target |
| SUBMIT-F006 | Persist submission record | System | Valid submission | Create | Draft |
| SUBMIT-F007 | Prevent duplicate requests | System | Repeated Submit / retry | Reliability | Legacy partial + TBD |
| SUBMIT-F008 | Enqueue execution job | System | Persisted submission | Async workflow | Legacy audit-reported |
| SUBMIT-F009 | Resolve language mapping | System | Job preparation | Configuration | Legacy audit-reported |
| SUBMIT-F010 | Build Judge0 request | System | Worker processing | Integration | Legacy audit-reported |
| SUBMIT-F011 | Submit execution request to Judge0 | System | Worker processing | External integration | Legacy audit-reported |
| SUBMIT-F012 | Store/correlate Judge0 execution reference | System | Judge0 acceptance | Data correlation | Draft |
| SUBMIT-F013 | Track processing and submission state | System | Queue/worker/result events | State | Decision-gated |
| SUBMIT-F014 | Handle Judge0/queue failures and retries | System/Operations | Error/timeout | Recovery | Decision-gated |
| SUBMIT-F015 | Retrieve submission status/result | Student/staff | Result page/API | Read | Draft |
| SUBMIT-F016 | Enforce submission/result ownership | System | Every read/write | Authorization | Derived |
| SUBMIT-F017 | List submission history/attempts | Student/staff | History page/API | Read | Draft |
| SUBMIT-F018 | Deliver completion updates | System | Grading completion | Event/notification | Legacy audit-reported |
| SUBMIT-F019 | Re-evaluate an existing submission | Authorized staff | Rerun action | Admin workflow | Module 7 / decision-gated |
| SUBMIT-F020 | Cancel pending/running submission | Student/staff | Cancel action | Workflow | Legacy reported absent; TBD |
| SUBMIT-F021 | Run code with custom input (scratchpad) | Student | Run Code UI | Optional feature | Legacy reported absent; TBD |
| SUBMIT-F022 | Apply rate limits and execution capacity controls | System | Submit/queue | Reliability/security | Legacy partial; target TBD |
| SUBMIT-F023 | Serialize results safely | System | Result response | Privacy/security | Derived |
| SUBMIT-F024 | Emit grading/audit integration events | System | Submission lifecycle | Integration | Legacy audit-reported |
| SUBMIT-F025 | Retain/archive submission history | System/admin | Retention policy | Data lifecycle | Decision-gated |

## 18. Detailed function specifications

### SUBMIT-F001 — Initiate formal code submission
**Requirement status:** Business Flow source-confirmed; detailed interaction derived. **Specification status:** Draft.  
**Primary actor:** Student. **System actors:** frontend, DEZOLVER API, submission service.

#### Purpose and outcome
Receive a Student’s formal code submission from a permitted Problem/activity context and return a submission identity/status that can be tracked through evaluation.

#### Permissions and scope
Student must have Create/Submit permission and access to the Problem and activity. Scope is Student-owned + permitted academic resource. HOD/Manager cannot submit on behalf of Students unless an explicit impersonation/submit-on-behalf rule is approved. Finance Team is not granted submission access.

#### Preconditions
- Student authenticated and active.
- Problem is resolvable and visible to the Student.
- The activity context, if any, is valid and permits submission.
- Submission rules are available from the owning module.
- Language and code are supplied.

#### Entry point and UI behavior
Business Flow shows Student access to Course/Problem, Lab/Problem, Contest Problem/Participation, and Practice Problems. The Page-to-DB mapping names Student Assignments, My Labs, Practice Problems, Contests, and My Courses. The legacy code audit reports a coding Assessment workspace with a Submit button. Exact target page routing and button labels are TBD.

UI behavior:
1. Student edits code and chooses a language.
2. Student invokes Submit.
3. UI prevents accidental duplicate clicks only as a convenience; server-side idempotency remains required.
4. UI receives a submission identifier and pending status.
5. UI tracks completion through the approved push/poll method.

#### Inputs

| Input | Source | Type | Required? | Validation | Notes |
|---|---|---|---|---|---|
| Problem ID | route/context | ID | Yes | Must exist and be accessible | May be derived from activity |
| Activity/context ID | route/context | ID/type | Conditional | Must be supported and valid | Course/Lab/Assessment/Contest/Practice; linkage TBD |
| Code | editor/request | string/text | Yes for formal submission | Non-empty; maximum size TBD | Legacy DTO reported min length 1 |
| Language | editor/request | string | Yes | Canonical language ID and Problem allowlist | Stored value vs Judge0 ID TBD |
| Student ID | authenticated principal | ID | System | Must not trust request body | Derived safeguard |
| stdin | request | string | Optional in legacy DTO | Policy TBD | Legacy frontend does not send it; custom input behavior is not a target contract |
| Idempotency key | request header | string | Target decision | Syntax/TTL/replay policy TBD | Legacy interceptor supports header but frontend reportedly omits it |

#### Main flow
1. Student invokes Submit.
2. Backend authenticates the actor.
3. Backend resolves Problem and activity context from trusted route/records.
4. Backend runs SUBMIT-F002 through F005.
5. Backend persists the submission using SUBMIT-F006.
6. Backend dispatches execution using SUBMIT-F008.
7. Backend returns a submission ID and status. Legacy audit reports HTTP 202 with pending status and poll URL.
8. Frontend subscribes/polls using SUBMIT-F018.

#### Business rules
- A formal submission is distinct from a scratchpad run.
- A client-provided Student ID cannot override authenticated identity.
- Submission context must be retained for the owning academic module to consume the result.
- A successful HTTP request means submission accepted for processing, not code accepted by Judge0 or final verdict accepted.

#### Validation / feedback
- Unauthenticated/inactive Student → reject.
- Inaccessible Problem/activity → reject.
- Invalid language/code/context → reject before queueing.
- Duplicate idempotency key → replay/reject according to approved policy.
- Exact error messages and response shape remain TBD.

#### Backend/data behavior
Creates or resolves a persisted submission record and execution job. No Judge0 credentials are exposed to the browser.

#### Failure/alternate/edge cases
- Double-click or network retry.
- Problem/activity changed after editor opened.
- User session expires during Submit.
- Problem becomes unavailable before persistence.
- Submission request accepted but queue dispatch fails.
- UI loses response after backend created record.

#### Audit/privacy
Submission creation event and source-code access controls are Module 15 decisions. Code must be treated as untrusted content.

#### Acceptance criteria
- Submission ID is returned only after an approved persistence/queue acceptance point.
- Student identity and context are server-derived/validated.
- UI does not treat pending as graded/accepted.
- Duplicate request behavior is deterministic.

**Trace:** Business Flow v2; RBAC coding flow; audit-reported POST endpoint. **Decisions:** SUB-D05, D07, D10, D26.

### SUBMIT-F002 — Authenticate and authorize submission
**Status:** Derived from RBAC. **Actor:** backend security layer.

- **Purpose:** ensure only an authenticated, authorized Student or explicitly approved delegated actor can create/read/manage a submission.
- **Permissions/scope:** Student Create + own records; HOD View within College/Department; Manager View within College/Department/Section/permitted Group; SUPER_ADMIN View/Manage global; Finance denied by default.
- **Preconditions:** valid authenticated principal with user/role/scope claims.
- **Entry:** every submit, retrieve, history, rerun, raw-run, or mutation request.
- **Inputs:** principal, action, Problem/context ID, submission ID. Client-supplied user/student/tenant/College/Department/Section values are untrusted.
- **Main flow:** authenticate → check role/action → resolve submission/problem/activity → check tenant/academic scope → check ownership/membership → permit data operation.
- **Rules:** role + scope + ownership/context must all pass; frontend visibility is not authorization; foreign resource IDs cannot widen access.
- **Validation:** unauthenticated → 401-equivalent; permission/scope failure → deny; existence-disclosure behavior (403 vs not-found) TBD.
- **Backend/data:** authorization must occur before code persistence, queue dispatch, result read, or raw Judge0 read.
- **Failure/edge:** role/scope changes mid-job, user disabled after submit, Staff scope changed, foreign College ID supplied.
- **Audit/privacy:** denied sensitive actions may be logged per Module 15.
- **Acceptance:** unauthorized request never creates a Judge0 job or reads another Student’s private code/results.
- **Trace:** RBAC Section 8, Code Submission API matrix. **Decision:** SUB-D17.

### SUBMIT-F003 — Validate activity and Problem eligibility
**Status:** Business flow contexts source-confirmed; eligibility details belong to Modules 4/8/9/10/11 and remain incomplete.

- **Actors:** Student + backend; context owner is Course/Lab/Assessment/Group/Contest module.
- **Preconditions:** valid Problem and context.
- **Inputs:** Problem ID, activity type/ID, actor ID, trusted context record.
- **Main flow:** resolve context → call/perform owner-module eligibility check → verify Problem association → validate membership/enrollment/attempt window/deadline/active status according to that module → accept or reject.
- **Rules:** Problem visibility alone does not imply submission permission; Course/Lab/Assessment/Contest eligibility is not interchangeable; activity ID from client cannot override server-resolved association; Group eligibility comes from Module 10.
- **Validation:** unsupported context → reject; stale relation → reject; closed/ineligible activity → owner module supplies policy.
- **Data:** no submission row or Judge0 job should be created on failed eligibility.
- **Edge:** membership changes after page load; multiple activity associations; same Problem appears in more than one activity; practice vs contest attempt semantics.
- **Acceptance:** every context is independently authorized; result can be attributed to the context.
- **Decision:** SUB-D07, D27; Modules 4/8/9/10/11.

### SUBMIT-F004 — Validate source code and language
**Status:** source code/language fields confirmed; rules partly legacy-audit-reported and partly TBD.

| Field | Legacy audit-reported behavior | Target decision |
|---|---|---|
| code | DTO string + minimum length 1; frontend checks nonempty; no DTO max length found in audit | Max bytes/chars, blank handling, encoding |
| language | frontend subset and static Judge0 map | Canonical IDs, version policy, normalization |
| stdin | optional DTO field; frontend reportedly does not populate | Formal submission vs scratchpad behavior |
| source-code payload | Express default body parser reported around 100 KB; no dedicated DTO max | API and Judge0 limit contract |

- **Main flow:** validate code type/nonempty → canonicalize language alias → verify Problem.allowedLanguages → verify Judge0 mapping is available → validate request size/format → continue.
- **Rules:** no client-supplied Judge0 language ID may bypass the allowlist; no dependency installation or custom compiler flags unless separately approved.
- **Failure:** unsupported language, empty code, too-large payload, invalid encoding → reject before execution.
- **Edge:** language aliases, language ID removed/upgraded, Unicode source, long code, request body limit.
- **Acceptance:** language accepted by DEZOLVER maps consistently to Problem and Judge0; code-size limit is explicitly documented and enforced server-side.
- **Decisions:** SUB-D04, D08, D26.

### SUBMIT-F005 — Enforce attempt and resubmission rules
**Status:** legacy audit-reported; target policy owned by Modules 8/9/11 and product.

- **Actor:** backend.
- **Preconditions:** activity context has an approved attempt policy.
- **Inputs:** Student ID, activity ID, existing attempt records, max attempts.
- **Legacy audit finding:** Assessment submission service counts submissions for user/assessment, rejects when maxAttempts is reached, increments attemptNumber inline; null maxAttempts reportedly permits unlimited attempts. No separate AssessmentAttempt entity was found in the audit.
- **Target flow:** read authoritative policy → count eligible attempts with concurrency-safe transaction/constraint → reject if limit exceeded → reserve/create attempt → persist attempt identity.
- **Rules:** attempt count must be activity-specific; rejudge must not count as a new Student attempt unless owner module says otherwise; Practice resubmission may differ from Assessment/Contest attempts.
- **Validation:** maxAttempts reached → reject; negative/invalid configured limit → owner module validation.
- **Data:** attempt counter/record requires schema/transaction confirmation.
- **Edge:** two parallel submissions near maxAttempts; queue error after attempt reservation; deleted/voided submissions; rerun.
- **Acceptance:** no race permits exceeding an approved attempt cap.
- **Decisions:** SUB-D09, D27, D28; Module 9/11.

### SUBMIT-F006 — Persist submission record
**Status:** conceptual schema source-confirmed; actual legacy model differs.

- **Actor:** backend.
- **Conceptual fields:** id, problemId, studentId, language, code, judge0Token, verdict, executionTimeMs, memoryUsageKb, failedTestCaseIndex, stdout, stderr, submittedAt, startedAt, completedAt.
- **Legacy audit-reported model:** assessment.submissions with assessmentId, userId/tenant association, code, language, attemptNumber, pending status, anti-cheat telemetry, score/verdict/result fields, and timestamps. This is not the same shape as conceptual CodeSubmission.
- **Inputs:** validated Student, Problem/context, code/language, request metadata.
- **Main flow:** derive actor → persist submission in approved model → initialize status/verdict/timestamps → return ID → queue.
- **Rules:** never persist a foreign studentId; initial status/verdict and transaction boundary require decision; persistence/queue dispatch must avoid orphaned submission or lost job.
- **Failure:** DB insert failure → no Judge0 request; queue failure after DB insert must be recoverable and visible.
- **Edge:** duplicate create, concurrent attempt number, database outage, Problem deletion/edit during request.
- **Acceptance:** submission is uniquely identifiable and its owner/context can be traced.
- **Decision:** SUB-D05, D06, D07, D28.

### SUBMIT-F007 — Prevent duplicate Submit requests
**Status:** legacy partial implementation; target idempotency is TBD.

- **Actor:** API middleware/backend.
- **Legacy audit finding:** backend IdempotencyInterceptor accepts an Idempotency-Key, validates key length/pattern, stores a Redis processing lock (reported 30-second TTL), replays successful responses for a reported 24-hour TTL, and returns 409 for an in-flight key. The frontend submission client reportedly does not send the key. No database unique constraint on user/assessment/code was reported.
- **Target flow:** require/accept approved key → bind key to actor + operation/context → reject/replay duplicates deterministically → avoid multiple records/jobs.
- **Rules:** identical code is not necessarily a duplicate; explicit repeated submissions may be valid attempts. Idempotency protects request retries, not academic attempt policy.
- **Validation:** malformed key → reject; same key/different payload behavior must be defined.
- **Data:** idempotency record/TTL/response cache contract TBD.
- **Edge:** client loses response, key expires, Redis outage, same key in two tenants, race between request and DB commit.
- **Acceptance:** transport retries do not create unintended duplicate attempts or Judge0 jobs.
- **Decision:** SUB-D10.

### SUBMIT-F008 — Enqueue execution job
**Status:** legacy audit-reported.

- **Actor:** submission service/queue.
- **Preconditions:** persisted valid submission and approved execution context.
- **Inputs:** submissionId, activityId/assessmentId, problemId, language, code, actor/user, tenant, resource configuration, optional rerun marker.
- **Legacy audit finding:** BullMQ queue named code-execution; job name reported as judge; payload includes submissionId, assessmentId, optional problemId, language, code, optional stdin, userId, tenantId, cpuTimeLimit, memoryLimitKb, isRerun. Worker concurrency reported as 20.
- **Main flow:** build minimal payload → enqueue with approved retry policy → record queue correlation/status → return submission pending.
- **Rules:** queue payload must be trusted server-side and not allow client to set privileged resource limits/tenant; sensitive source code in Redis requires access/retention controls.
- **Failure:** queue unavailable → submission must not appear successfully queued unless durable recovery is in place.
- **Edge:** worker receives duplicate job; retry; stale queued job; queue backlog.
- **Acceptance:** submission/job correlation remains one-to-one or is explicitly designed for fan-out.
- **Decision:** SUB-D11/D12/D23.

### SUBMIT-F009 — Resolve language mapping
**Status:** legacy audit-reported; target catalogue decision open.

- **Actor:** worker/backend.
- **Legacy frontend subset reported:** Python, Java, C++, C, JavaScript, TypeScript, Go, Rust, Kotlin; default UI language reported as Python.
- **Legacy static IDs reported:** Python 71; Java 62; C++ aliases cpp/cpp17/c++→54 and cpp14→48; C 50; JS/javascript/node→63; TypeScript 74; Go 60; Rust 73; Kotlin 78. The audit lists additional languages/aliases (Bash 46, C# 51, Clojure 86, COBOL 77, CoffeeScript 58, D 56, Elixir 57, Erlang 58 as reported, Fortran 59, F# 87, Groovy 88, Haskell 61, Lua 64, Objective-C 79, OCaml 65, Octave 66, Pascal 67, Perl 85, PHP 68, Prolog 69, R 80, Ruby 72, Scala 81, SQL/SQLite 82, Swift 83, VB.NET 84). Treat list as legacy code report, not a target promise.
- **Flow:** canonical DEZOLVER language → approved mapping → Judge0 language ID/version → verify Problem allowlist.
- **Rules:** frontend/editor support does not prove server support; Judge0 language IDs are deployment-version-specific.
- **Edge:** missing mapping, changed Judge0 version, alias mismatch.
- **Acceptance:** unsupported language cannot reach Judge0.
- **Decision:** SUB-D04.

### SUBMIT-F010 — Build Judge0 execution request
**Status:** legacy audit-reported; request contract needs target architecture approval.

- **Actor:** worker.
- **Legacy reported integration:** batch POST to /submissions/batch with base64_encoded=true; one Judge0 execution per testcase. Source code, stdin and expected output are base64-encoded. Request fields include source_code, language_id, stdin, expected_output, cpu_time_limit, wall_time_limit, memory_limit, and optional compiler_options/command_line_arguments.
- **Flow:** retrieve approved execution configuration from Module 5/7 → create one execution request per testcase → encode payload as required by the deployed Judge0 version → attach only authorized fields → submit batch.
- **Rules:** DEZOLVER—not the browser—controls expected outputs, hidden tests, and resource settings; no hidden test payload returned to Student.
- **Validation:** missing language ID/testcase data/resource configuration → stop before external request and report operational error.
- **Edge:** oversized batch, empty testcase set, encoding errors, config changed while queued.
- **Acceptance:** each execution request maps to exactly one submission + testcase identity in the approved design.
- **Decision:** SUB-D02/D21/D22.

### SUBMIT-F011 — Submit execution request to Judge0
**Status:** Business Flow source-confirmed; protocol legacy-audit-reported.

- **Actor:** worker/backend.
- **Preconditions:** submission is authorized and persisted; Judge0 config available.
- **Inputs:** batch payload, endpoint/base URL, server-side authentication configuration.
- **Legacy audit report:** standard Judge0 client uses JUDGE0_BASE_URL (reported default https://judge.truenooklms.org) and optional X-Auth-Token; an alternate RapidAPI client exists. Reported endpoint POST /submissions/batch?base64_encoded=true.
- **Flow:** call configured Judge0 endpoint → validate response shape/status → collect token(s) → correlate response to expected request count/order/IDs.
- **Rules:** never expose API token/secret to frontend; endpoint and credentials must be environment/configuration managed; do not assume external host/secret values are approved for target.
- **Failure:** non-2xx, malformed response, missing token, response-count mismatch → operational failure; do not label as WA.
- **Edge:** Judge0 accepts some batch entries but rejects others; request timeout after Judge0 created work.
- **Acceptance:** no token mismatch; retry policy prevents duplicate external work or records it as a separate execution attempt.
- **Decision:** SUB-D01/D02/D11.

### SUBMIT-F012 — Store and correlate Judge0 references
**Status:** conceptual schema has judge0Token; legacy audit reports per-testcase token array.

- **Actor:** worker/backend.
- **Inputs:** submission ID, testcase IDs/order, Judge0 token(s).
- **Flow:** persist token mapping → associate token with submission + testcase/run → use correlation for polling/result persistence.
- **Rules:** conceptual single judge0Token field and legacy per-testcase tokens are incompatible shapes unless the target model explicitly supports one token per testcase or run entity.
- **Validation:** duplicate token or missing token mapping is an error.
- **Data:** token mapping, execution attempt, testcase association, and timestamp require schema confirmation.
- **Edge:** token expiry, partial token response, token lost on worker retry.
- **Acceptance:** every returned Judge0 result resolves to the correct submission and testcase.
- **Decision:** SUB-D06/D21.

### SUBMIT-F013 — Track processing and submission state
**Status:** conceptual verdict enum source-confirmed; state machine is TBD.

- **Conceptual JudgeVerdict:** PENDING, RUNNING, ACCEPTED, WRONG_ANSWER, TIME_LIMIT_EXCEEDED, MEMORY_LIMIT_EXCEEDED, RUNTIME_ERROR, COMPILATION_ERROR.
- **Legacy audit-reported status:** pending → queue/worker processing → graded, with verdict/score separate; error/internal_error may appear in implementation.
- **Flow:** create pending → queue/worker start → Judge0 accepted/processing → evaluation complete → persist final result; state/timestamp mapping is Module 17-owned.
- **Rules:** do not add persisted verdict values (PARTIAL, INTERNAL_ERROR, CANCELLED, etc.) without schema approval; infrastructure state may need a separate status field.
- **Timestamps:** submittedAt, startedAt, completedAt are conceptual fields; authority/nullability/clock source TBD.
- **Edge:** worker starts but cannot write; late completion after retry; stale PENDING/RUNNING.
- **Acceptance:** legal transitions and timestamps are deterministic and documented.
- **Decision:** SUB-D13/D14/D15; Module 17.

### SUBMIT-F014 — Handle Judge0 and queue failures
**Status:** legacy audit-reported and decision-gated.

- **Legacy findings:** BullMQ job attempts reported as 3 with exponential backoff starting at 2 seconds; polling deadline reported as 60 seconds; poll delay grows from 1s by 0.5s up to 4s. On worker exception, audit reports status/error/internal_error update then rethrow/retry. Poll timeout reportedly returns partial results rather than throwing. The source audit reports that retries can submit a fresh batch/token set, creating duplicate external execution.
- **Target flow:** classify transient vs permanent error → retry only according to approved policy → maintain correlation and attempt history → mark unresolved work operationally recoverable → avoid grading incomplete results as student failures.
- **Rules:** infrastructure errors are not Wrong Answer; partial Judge0 responses are not silently treated as complete; duplicate execution side effects must be accounted for.
- **Edge:** Judge0 outage, 429, 5xx, token not terminal, queue retry exhaustion, worker crash, Redis/database failure, partial batch response.
- **Acceptance:** failure state is visible, recoverable, and does not incorrectly penalize Student.
- **Decision:** SUB-D11/D12/D14; Modules 17/18.

### SUBMIT-F015 — Retrieve submission status and result
**Status:** source/RBAC-derived; legacy API paths are audit references only.

- **Actors:** Student (own submission), HOD/Manager (scope-limited), SUPER_ADMIN (approved global access).
- **Entry:** result panel, submission detail, refresh/poll endpoint.
- **Inputs:** submission ID; caller principal.
- **Flow:** authenticate → authorize via F002/F016 → load persisted status/result → return approved DTO fields → UI displays status.
- **Legacy audit endpoints reported:** GET /api/v1/submissions/:id; GET /api/v1/me/submissions; exact contract is not target-approved.
- **Rules:** pending/running/graded/error states must not be conflated; Student cannot access another user’s submission; result detail must not include hidden test secrets.
- **Edge:** nonexistent/foreign submission, stale response, partial evaluation, rerun currently processing.
- **Acceptance:** result read is permission-checked and consistent with current persisted state.
- **Decision:** SUB-D16/D17/D20.

### SUBMIT-F016 — Enforce submission ownership and staff scope
**Status:** RBAC source-confirmed/derived.

- **Actors:** backend authorization.
- **Student:** may view own submission only.
- **HOD:** within College + Department.
- **Manager:** within College + Department + assigned Section/permitted Group.
- **SUPER_ADMIN:** global.
- **Finance Team:** no Code Submission grant in RBAC matrix.
- **Flow:** resolve submission owner + Problem + activity context → apply actor role/scope/ownership → return permitted record fields.
- **Rules:** staff scope does not imply permission to view raw source/hidden output; field-level permission is separate; changing submission ID cannot bypass ownership.
- **Edge:** user moved to a new Department/Section after submission; archived Student; group membership changed.
- **Acceptance:** direct API access and list page enforce same scope; foreign IDs are denied or safely masked.
- **Decision:** SUB-D17/D19.

### SUBMIT-F017 — List submission history and attempts
**Status:** conceptual source mapping; legacy audit-reported endpoint.

- **Actors:** Student own history; HOD/Manager scoped views if authorized; SUPER_ADMIN.
- **Inputs:** filters, cursor/page, Problem/activity, status, dates; query contract TBD.
- **Legacy audit report:** GET /api/v1/me/submissions with cursor-based pagination ordered by submittedAt descending; submission rows retained indefinitely in the reported snapshot.
- **Flow:** authorize → filter by ownership/scope before pagination → sort deterministically → return safe summary → load detail on demand.
- **Rules:** history list must not expose foreign Student code/results; rejudge is not a new Student attempt; deleted/deactivated user retention policy TBD.
- **Edge:** cursor stability, repeated submission timestamps, large history, activity filters.
- **Acceptance:** stable pagination, correct owner/scope, safe list DTO.
- **Decision:** SUB-D19/D20.

### SUBMIT-F018 — Deliver completion updates
**Status:** legacy audit-reported.

- **Actors:** backend event/outbox, Redis, Socket.io client, polling fallback.
- **Legacy report:** grading event is emitted to outbox/Redis channels and frontend receives submission:graded; Assessment workspace reportedly polls every 3 seconds as fallback.
- **Flow:** persist final result → emit approved event → push status update → client updates result panel; polling remains fallback if push is unavailable.
- **Rules:** event payload should contain only safe status/summary fields; event delivery must not be treated as source of truth over DB; no duplicate notification side effects.
- **Failure:** socket disconnect, missed event, Redis failure, stale poll, duplicate event.
- **Acceptance:** client eventually reflects authoritative persisted result without leaking another Student’s result.
- **Decision:** SUB-D25; Modules 13/15/18.

### SUBMIT-F019 — Re-evaluate an existing submission
**Status:** legacy audit-reported; primarily Module 7.

- **Actors:** authorized faculty/platform administrator only if approved.
- **Legacy report:** POST /api/v1/submissions/:id/rerun requires manage:assessment and legacy platform_admin/faculty role checks; rerun resets submission to pending and enqueues a new job with isRerun=true; main aggregate record is overwritten while judge_runs are retained.
- **Flow:** authorize rerun → record reason/actor (if required) → re-evaluate under approved Problem/testcase version → preserve run history → update aggregate and downstream events per Module 7/15.
- **Rules:** rejudge is not a new Student attempt by default; permission and historical result behavior require approval; no assumption that legacy role names map to DEZOLVER roles.
- **Edge:** rerun while original job is active; partial rerun; testcase changed; old result overwritten; contest/assessment already finalized.
- **Acceptance:** rerun is traceable, role-gated, and does not corrupt attempt counts.
- **Decision:** SUB-D20; Module 7.

### SUBMIT-F020 — Cancel pending/running submission
**Status:** legacy report says no cancellation endpoint/job/Judge0 cancellation path; target feature TBD.

- **Actor:** Student or staff only if approved.
- **Inputs:** submission ID and cancel action.
- **Preconditions:** submission is in a cancellable state and caller has permission.
- **Flow:** authorize → check state → request cancellation → update canonical state only after confirmed policy/worker behavior → prevent later result from silently overwriting cancellation.
- **Rules:** cancelling DEZOLVER queue job may not cancel external Judge0 execution; refund/attempt-count behavior is activity-owned.
- **Edge:** already running, Judge0 already accepted, worker race, completed submission, retries.
- **Acceptance:** no false claim that remote execution stopped; cancellation state/attempt behavior are defined.
- **Decision:** SUB-D18; Modules 17/18.

### SUBMIT-F021 — Run code with custom input (scratchpad)
**Status:** legacy code audit reports absent; this is not a current feature.

- **Actor:** Student; optional future feature.
- **Inputs:** source code, language, custom stdin.
- **Flow:** not defined as a target feature until approved. If adopted, it must be a distinct workflow from formal Submit.
- **Rules:** scratchpad execution must not create academic attempts, score, submission history, contest penalties, or completion events unless explicitly approved.
- **Legacy finding:** optional stdin existed in DTO but frontend did not expose a custom-input UI; if Problem ID exists, worker loads testcase data and ignores stdin; separate Run Code pipeline was reported absent.
- **Acceptance:** feature cannot be presented as existing or enabled without separate product/architecture design.
- **Decision:** SUB-D21.

### SUBMIT-F022 — Apply rate limits and execution capacity controls
**Status:** legacy partial implementation; target quotas TBD.

- **Legacy audit reports:** plan-based per-tenant request rates (starter 1,000/min, professional 10,000/min, enterprise 50,000/min, unlimited/direct-free unbounded); worker concurrency 20; tenant fairness guard reported as max 3 tenant jobs in a 60-second window. Per-user submission throttle reportedly absent.
- **Target flow:** validate plan/tenant limits → validate per-user/activity policy → queue fairly → return accepted/rate-limited response with retry guidance.
- **Rules:** audited numbers are not approved target quotas; tenant and per-user limits need explicit decision. Do not allow a Student to override tenant identity.
- **Edge:** burst submit, one tenant saturating queue, plan downgrade, Redis limit service failure.
- **Acceptance:** fairness and capacity policy is approved and observable; limit responses do not create partial submissions.
- **Decision:** SUB-D09/D22.

### SUBMIT-F023 — Serialize result/output safely
**Status:** derived security requirement; detailed masking belongs Module 7.

- **Actors:** backend result DTO/serializer.
- **Flow:** authorize caller → map stored raw execution to role-specific response → apply hidden-case masking and field policy → limit/redact outputs → return.
- **Rules:** hidden inputs/expected outputs, raw tokens, secrets, internal hostnames, unfiltered stderr/compiler output, and infrastructure internals must not leak to Students. Masking must be server-side.
- **Legacy audit warning:** reported hidden actual/expected outputs were masked, but stderr/compileOutput were not masked by sample/hidden classification.
- **Edge:** direct API, WebSocket event, logs, error response, output truncation, malicious code writing secrets to stderr.
- **Acceptance:** every transport (REST, socket, logs where appropriate) applies the approved privacy policy.
- **Decision:** SUB-D16/D23; Module 7.

### SUBMIT-F024 — Emit grading and audit integration events
**Status:** legacy audit-reported; event contract TBD.

- **Actors:** submission service, outbox, event consumers.
- **Legacy audit:** code execution worker emits SubmissionGraded; a course progress consumer reportedly listens for SubmissionAccepted, creating an event-name mismatch; contest consumer reportedly accepts accepted/partial verdicts and updates score.
- **Flow:** after final result is persisted, emit one canonical grading event → owning modules consume according to contract → record delivery/retry.
- **Rules:** no downstream score/progress change before authoritative result persistence; event delivery should be idempotent; Module 11 owns contest scoring and Module 8 lab score.
- **Edge:** event duplicate, event missing, consumer unavailable, rerun re-emits grading event.
- **Acceptance:** downstream modules can identify submission, Student, context, verdict, score, and run version; consumers do not double count.
- **Decision:** SUB-D24/D25; Modules 8/9/11/13/15/18.

### SUBMIT-F025 — Retain and archive submission history
**Status:** conceptual lifecycle unresolved; legacy audit reports indefinite retention.

- **Actors:** system/data governance/admin.
- **Inputs:** approved retention policy, submission owner/context, legal/operational requirements.
- **Legacy audit:** submissions reportedly persist permanently; no auto-purge/archival cron was found in audited snapshot.
- **Flow:** apply approved retention → preserve required academic/audit records → restrict access or archive/delete only under policy → ensure external Judge0 tokens/artifacts follow retention.
- **Rules:** Student deactivation or Problem deletion must not silently destroy assessment history; source code and raw outputs are sensitive.
- **Edge:** deletion request, inactive Student, data export, legal hold, retention expiry, rejudge history.
- **Acceptance:** retention, deletion, and archival behavior are explicit; no indefinite storage is assumed for target.
- **Decision:** SUB-D19/D23.

## 19. Business rules and validation matrix

| Rule ID | Rule | Evidence/status |
|---|---|---|
| SUB-R01 | Submission must be associated with authenticated Student and Problem. | Schema + derived |
| SUB-R02 | Student submits only in a permitted academic context. | RBAC + Business Flow |
| SUB-R03 | Finance Team has no Code Submission permission in source RBAC. | RBAC |
| SUB-R04 | HOD/Manager submission inspection is scope-limited. | RBAC |
| SUB-R05 | Judge0 request is server-side; browser does not own privileged execution. | RBAC Judge0 flow |
| SUB-R06 | Formal submission acceptance is not equivalent to ACCEPTED verdict. | Derived |
| SUB-R07 | Problem.allowedLanguages/resource limits are governed by Module 5. | Module 5 |
| SUB-R08 | Testcase evaluation and score are Module 7 responsibilities. | Module boundary |
| SUB-R09 | Activity attempt/deadline rules are owned by Modules 8–11. | Module boundary |
| SUB-R10 | Rejudge must not count as new Student attempt unless approved. | Derived |
| SUB-R11 | Infrastructure failure must not be mapped to Wrong Answer by default. | Derived |
| SUB-R12 | Client-provided Student/tenant/scope values cannot widen access. | RBAC |
| SUB-R13 | Result and raw output visibility is role/ownership-controlled. | RBAC + derived |
| SUB-R14 | Code source and execution output are untrusted/sensitive. | Security derived |
| SUB-R15 | Legacy code uses assessment-specific Submission model; conceptual schema uses CodeSubmission. | Source conflict |
| SUB-R16 | Legacy per-testcase Judge0 token fanout conflicts with conceptual single token field. | Source conflict |
| SUB-R17 | Legacy audit reports idempotency partial; frontend omitted key. | Audit-reported |
| SUB-R18 | Legacy audit reports no Run Code/cancel flow. | Audit-reported |
| SUB-R19 | Legacy audit reports no plagiarism similarity feature. | Audit-reported; not target promise |
| SUB-R20 | Submission history retention must be approved, not inferred from legacy permanence. | Decision required |

| Validation ID | Condition | Expected behavior | Decision |
|---|---|---|---|
| SUB-V01 | Unauthenticated request | Reject before persistence/queue | RBAC |
| SUB-V02 | Student inactive/disabled | Reject per Module 1 | AUTH/Module 1 |
| SUB-V03 | Student submits foreign Problem | Deny | RBAC/Module 5 |
| SUB-V04 | Activity context invalid/closed | Deny by owner module | D27 |
| SUB-V05 | Problem not in activity | Deny | D07 |
| SUB-V06 | Unsupported activity context | Reject; no guessed behavior | D07 |
| SUB-V07 | Missing/blank code | Reject | D08 |
| SUB-V08 | Code exceeds approved payload limit | Reject before queue | D08 |
| SUB-V09 | Unsupported language | Reject | D04 |
| SUB-V10 | Language not in Problem allowedLanguages | Reject | Module 5 |
| SUB-V11 | Client supplies foreign studentId/tenantId | Ignore/deny; use principal | RBAC |
| SUB-V12 | Max attempts reached | Reject | D09 |
| SUB-V13 | Concurrent submissions near attempt cap | Atomic cap enforcement | D09 |
| SUB-V14 | Duplicate request with same key | Replay/reject per idempotency contract | D10 |
| SUB-V15 | Same key but different payload | Reject/conflict per policy | D10 |
| SUB-V16 | Judge0 request non-2xx | Operational failure; not WA | D11/D14 |
| SUB-V17 | Missing/duplicate Judge0 token | Correlation failure | D06 |
| SUB-V18 | Partial batch response | Incomplete/operational outcome | D12/D14 |
| SUB-V19 | Result retrieval by another Student | Deny | RBAC |
| SUB-V20 | Staff views out-of-scope submission | Deny | RBAC |
| SUB-V21 | Student requests raw judge-run record | Deny unless policy grants | D17 |
| SUB-V22 | Result contains hidden case diagnostics | Mask per Module 7 | Module 7 |
| SUB-V23 | Socket event for foreign submission | Do not emit/serialize to unauthorized user | D25 |
| SUB-V24 | Rejudge by Student | Deny | D20 |
| SUB-V25 | Cancel completed submission | Reject/no-op per state policy | D18 |
| SUB-V26 | Queue retry creates repeated external jobs | Deduplicate or retain run IDs per policy | D11 |
| SUB-V27 | Submission status stuck pending | Operational detection/recovery per Module 18 | D12 |
| SUB-V28 | Retention expiry | Apply approved retention rule | D19 |

## 20. Data and state reference

### 20.1 Conceptual CodeSubmission (source schema reconstruction)

| Field | Type | Meaning | Notes |
|---|---|---|---|
| id | varchar | Submission ID | generation unspecified |
| problemId | varchar | Problem reference | source |
| studentId | varchar | Student reference | owner |
| language | varchar(50) | Language name/identifier | canonical format TBD |
| code | text | Source code | retention/size TBD |
| judge0Token | varchar(255) | Judge0 reference | one token field in conceptual schema |
| verdict | JudgeVerdict | execution/evaluation verdict | enum source |
| executionTimeMs | int | execution time | aggregation/units semantics TBD |
| memoryUsageKb | int | memory usage KB | aggregation semantics TBD |
| failedTestCaseIndex | int | failed testcase pointer | index base/meaning TBD |
| stdout | text | output | exposure/size policy TBD |
| stderr | text | error output | exposure/sanitization TBD |
| submittedAt | timestamp | submission time | clock source TBD |
| startedAt | timestamp | execution start | clock source/nullability TBD |
| completedAt | timestamp | completion | clock source/nullability TBD |

### 20.2 Conceptual JudgeVerdict enum

PENDING, RUNNING, ACCEPTED, WRONG_ANSWER, TIME_LIMIT_EXCEEDED, MEMORY_LIMIT_EXCEEDED, RUNTIME_ERROR, COMPILATION_ERROR.

The legacy audit reports other status strings in the physical assessment submission model (e.g. graded/error/internal_error/partial). These models must be reconciled before schema/API finalization.

### 20.3 Legacy audited persistence model — distinct, not merged

| Entity/field family | Audit-reported legacy behavior | Conflict/decision |
|---|---|---|
| assessment.submissions | Submission rows include assessmentId, userId, attemptNumber, code/language, status/verdict, score, passed/total, testCaseResults, anti-cheat signals, submitted/graded timestamps | Not same as conceptual CodeSubmission |
| assessment.judge_runs | Per-testcase raw execution rows with token/status/output/runtime/memory/compile output/exit code | conceptual CodeSubmission has one judge0Token |
| outbox events | SubmissionGraded event recorded after grading | canonical event contract TBD |
| Redis/WebSocket | submission:graded push, user/submission channels; UI poll fallback | transport contract TBD |
| testcases | content.problem_test_cases with isSample, weight (audit) | schema reconstruction uses TestCase.isHidden/orderIndex |

### 20.4 State reference (not canonical state machine)

| Concept | Source/audit states | Status |
|---|---|---|
| Conceptual verdict | PENDING/RUNNING + terminal JudgeVerdict enum values | Schema confirmed; transitions TBD |
| Legacy submission status | pending → graded; error on worker exception; separate verdict/score | Audit-reported |
| Judge0 status | queued/processing/terminal Judge0 native status IDs | External protocol |
| Queue job | waiting/active/retry/completed/failed according to BullMQ | Legacy architecture; Module 17/18 |
| Rerun | submission row reset pending, fresh Judge0 tokens; old aggregate overwritten, raw judge-run history retained | Audit-reported; target TBD |

## 21. Legacy code audit findings and source conflicts

The supplied **DEZOLVER Code Execution and Evaluation Deep Dive** reports the following. These are snapshot-specific implementation observations, not target decisions:

1. Formal code submission is Assessment-bound in the legacy code: POST /api/v1/assessments/:id/submissions; assessment kind must be coding_problem.
2. Legacy API reportedly returns 202 Accepted with submissionId/status/pollUrl.
3. Legacy code enforces Assessment.maxAttempts when set; null reportedly allows unlimited attempts; attemptNumber is inline.
4. Legacy pipeline is asynchronous: submission service → PostgreSQL → BullMQ/Redis → worker → Judge0 → persisted result → outbox/Redis/WebSocket.
5. Judge0 standard client uses server-side config and X-Auth-Token; alternate RapidAPI client also exists.
6. Legacy worker reportedly creates one Judge0 token per testcase and sends all cases in a batch; Module 7 owns per-testcase evaluation semantics.
7. Legacy language IDs are static; listLanguages exists but is reportedly not used in submission lifecycle.
8. Legacy CPU limit derivation reportedly uses Assessment.timeLimitMinutes × 60 as per-testcase CPU seconds, a potential severe mismatch with Problem.timeLimitMs. This is an audit finding to verify/correct, not target behavior.
9. Legacy body/DTO validation reportedly lacks explicit code maximum; frontend checks nonempty only; Express default parser size is cited in audit. Target request size must be explicitly designed.
10. Legacy plan-based tenant rate limiting is reported; per-user throttling is reportedly absent.
11. Legacy worker concurrency is reported as 20; tenant fairness guard reports 3 per tenant in a 60-second window. These values are not approved target quotas.
12. Legacy retry behavior reportedly re-submits Judge0 batch on BullMQ retry, risking duplicate external executions.
13. Legacy Judge0 polling deadline reportedly 60 seconds; unfinished tokens can return partial results; infrastructure mapping requires review.
14. Legacy IdempotencyInterceptor is reported, but frontend omits Idempotency-Key, leaving normal Submit workflow exposed to duplicate requests.
15. Legacy cancellation and separate Run Code/custom input UI are reported absent.
16. Legacy submission records are reported as retained indefinitely; target retention is TBD.
17. Legacy source audit reports no plagiarism/similarity engine.
18. Legacy output/result DTO may expose stderr/compiler diagnostics without hidden-case masking; Module 7 must address.
19. Legacy course progress event mismatch is reported: worker emits SubmissionGraded while progress consumer listens for SubmissionAccepted.
20. Legacy quiz/MCQ submissions are reportedly rejected by code submission service; do not assume quiz support.

### 21.1 Schema reconciliation register

| Conflict | Conceptual source | Legacy audit | Resolution needed |
|---|---|---|---|
| Submission entity | CodeSubmission | assessment.submissions / Submission | D06 |
| Context | Problem + Student fields; academic pages imply multiple contexts | AssessmentId mandatory in legacy | D07 |
| Judge0 token cardinality | one judge0Token field | per-testcase token array/run rows | D06/D21 |
| Verdict/state | JudgeVerdict enum | status + string verdict/score fields | D13/D14 |
| Attempt tracking | not shown | attemptNumber inline | D09/D28 |
| Code source retention | not stated | legacy records indefinite | D19/D23 |
| Rate/queue config | not stated | plan limit, worker concurrency, fairness settings | D09/D22 |
| Run code/cancel | not specified | audit reports not implemented | D18/D21 |

## 22. Cross-module dependencies

| Module | Dependency / contract |
|---|---|
| 1 Authentication | authenticated identity, active status, role, session |
| 2 Organization | College/Department/Section scope |
| 4 Course & Enrollment | course membership/problem access/submission context |
| 5 Problem Management | allowed languages, visibility, time/memory, Problem metadata |
| 7 Test Case Evaluation | testcase execution, result mapping, score/verdict |
| 8 Labs & Lab Scoring | Lab context association and score consumption |
| 9 Assessments | Assessment availability, attempt/deadline policy, assessment submission association |
| 10 Groups | group membership and eligibility |
| 11 Contests | contest participation/window/penalty/score/rank |
| 13 Notifications | submission result notification triggers |
| 15 Audit | submission, rerun, result/raw data access events |
| 17 State Machines | canonical submission states and transitions |
| 18 Edge Cases | queue/Judge0 retry, stuck job, duplicate, timeout, recovery |

## 23. Decision log status

SUB-D01–SUB-D28 are **Open** unless explicitly approved by the product owner. This documentation commit does not resolve them.

| Decision | Question | Owner | Status |
|---|---|---|---|
| SUB-D01 | Judge0 deployment/environment | Architecture/DevOps | Open |
| SUB-D02 | Judge0 API contract/auth/version | Architecture | Open |
| SUB-D03 | sync/async/poll/callback delivery | Architecture/Product | Open |
| SUB-D04 | canonical language IDs and versions | Product/Architecture | Open |
| SUB-D05 | submission record creation + initial state | Product/Architecture | Open |
| SUB-D06 | CodeSubmission vs legacy Submission schema; token cardinality | Architecture/Data | Open |
| SUB-D07 | context fields/association for Course/Lab/Assessment/Contest/Practice | Product/Data | Open |
| SUB-D08 | code payload max size/validation | Product/Security | Open |
| SUB-D09 | attempt, rate, concurrency, fairness limits | Product/Architecture | Open |
| SUB-D10 | idempotency keys/replay/duplicate request | Architecture | Open |
| SUB-D11 | queue/Judge0 retry policy and duplicate external job prevention | Architecture/Ops | Open |
| SUB-D12 | Judge0 and worker timeouts, stuck submissions | Architecture/Ops | Open |
| SUB-D13 | native Judge0 status → DEZOLVER verdict mapping | Product/Architecture | Open |
| SUB-D14 | infrastructure error status separate from student-code verdict | Product/Architecture | Open |
| SUB-D15 | timestamps/runtime/memory/index semantics | Product/Architecture | Open |
| SUB-D16 | stdout/stderr/compiler output masking, limits, retention | Security/Product | Open |
| SUB-D17 | Student/HOD/Manager/SUPER_ADMIN raw code/result permissions | Security/Product | Open |
| SUB-D18 | cancellation behavior and attempt impact | Product/Architecture | Open |
| SUB-D19 | history retention/deletion/deactivation behavior | Product/Privacy | Open |
| SUB-D20 | rerun/rejudge roles, reason, history, score overwrite | Product/Architecture | Open |
| SUB-D21 | Run Code/custom stdin vs formal Submit | Product | Open |
| SUB-D22 | tenant/user queue/rate/fairness quotas | Product/Ops | Open |
| SUB-D23 | source code/raw execution artifact protection | Security/Privacy | Open |
| SUB-D24 | canonical grading event/outbox contract | Architecture | Open |
| SUB-D25 | push/poll/notification behavior | Product/Architecture | Open |
| SUB-D26 | code request schema and response/API contract | Product/Architecture | Open |
| SUB-D27 | academic activity context integration and permission | Product/Module owners | Open |
| SUB-D28 | attempt numbering and race-safe reservation | Product/Data | Open |

## 24. Review and readiness checklist

- [ ] SUBMIT-F001–F025 reviewed by Product, Architecture, Security, and relevant academic module owners.
- [ ] RBAC action/scope/ownership rules approved, including field-level raw code/output access.
- [ ] Practice/Course/Lab/Assessment/Contest submission context is confirmed and represented.
- [ ] Conceptual CodeSubmission and legacy assessment.submissions schema conflict resolved.
- [ ] Judge0 token cardinality and correlation model resolved.
- [ ] Judge0 deployment, API contract, authentication, language IDs, and versions approved.
- [ ] Problem language/resource limits are applied consistently (Module 5).
- [ ] Attempt/resubmission/rerun rules approved for each context.
- [ ] Idempotency and queue retry behavior prevent accidental duplicate attempts/executions.
- [ ] Status/verdict lifecycle and infrastructure failure representation approved (Module 17).
- [ ] Code size, rate limits, concurrency, fairness, cancellation, and custom-run scope approved.
- [ ] Student/staff result visibility, hidden-data masking, output limits, and retention approved.
- [ ] Event/outbox/notification contracts align with Modules 8–11/13/15.
- [ ] QA covers role, scope, duplicate, queue, Judge0, security, and failure scenarios.
- [ ] Legacy findings, derived safeguards, recommendations, and target TBDs remain distinct.

**Review gate:** Module 6 detailed draft is ready for Product/Architecture/Security review. Module 7 is being upgraded in parallel under the user's explicit instruction. GitHub inclusion versions documentation; it does not approve unresolved target behavior.
