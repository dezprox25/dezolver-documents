# 7. Test Case Evaluation

**Document type:** Functional Specification  
**Version:** 1.0 — Adopted target baseline  
**Status:** Product decisions adopted (EVAL-D01–EVAL-D27); engineering and QA execution pending  
**Module owner:** DEZOLVER Evaluation / Judge0 integration  
**Primary technical source:** `DEZOLVER_EVALUATION_ALGORITHM_AND_TEST_CASE_SCORING_DEEP_DIVE.md`  
**Cross-reference sources:** Business Flow v2, Page-to-DB Mapping v2, schema reconstruction, Module 5 — Problem Management, Module 6 — Code Submission & Judge0

> **Adopted target baseline:** This specification reconciles the supplied source and legacy audit with Modules 5 and 6. Sections 30–33 and ADOPTED-DECISIONS.md are normative. Earlier audit descriptions remain evidence about a legacy snapshot, not target implementation requirements.

---

## 1. Purpose

Define how DEZOLVER evaluates submitted source code against a Problem’s test cases, determines individual test-case outcomes, aggregates results into a submission verdict and score, protects hidden test-case information, and makes evaluation results available to authorized users and downstream academic workflows.

The Business Flow v2 establishes the high-level path:

**Code Submission → DEZOLVER Backend → Judge0 → Evaluation → Result → Score**

Module 6 — Code Submission & Judge0 covers submission intake and the backend/Judge0 handoff. This module defines evaluation behavior after a submission is accepted for judging.

## 2. Scope

### 2.1 Included

- Test-case retrieval for an eligible Problem
- Test-case execution through Judge0
- Batch submission and per-testcase execution references
- Output comparison boundary
- Sample and hidden test-case handling
- Per-testcase result collection
- Weighted scoring and score rounding
- Overall verdict aggregation
- Compilation, runtime, time-limit, memory-limit, and wrong-answer outcomes
- Result persistence and student/staff result visibility
- Re-evaluation / rejudge behavior as it affects evaluation
- Hidden-test data protection
- Evaluation defects, operational risks, and unresolved decisions

### 2.2 Excluded or owned by other modules

| Topic | Owner |
|---|---|
| Problem authoring, statement, allowed languages, resource limits, publication | Module 5 — Problem Management |
| Submission intake, source-code capture, Judge0 integration boundary | Module 6 — Code Submission & Judge0 |
| Lab assignment and lab score aggregation | Module 8 — Labs & Lab Scoring |
| Assessment availability, time windows, attempts, overall assessment scoring | Module 9 — Assessments |
| Group membership and activity eligibility | Module 10 — Groups |
| Contest participation, penalties, ranking and tie-breaking | Module 11 — Contests |
| Notifications | Module 13 — Notifications |
| Audit events and audit retention | Module 15 — Audit Logging |
| Canonical lifecycle/state-machine rules | Module 17 — Functional State Machines |
| Cross-module failure and recovery catalogue | Module 18 — Edge Cases & Failure Handling |

Test-case evaluation may be invoked from Practice Problems, Courses, Labs, Assessments, and Contests, as suggested by Business Flow v2 and Page-to-DB Mapping v2. The exact context-specific policies are not defined here unless directly related to evaluation.

## 3. Sources and evidence interpretation

### 3.1 Source documents

1. **DEZOLVER Evaluation Algorithm & Test Case Scoring Deep Dive** — code-level investigation of the NestJS backend, Judge0 client, worker, Prisma models, API, and frontend result handling.
2. **Business Flow v2** — establishes the backend → Judge0 → Evaluation → Result → Score flow and the academic contexts where submissions occur.
3. **Page-to-DB Mapping v2** — maps Problem/TestCase/CodeSubmission to Problem Management and Practice Problems, and CodeSubmission to Courses & Labs, Assessments, and Contests.
4. **Schema reconstruction (`scheema.pdf` → Markdown)** — provides the original conceptual `Problem`, `TestCase`, and `CodeSubmission` fields.
5. **Module 5 and Module 6 drafts** — define adjacent problem and submission responsibilities.

### 3.2 Evidence labels used in this document

- **[AUDIT—CURRENT]**: behavior reported as implemented in the deep-dive code investigation.
- **[SOURCE—BUSINESS]**: workflow supported by Business Flow v2 or Page-to-DB Mapping v2.
- **[DERIVED REQUIREMENT]**: functional safeguard or behavior proposed based on the sources; it needs product/technical confirmation if not explicitly documented.
- **[OPEN DECISION]**: source material does not establish a definitive rule.
- **[ADOPTED TARGET]**: resolved product policy EVAL-D01–EVAL-D27; see Sections 30–33 and ADOPTED-DECISIONS.md.
- **[ENGINEERING GATE]**: schema migration, implementation, deployment verification, integration or QA execution remains pending.

The deep-dive report describes a specific source-code snapshot. It should not automatically override the original schema reconstruction if they represent different versions.

## 4. Business context and actors

### 4.1 Evaluation entry points

| Context | Source evidence | Context-specific policy owner |
|---|---|---|
| Practice Problems | Student Practice Problems page maps Problem, TestCase, CodeSubmission | Module 5 / Module 6 |
| Course problems | Business Flow shows Course → Code Submission; mapping includes CodeSubmission | Course-related module(s) |
| Labs | Business Flow shows Group → Lab → Problem; mapping includes LabSubmission and CodeSubmission | Module 8 |
| Assessments | Page mapping includes Assessment, AssessmentProblem, Problem, CodeSubmission | Module 9 |
| Contests | Business Flow shows Contest → Contest Problem → Participation → Code Submission | Module 11 |

**[DERIVED REQUIREMENT]** Evaluation must preserve the relationship to the originating submission and Problem. Activity eligibility, deadlines, attempt policy, and downstream score aggregation must be enforced by the owning module and must not be inferred solely from evaluation results.

### 4.2 Roles and visibility

The source mapping identifies Students and academic staff (HOD/Manager) in Problem, Course/Lab, Assessment, and Contest workflows. Exact staff permissions for raw execution details are not fully defined.

**[DERIVED REQUIREMENT]**
- A Student may view evaluation results only for their own submission, subject to activity rules.
- Faculty/HOD/Manager access must be limited to their authorized academic scope.
- Platform administrators may inspect execution details only under an approved permission policy.
- Hidden test input, expected output, and other secret test-case material must not be exposed through Student-facing APIs, UI, logs, or error details.
- Access to raw Judge0 payloads and execution artifacts must be explicitly permission-controlled.

**[OPEN DECISION]** Define which staff roles may view source code, raw stdout/stderr, compiler output, hidden testcase metadata, and raw Judge0 responses.

## 5. Evaluation architecture and lifecycle

### 5.1 Reported current architecture [AUDIT—CURRENT]

The deep-dive reports this asynchronous flow:

1. Student submits source code through DEZOLVER.
2. Submission service validates the submission context and attempt rules, then persists a pending submission.
3. A BullMQ job is enqueued in the `code-execution` queue.
4. Worker loads the Problem’s sample and hidden test cases.
5. Worker constructs one Judge0 execution payload per testcase.
6. Worker submits the testcase payloads in one Judge0 batch request.
7. Judge0 returns a separate token for each testcase.
8. DEZOLVER polls the batch until all returned tokens are terminal or the configured polling limit is reached.
9. Worker maps Judge0 responses to testcase result objects.
10. Worker persists per-testcase judge-run records.
11. Worker calculates the aggregate verdict and score.
12. Worker updates the submission record, emits a grading event, and publishes result updates.
13. Frontend displays the result using a socket update or fallback polling.

The deep-dive identifies BullMQ retry configuration as 3 attempts with exponential backoff starting at 2 seconds, and a Judge0 polling limit of 60 seconds. Treat these as current implementation details from the audited snapshot—not approved configuration values.

### 5.2 Evaluation lifecycle states

**[AUDIT—CURRENT]** The submission record is initially reported as `pending`, then finalization writes `graded`. The runtime verdict is stored separately.

**[OPEN DECISION]** The current source materials do not establish the complete canonical lifecycle for:
- queued
- running/executing
- graded/completed
- retrying
- infrastructure failure
- cancelled
- timed out / stuck

Module 17 should define canonical states and legal transitions. Do not introduce new persisted status/verdict enum values in this module without schema approval.

### 5.3 Test-case execution model

**[AUDIT—CURRENT]**
- All retrieved test cases are sent to Judge0 as one batch request.
- Each testcase has its own Judge0 submission payload and token.
- DEZOLVER polls the tokens as a batch.
- The application does not stop dispatching/executing after the first testcase failure.
- Judge0’s internal scheduling/concurrency is external to DEZOLVER; a batch request alone does not guarantee simultaneous execution.

**[DERIVED REQUIREMENT]** DEZOLVER must correlate each Judge0 result with the correct testcase and submission. Correlation must not rely on an unstable or unspecified database return order.

**[OPEN DECISION]** Confirm whether batch execution remains the required design, and define any concurrency, batch-size, or rate-limit constraints for the deployed Judge0 instance.

## 6. Test-case retrieval and management

### 6.1 Retrieval

**[AUDIT—CURRENT]** The code-level report states that the worker requests test cases with hidden cases included for formal evaluation.

**[SOURCE—SCHEMA]** The original schema reconstruction describes a TestCase associated with Problem, with fields including input, expectedOutput, isHidden, and orderIndex.

**[AUDIT—CURRENT / VERSION CONFLICT]** The deep-dive reports a different code-level model (`ProblemTestCase`) with `isSample`, `weight`, and no explicit order field/query ordering.

These models must not be merged without version reconciliation (see Section 15).

### 6.2 Test-case authoring and maintenance

Problem Management owns authoring and lifecycle rules. Evaluation depends on the following properties being valid:

- Test case is associated with the correct Problem.
- Input and expected output are stored in the intended representation.
- Visibility classification is unambiguous.
- Testcase identity and order are stable.
- Scoring weight, if supported, is valid.
- Published Problems have a defined testcase set/version.

**[OPEN DECISION]** This source set does not fully define testcase create/edit/delete permissions, validation, testcase versioning, whether published test cases can be edited, or what happens to old submissions after testcase changes. These should be specified in Module 5 or an approved Problem versioning design.

### 6.3 Ordering

**[AUDIT—CURRENT]** The deep-dive reports no `orderBy` clause in testcase retrieval and no ordering field in the audited code-level model. It also reports testcase result indices based on array position.

**Risk:** without a stable order, displayed testcase indices and “first failure” semantics may change across runs. A database query without explicit ordering has unspecified order; physical row order must not be relied on.

**[DERIVED REQUIREMENT]** Define a stable testcase ordering or use testcase IDs as the canonical identity. Any displayed index should be deterministic for a given Problem version.

## 7. Test-case execution requirements

### EVAL-001 — Evaluate the Problem’s configured testcase set

**[DERIVED REQUIREMENT]** For a valid code submission, evaluate against the testcase set associated with the submitted Problem version and applicable evaluation policy.

**Acceptance criteria**
- Every testcase in the evaluation set is accounted for in the result.
- The result can be traced to the Problem and submission.
- A missing testcase or missing execution result is not silently treated as a pass.

**[OPEN DECISION]** Define whether the testcase set is snapshot/versioned at submission time or loaded at worker execution time. The audit indicates testcase retrieval happens during worker processing, which could allow testcase edits between submission and execution to affect grading.

### EVAL-002 — Execute sample and hidden test cases

**[AUDIT—CURRENT]** The deep-dive reports that sample and hidden cases are included together in formal evaluation.

**[DERIVED REQUIREMENT]** Sample/visible test cases must not be the only cases used for formal grading when hidden cases are configured.

**Acceptance criteria**
- Hidden cases contribute to the aggregate result according to the configured scoring policy.
- The Student-facing result distinguishes visible/sample and hidden cases only as approved.
- Hidden inputs and expected outputs remain confidential.

### EVAL-003 — Continue evaluation after testcase failures

**[AUDIT—CURRENT]** All cases are dispatched upfront and the worker aggregates the full result list; the reported implementation does not short-circuit after a failure.

**[OPEN DECISION]** Confirm whether this is the intended product behavior. Possible reasons to continue include full feedback and partial scoring; possible reasons to stop include resource cost. No alternate policy is assumed here.

### EVAL-004 — Map execution outcome per testcase

The per-testcase result should represent at least:
- testcase identity/index
- sample/hidden classification
- Judge0 execution status
- pass/fail outcome
- execution time
- memory usage
- exit code, where available
- student-visible diagnostic fields permitted by visibility policy

**[AUDIT—CURRENT]** The deep-dive reports a `TestCaseResult` structure containing index, isSample, status, passed, timeMs, wallTimeMs, memoryKb, exitCode, actualOutput, expectedOutput, stderr, and compileOutput.

**[OPEN DECISION]** Confirm which fields are persisted, which are returned to Students, which are staff-only, and which are retained only in internal execution logs.

## 8. Output comparison

### 8.1 Comparison boundary

**[AUDIT—CURRENT]** The application passes expected output to Judge0 and uses Judge0’s returned status ID to determine whether a testcase passed. The DEZOLVER worker does not perform a custom actual-vs-expected output comparison.

### 8.2 Comparator behavior

The deep-dive reports:
- No application-side trimming, case conversion, or newline normalization.
- No custom checker, regex comparator, or comparison script.
- No floating-point tolerance/delta comparison implemented in DEZOLVER.
- Judge0 handles comparison.

**Important verification note:** Exact whitespace/newline behavior must be validated against the deployed Judge0 CE version/configuration. The audit’s whitespace claims should not be promoted to a DEZOLVER guarantee until verified in the actual instance.

**[OPEN DECISION]**
- Is standard Judge0 comparison the intended permanent behavior?
- Should comparison be strict, whitespace-tolerant, token-based, or custom per Problem?
- Are floating-point outputs expected to use exact comparison or tolerance?
- Can a Problem define a custom checker?
- Which Judge0 CE version/configuration is authoritative?

### EVAL-005 — Use an approved comparator policy

**[DERIVED REQUIREMENT]** Every Problem must be evaluated using a known comparator policy. The policy must be consistent for a Problem version and visible to Problem authors where it affects testcase authoring.

**Acceptance criteria**
- Comparator behavior is documented and testable.
- A Problem cannot silently change comparator behavior between attempts unless a version/configuration change is recorded.
- Unsupported comparator modes are not represented as available features.

## 9. Scoring and aggregation

### 9.1 Current code-level scoring formula [AUDIT—CURRENT]

The deep-dive reports weighted partial scoring:

\[
\text{score} =
\begin{cases}
\operatorname{round}\left(100 \times
\frac{\sum_{i \in \text{passed}} w_i}
{\sum_{i=1}^{N} w_i}\right), & \text{if total weight} > 0 \\
0, & \text{otherwise}
\end{cases}
\]

Where:
- \(N\) is the number of returned testcase results.
- \(w_i\) is the corresponding testcase weight, defaulting to 1 in the audited code.
- A testcase contributes its weight only when Judge0 reports Accepted.
- The deep-dive reports JavaScript `Math.round()` for integer percentage rounding.

**Illustration only:** if 3 of 4 equal-weight test cases pass, the reported formula yields 75%.

### 9.2 Weight support and source conflict

**[AUDIT—CURRENT]** The deep-dive reports `ProblemTestCase.weight`, default weight 1, and validation of 1–100 in the audited content DTO.

**[SOURCE—SCHEMA / CONFLICT]** The original schema reconstruction does not list a testcase weight field. It lists `isHidden` and `orderIndex` instead.

**[OPEN DECISION]** Confirm which schema is authoritative for the target release. Until then, weighted scoring is described as current code-audit behavior, not as universally confirmed product/schema behavior.

### 9.3 Partial credit

**[AUDIT—CURRENT]** Partial numeric score is calculated from passed testcase weights.

**Important distinction:** the numeric score can be partial even though the overall verdict may be a failure verdict. The deep-dive reports that a `partial` verdict is effectively unreachable because the verdict is changed to a failure value as soon as a testcase fails, while the later condition attempts to assign `partial` only when the verdict is still `accepted`.

**[OPEN DECISION]**
- Is the intended verdict `PARTIAL` when some cases pass?
- Or should the overall verdict remain the highest-priority failure while score conveys partial credit?
- Should score be integer percentage or preserve decimal precision?
- Should zero-weight test cases be permitted? The audited DTO reportedly disallows weights below 1, but the original schema does not define weight.

### EVAL-006 — Calculate aggregate score

**[DERIVED REQUIREMENT]** The evaluation result must contain a deterministic score based on the approved scoring policy.

**Acceptance criteria**
- Score is within the configured valid range.
- No testcase result is counted more than once.
- Missing/failed execution results are not counted as passing.
- Rounding and precision are defined.
- The score calculation can be reproduced from the stored testcase results and weights.

### EVAL-007 — Aggregate testcase results into a final verdict

**[AUDIT—CURRENT]** The deep-dive reports that the audited implementation initializes verdict as accepted and assigns the first encountered failure type. The comment says the intended priority is TLE > MLE > CE > RE > WA, but the code’s condition only applies while the verdict remains accepted. Therefore, the actual result is first-failure-in-array order, not the stated global priority.

**[DERIVED REQUIREMENT]** The final verdict policy must be explicitly approved and implemented consistently. Do not treat the current first-failure behavior as an intentional product rule without approval.

**[OPEN DECISION]** Define:
- Verdict precedence when multiple testcase failures exist.
- Whether partial score should produce a distinct PARTIAL verdict.
- What verdict applies when some execution results are missing or infrastructure failures occur.
- Whether overall ACCEPTED requires every required testcase to pass.

## 10. Failure and execution outcome handling

### 10.1 Outcomes reported by the audit

The deep-dive reports Judge0 status mappings such as:
- Accepted
- Wrong Answer
- Time Limit Exceeded
- Memory Limit Exceeded
- Compilation Error
- Runtime Error (including status IDs in the reported 7–11 range)
- Other/unrecognized outcome → internal error in the reported aggregation logic

These are current code-level mappings and should be reconciled with the deployed Judge0 version and DEZOLVER’s persisted enum.

### 10.2 Compilation errors

**[AUDIT—CURRENT]**
- Compilation errors are reported for testcase executions.
- Score remains 0 for failing compilation cases.
- Compiler output is captured in the reported result structures and raw judge-run persistence.

**[DERIVED REQUIREMENT]** Students may receive useful compilation feedback, but hidden testcase data must not be exposed through compiler output or any other diagnostics.

**[OPEN DECISION]** Determine whether compile diagnostics are stored once per submission or duplicated across per-testcase results, and what limits/sanitization apply.

### 10.3 Runtime errors, TLE, MLE, and wrong answers

**[AUDIT—CURRENT]** A testcase that does not return Accepted contributes no passed weight. Other cases continue to run in the audited batch model, allowing numeric partial score.

**[DERIVED REQUIREMENT]** Runtime failure must not be interpreted as a pass. Resource-limit outcomes must be mapped to the approved verdict and score policy.

### 10.4 Infrastructure and incomplete execution failures

The sources do not fully define the product behavior for:
- Judge0 unavailable / request rejected
- Batch submission partially accepted
- Polling timeout
- Judge0 token missing or invalid
- Result list missing or shorter/longer than testcase list
- BullMQ retry exhaustion
- Worker crash
- Submission stuck pending/running
- Duplicate execution or duplicate grading event

**[DERIVED REQUIREMENT]** Infrastructure failure must be distinguishable from a Student’s incorrect code. It must not silently award zero as if all missing cases were wrong answers, nor award a pass. The system should support a recoverable state and operational visibility.

**[OPEN DECISION]** Exact status, retry, cancellation, requeue, and user-facing message rules are owned jointly by Modules 17 and 18 and the Judge0 integration design.

## 11. Testcase index and failure pointer

### 11.1 Current implementation report

**[AUDIT—CURRENT]**
- The deep-dive reports no `failedTestCaseIndex` field generated by the audited worker.
- Per-testcase results use `index: i + 1` (one-based), based on array position.
- The API returns the complete testcase result array.
- The frontend renders the array rather than relying on one scalar failed-test pointer.

**[SOURCE—SCHEMA / CONFLICT]** The original schema reconstruction includes `failedTestCaseIndex` on CodeSubmission.

### 11.2 Required clarification

**[OPEN DECISION]**
- Is `failedTestCaseIndex` required in the target schema/API?
- If so, is it zero-based or one-based?
- Does it identify first failed, highest-priority failed, or another testcase?
- Is it meaningful for compilation errors, infrastructure errors, or hidden cases?
- Should it be replaced by a list of failed testcase IDs/results?

**[DERIVED REQUIREMENT]** Stable testcase identity should be used internally. A displayed ordinal is not a durable identifier if testcase order can change.

## 12. Hidden testcase confidentiality and result visibility

### 12.1 Reported current behavior [AUDIT—CURRENT]

The deep-dive reports:
- Hidden actual output and expected output are omitted from Student-facing testcase results.
- Problem retrieval excludes hidden testcase content from the Problem statement response.
- Hidden testcase status, runtime, and memory are displayed.
- `stderr` and `compileOutput` are assigned without the same hidden-case masking in the audited code. The report flags this as a potential information leak.

### 12.2 Required confidentiality policy

**[DERIVED REQUIREMENT]**
- Hidden input and expected output must never be exposed to Students.
- Hidden actual output, stderr, compiler output, raw Judge0 responses, and any diagnostic that could reveal hidden inputs must be masked or filtered before Student-facing serialization.
- Masking must be applied server-side, not only in the frontend.
- Raw execution artifacts must have role-scoped access and retention rules.
- Logs and error messages must not accidentally contain hidden testcase values.
- Hidden testcase counts/pass/fail feedback may be shown only according to the approved product policy.

**Acceptance criteria**
- Student API responses do not contain hidden input or expected output.
- Student responses do not contain unfiltered hidden stdout/stderr/compiler output.
- Direct API calls cannot bypass the hidden-data policy.
- Staff access to raw hidden-case details is permission-checked and auditable where required.

**[OPEN DECISION]** Confirm whether students may see hidden testcase pass/fail, hidden case ordinal, runtime/memory, and aggregate hidden-case count.

## 13. Persistence and result retrieval

### 13.1 Reported current persistence [AUDIT—CURRENT]

The deep-dive reports:
- Per-testcase execution data is stored as rows in `assessment.judge_runs`.
- Submission aggregate fields are updated on `assessment.submissions`.
- Aggregate fields include verdict, score, passed/total testcase counts, testcase result JSON, max runtime/memory, and graded timestamp.
- A grading event is recorded in an outbox and published to Redis channels.
- Frontend receives updates through Socket.io or fallback polling.

The exact persistence model must be checked against the authoritative schema snapshot because the original schema reconstruction uses a different CodeSubmission representation.

### 13.2 Result retrieval

**[AUDIT—CURRENT]** The report identifies:
- Submission creation endpoint: `POST /api/v1/assessments/:id/submissions`
- Submission result endpoint: `GET /api/v1/submissions/:id`
- Rerun endpoint: `POST /api/v1/submissions/:id/rerun`
- Raw Judge run endpoint: `GET /api/v1/judge-runs/:id`

These are reported endpoint paths from the code audit. They should be treated as implementation references, not an API contract until the target release is confirmed.

**[DERIVED REQUIREMENT]**
- Student result retrieval must enforce submission ownership.
- Staff retrieval must enforce academic scope and permission.
- Student response DTOs must exclude hidden testcase secrets and privileged raw data.
- API responses must distinguish pending/running from completed evaluation.
- Result payload should expose score, verdict, passed/total counts, and only approved testcase feedback.

## 14. Re-evaluation / rejudge

### 14.1 Reported current behavior [AUDIT—CURRENT]

The deep-dive reports:
- A dedicated rerun endpoint exists.
- Rerun is restricted to platform administrator/faculty roles with assessment-management permission.
- Rerun creates fresh Judge0 executions/tokens.
- The main submission row is updated in place; previous aggregate verdict/score/result JSON are overwritten.
- Per-testcase judge-run history is retained in `assessment.judge_runs`.
- Contest results/leaderboards are reportedly recalculated through a grading event.
- Course/path progress is reportedly not updated due to an event-name mismatch in the audited code.

### 14.2 Functional requirements

**[DERIVED REQUIREMENT]**
- Re-evaluation must be permission-controlled.
- A rerun must identify the submission being re-evaluated and create a new execution attempt.
- A rerun must not be mistaken for a new Student submission/attempt unless the owning module explicitly defines that behavior.
- Re-evaluation must update downstream score/rank/progress only according to each module’s approved event contract.
- The system should preserve enough history to explain changes in grading.

**[OPEN DECISION]**
- Is in-place overwrite the intended business behavior?
- Should a grading-version/history record preserve old aggregate scores and verdicts?
- What happens if a rerun fails after the old result is reset to pending?
- Who may request rejudge, and must a reason be supplied?
- Should rejudge trigger notifications or audit events?
- How are changed testcases/problem versions handled during rejudge?

## 15. Source-version discrepancies and reconciliation

This is a high-priority review section because the deep-dive and original schema reconstruction do not align.

| Topic | Original schema reconstruction | Deep-dive code audit | Needed action |
|---|---|---|---|
| Testcase model name | `TestCase` | `ProblemTestCase` | Confirm target schema/model |
| Visibility flag | `isHidden` | `isSample` | Confirm mapping/meaning; avoid assuming they are direct inverses |
| Testcase ordering | `orderIndex` | No order column/orderBy reported | Confirm target ordering model |
| Testcase weight | Not listed | `weight` reported, default 1 | Confirm whether weighted scoring is in target release |
| Submission failure pointer | `failedTestCaseIndex` listed | Report says field is not generated in audited code | Confirm schema/API contract |
| Verdict enum | `PENDING`, `RUNNING`, `ACCEPTED`, `WRONG_ANSWER`, `TLE`, `MLE`, `RE`, `CE` in reconstructed schema | Code audit reports values such as `partial` and `internal_error` as possible strings | Confirm enum, database type, and serialization |
| Persistence model | `CodeSubmission` entity with judge0Token and aggregate fields | `assessment.submissions` + `assessment.judge_runs` in audited code | Confirm whether this is a newer schema or a separate assessment subsystem |

**[REQUIRED BEFORE FINAL SPECIFICATION]** Confirm which source snapshot is the target implementation. Until then, all conflicting properties are marked as version-dependent and must not be silently merged.

## 16. Acceptance criteria

The following are proposed high-level acceptance criteria for the evaluation feature. They require confirmation where they introduce intended behavior beyond the audited implementation.

- [ ] An eligible submission is evaluated against the approved testcase set for the Problem version.
- [ ] Sample and hidden testcases participate according to the approved evaluation policy.
- [ ] Each testcase result is correlated to the correct testcase and submission.
- [ ] Evaluation records pass/fail and approved runtime/memory diagnostics for each testcase.
- [ ] The comparator policy is documented and reproducible for a Problem version.
- [ ] Aggregate score follows the approved weighting, precision, and rounding rules.
- [ ] Overall verdict follows an explicitly approved precedence/partial-credit policy.
- [ ] Compilation/runtime/resource-limit failures do not count as passed testcases.
- [ ] Missing/incomplete Judge0 results are handled as evaluation/infrastructure failures, not silently treated as passes.
- [ ] Hidden testcase secrets are never returned through Student-facing APIs or UI.
- [ ] Student and staff result access is permission-checked.
- [ ] Re-evaluation is authorized, traceable, and updates downstream modules according to approved event contracts.
- [ ] Result history/version behavior is documented.
- [ ] Evaluation behavior is testable against the deployed Judge0 version/configuration.

## 17. Test scenarios for review and QA

| ID | Scenario | Expected behavior to confirm |
|---|---|---|
| TC-EVAL-01 | All testcases pass | Accepted verdict and full score under approved policy |
| TC-EVAL-02 | One visible testcase fails | Correct testcase result, aggregate score/verdict |
| TC-EVAL-03 | One hidden testcase fails | Score/verdict reflect failure; hidden data remains confidential |
| TC-EVAL-04 | Some cases pass and some fail | Partial score behavior and final verdict follow approved rules |
| TC-EVAL-05 | Compilation error | Zero testcase credit where applicable; safe compiler diagnostics |
| TC-EVAL-06 | Runtime error on one testcase | Failure status and score aggregation are correct |
| TC-EVAL-07 | Time limit exceeded | TLE mapping and aggregate precedence are correct |
| TC-EVAL-08 | Memory limit exceeded | MLE mapping and aggregate precedence are correct |
| TC-EVAL-09 | Multiple different failures | Final verdict follows approved precedence, not accidental result order |
| TC-EVAL-10 | Empty expected output | Empty output is handled intentionally and correctly |
| TC-EVAL-11 | Whitespace/newline differences | Actual deployed comparator behavior matches documented policy |
| TC-EVAL-12 | Floating-point output | Exact or tolerance-based behavior matches Problem policy |
| TC-EVAL-13 | Judge0 unavailable / request rejected | Submission enters recoverable error/retry state; not misgraded as WA |
| TC-EVAL-14 | Poll timeout / missing token result | Incomplete evaluation is detected and recoverable |
| TC-EVAL-15 | Testcase set changes before worker execution | Evaluation uses the approved snapshot/version policy |
| TC-EVAL-16 | Testcase order changes | Stable identity/indexing prevents misleading failure index |
| TC-EVAL-17 | Student requests another student’s result | Access denied |
| TC-EVAL-18 | Student inspects hidden testcase response | No hidden input/output/stderr/compiler output leakage |
| TC-EVAL-19 | Authorized staff rejudge | Fresh execution, traceability, approved history/downstream update |
| TC-EVAL-20 | Rejudge fails midway | Existing/current result behavior follows approved recovery policy |

## 18. Original source questions — resolved for the v1 target

The original draft listed 27 questions spanning activity contexts, partial credit, verdict precedence, hidden feedback, staff access, rejudge, testcase schema/weight/identity, Judge0 mapping/comparator, resource limits, retries, versioning, persistence, events, retention, raw payloads, output caps, audit and monitoring.

All EVAL-D01–EVAL-D27 are adopted. The original questions are preserved as review-scope history, not current open decisions. Resolved target behavior is specified in Sections 30–33 and ADOPTED-DECISIONS.md.

Engineering implementation, deployed Judge0 verification, integrations and QA execution remain pending.

## 19. Dependencies

- Module 5 — Problem Management: Problem metadata, test-case authoring, visibility, ordering, versioning, time/memory limits, allowed languages.
- Module 6 — Code Submission & Judge0: submission intake, code/language, Judge0 request/correlation.
- Module 8 — Labs & Lab Scoring: mapping evaluation score to lab score.
- Module 9 — Assessments: assessment attempts, assessment-level score and status.
- Module 11 — Contests: contest points, penalty, ranking and tie-breaking.
- Module 13 — Notifications: submission graded / regraded notifications.
- Module 15 — Audit Logging: testcase changes, raw result access, rejudge events.
- Module 17 — Functional State Machines: submission/evaluation lifecycle.
- Module 18 — Edge Cases & Failure Handling: Judge0/queue/worker failures and recovery.

## 20. Source references

- Business Flow v2: `docs/source-text/Business Flow v2.md`
- Page-to-DB Mapping v2: `docs/source-text/Page - DB Mapping v2.md`
- Schema reconstruction: `docs/source-text/scheema.md`
- Module 5: `FUNCTIONAL-SPECIFICATION/5. Problem Management/README.md`
- Module 6: `FUNCTIONAL-SPECIFICATION/6. Code Submission & Judge0/README.md`
- Deep-dive report: `DEZOLVER_EVALUATION_ALGORITHM_AND_TEST_CASE_SCORING_DEEP_DIVE.md`

Deep-dive code references cited in the report include:
- `src/worker/consumers/code-execution.consumer.ts`
- `src/assessment/judge/judge0.client.ts`
- `src/assessment/services/submission.service.ts`
- `src/assessment/repositories/submissions.repository.ts`
- `src/content/repositories/problems.repository.ts`
- `src/assessment/controllers/submissions.controller.ts`
- `frontend/src/components/assessment/VerdictPanel.tsx`
- `frontend/src/hooks/useSubmissions.ts`
- Prisma assessment/content schemas

---

**Review checkpoint:** The user approved this source-grounded draft for repository inclusion. Module 7 is committed in its own folder. This does not resolve the open business, technical, or security decisions listed above. Do not begin Module 8 until the user separately approves proceeding.

---

# Functional Specification Standard v1.0 compliance expansion

> This expansion supplements EVAL-001–EVAL-007. It distinguishes conceptual schema, audit-reported legacy behavior, derived safeguards, and target decisions. Function IDs use TEST-Fxxx. This documentation does not approve legacy behavior.

## 21. Evidence and status labels

- **SOURCE—BUSINESS:** Business Flow v2 / Page-to-DB Mapping v2.
- **SOURCE—SCHEMA:** conceptual schema reconstruction.
- **AUDIT—CURRENT:** behavior reported by the supplied source-code forensic report for its investigated snapshot.
- **DERIVED REQUIREMENT:** safeguard inferred from sources.
- **TARGET REQUIREMENT:** not approved unless product owner confirms.
- **OPEN DECISION:** source gaps/conflicts requiring review.

Audit claims are not runtime verification of the current deployment and must not automatically become target requirements.

## 22. Terminology

| Term | Definition | Status |
|---|---|---|
| TestCase | Conceptual Problem input/expectedOutput record with isHidden and orderIndex | Schema |
| ProblemTestCase | Legacy audited model in content.problem_test_cases with isSample and weight | Audit |
| Sample testcase | Legacy isSample=true; sample details reportedly shown in UI | Audit |
| Hidden testcase | Conceptual isHidden=true or legacy isSample=false; equivalence unresolved | Conflict |
| JudgeRun | Legacy per-testcase execution record | Audit |
| TestCaseResult | Per-testcase result DTO | Audit |
| Rejudge | Re-evaluation of existing submission | Audit + target TBD |
| Snapshot | Testcase/Problem/config version used for a run | TBD |

## 23. Role, action, and scope matrix

| Role | View evaluation | Raw JudgeRun details | Rejudge | Scope |
|---|---|---|---|---|
| SUPER_ADMIN | View/Manage conceptual Code Submission | Raw access policy TBD | Audit reports legacy platform_admin; role mapping TBD | Global |
| FINANCE_TEAM | No Code Submission permission | Deny unless explicitly granted | Deny | Finance College scope only |
| HOD | View | Field-level access TBD | Not explicitly granted by conceptual RBAC | College + Department |
| MANAGER | View* | Field-level access TBD | Not granted by conceptual RBAC | College + Department + assigned Section/permitted Group |
| STUDENT | View own | Filtered result only | Deny | Own + permitted academic resources |

Role, scope, ownership, and field-level permission are separate checks. Legacy faculty/platform_admin roles are not assumed to map to DEZOLVER roles.

## 24. Functional inventory

| Function ID | Function | Actor | Entry point | Type | Status |
|---|---|---|---|---|---|
| TEST-F001 | Resolve testcase set | Worker | Submission job | Workflow/data | Draft |
| TEST-F002 | Bind testcase set to Problem/run version | System | Evaluation start | Integrity/version | TBD |
| TEST-F003 | Build per-testcase Judge0 executions | Worker | Testcase set resolved | Integration | Audit-reported |
| TEST-F004 | Batch-submit executions | Worker | Evaluation job | External integration | Audit-reported |
| TEST-F005 | Correlate Judge0 tokens/results | Worker | Judge0 response | Integrity | Draft |
| TEST-F006 | Poll/collect statuses | Worker | Tokens issued | Workflow | Audit-reported |
| TEST-F007 | Map Judge0 status | Worker | Terminal result | Evaluation | Draft |
| TEST-F008 | Compare output | Judge0/DEZOLVER | Testcase run | Comparator | Audit-reported |
| TEST-F009 | Evaluate visible/sample cases | Worker | Testcase run | Evaluation | Draft |
| TEST-F010 | Evaluate hidden cases | Worker | Testcase run | Evaluation/security | Draft |
| TEST-F011 | Mask hidden testcase details | Serializer/API | Result output | Security/privacy | Derived |
| TEST-F012 | Persist raw JudgeRun data | Worker | Testcase completion | Data | Audit-reported |
| TEST-F013 | Build TestCaseResult DTO | Worker | Result aggregation | Data mapping | Audit-reported |
| TEST-F014 | Calculate aggregate score | Worker | Testcase outcomes | Calculation | Audit-reported |
| TEST-F015 | Aggregate final verdict | Worker | Testcase outcomes | Calculation | Audit-reported |
| TEST-F016 | Handle incomplete results | Worker | Poll/batch failure | Failure | Draft |
| TEST-F017 | Update aggregate submission | Worker | Evaluation complete | Data | Audit-reported |
| TEST-F018 | Retrieve Student result | Student/API | Result panel/poll | Read | Draft |
| TEST-F019 | Retrieve staff result | Staff/API | Staff page | Read | Draft |
| TEST-F020 | Rejudge submission | Authorized staff | Rerun action | Admin workflow | Audit-reported/TBD |
| TEST-F021 | Preserve run history | System | Rerun | History | Audit-reported |
| TEST-F022 | Emit grading event | Worker/outbox | Final result | Integration | Audit-reported |
| TEST-F023 | Apply testcase identity/order/version | System | Every evaluation | Integrity | TBD |
| TEST-F024 | Protect raw output and diagnostics | System | Storage/response/log | Security | Derived |
| TEST-F025 | Monitor stuck evaluation | Operations | Queue/worker/poll | Operations | TBD |

## 25. Detailed function specifications

### TEST-F001 — Resolve evaluation testcase set
**Requirement status:** audit-reported + schema conflict. **Specification status:** Decision required. **Actors:** worker/Problem repository.

**Purpose:** identify the official testcases for a submission. **Permissions/scope:** only for an authorized persisted submission and its Problem/context. **Preconditions:** submission, Problem and execution config exist. **Entry:** evaluation job.

**Inputs:** submission ID, Problem ID/version, testcase collection, visibility/order/weight where supported.

**Main flow:** load submission → resolve Problem → load configured testcases including official hidden cases → validate IDs/fields → bind set to run.

**Rules:** Module 5 owns authoring; audit reports sample+hidden retrieval from legacy content.problem_test_cases; conceptual schema instead lists TestCase.isHidden/orderIndex. Do not merge schemas silently.

**Validation:** missing Problem/testcases or invalid testcase data → configuration/incomplete failure, not Accepted. **Data:** read-only; snapshot timing TBD.

**Failure/edge:** empty set, testcase edit/delete after submit, cross-College Problem, unstable DB order.

**Acceptance:** run identifies its testcase set. **Decision:** EVAL-D09/D10/D20/D21; Module 5.

### TEST-F002 — Bind testcase set to Problem/run version
**Status:** derived integrity requirement; snapshot model TBD.

**Purpose:** ensure pending/historical evaluation has reproducible meaning. **Actors:** evaluation service/Problem owner. **Inputs:** submission timestamp, Problem/testcase/config version.

**Flow:** resolve approved version policy → record version/snapshot identity → evaluate against that exact set. Audit reports testcase retrieval at worker processing, potentially after submission; conceptual schema lacks version fields.

**Rules:** no snapshot mechanism assumed. **Validation:** version unavailable → stop or approved fallback. **Data:** snapshot schema/strategy requires decision.

**Edge:** Problem edit while queued, testcase reorder/delete, active contest/assessment. **Acceptance:** results can be reproduced. **Decision:** EVAL-D20/D21; Modules 5/17.

### TEST-F003 — Build per-testcase Judge0 executions
**Status:** legacy audit-reported.

**Inputs:** code, language ID, testcase stdin/expected output, resource limits. **Preconditions:** testcase set and configuration validated.

**Flow:** build one Judge0 execution per testcase → bind to stable testcase ID → encode fields → prepare batch.

**Rules:** expected output is server-controlled; browser cannot override official tests. Audit reports base64 encoding and batch construction.

**Validation:** unsupported language, missing testcase input/output, invalid limits → configuration failure. **Edge:** large batch, encoding error, language mapping drift.

**Acceptance:** every request maps to one testcase/submission. **Decision:** EVAL-D09/D10/D17.

### TEST-F004 — Batch-submit executions
**Status:** audit-reported; target architecture TBD.

**Inputs:** execution batch, endpoint/auth config. **Audit:** POST /submissions/batch?base64_encoded=true; one token per testcase reported.

**Flow:** call configured Judge0 → validate response → verify token count/shape → correlate tokens. **Rules:** server-side credentials; batch submission does not guarantee parallel execution.

**Failure:** non-2xx, malformed response, missing/duplicate token, partial batch → infrastructure failure, not Wrong Answer.

**Edge:** Judge0 accepted some requests before timeout. **Acceptance:** token-to-testcase correlation is correct. **Decision:** EVAL-D19/D21; Module 6.

### TEST-F005 — Correlate Judge0 tokens/results
**Status:** derived integrity.

**Inputs:** submission/run ID, testcase ID, token, result. **Flow:** store mapping → poll by token → attach response to correct testcase/run.

**Rules:** never rely on unspecified database array order. Audit reports one-based result indices; conceptual schema has failedTestCaseIndex but audit says that field was not generated.

**Validation:** missing/duplicate token or count mismatch → incomplete run. **Edge:** retries create fresh tokens, result order changes.

**Acceptance:** unique correct mapping. **Decision:** EVAL-D11/D12/D21.

### TEST-F006 — Poll and collect execution statuses
**Status:** audit-reported.

**Audit details:** batch GET polling; 1-second initial interval, +0.5 seconds to 4-second cap; 60-second deadline; timeout reportedly returns partial results. These values are not target-approved.

**Target flow:** poll under approved deadline/backoff → stop when terminal or explicit timeout/error → preserve incomplete-token list.

**Rules:** queued/processing is not a final verdict; incomplete cases cannot silently count as WA/pass. **Edge:** token timeout, network failure, token expiration.

**Acceptance:** bounded polling and explicit incomplete status. **Decision:** EVAL-D19; Module 18.

### TEST-F007 — Map Judge0 status to testcase result
**Status:** audit-reported; mapping needs approval.

**Audit status IDs:** 3 Accepted; 4 Wrong Answer; 5 TLE; 6 Compilation Error; 7–11 Runtime Error; 12 MLE; unknown statuses may map to internal_error.

**Flow:** validate terminal status → map canonical outcome → attach metrics → mark pass only under approved rule.

**Rules:** unknown/incomplete statuses never count as Accepted. **Edge:** Judge0 version introduces new status. **Acceptance:** versioned mapping and test coverage. **Decision:** EVAL-D13/D14.

### TEST-F008 — Compare program output
**Status:** audit says comparator delegated to Judge0; target policy TBD.

**Inputs:** actual output, expected output, comparator config. **Flow:** send expected output to Judge0 → use returned status unless custom checker approved.

**Rules:** audit reports no DEZOLVER string comparison, float tolerance, custom checker, or epsilon; exact whitespace/newline behavior must be validated against deployed Judge0.

**Validation:** unsupported comparator → reject config or approved fallback. **Edge:** trailing whitespace, case, newline, floats, empty output.

**Acceptance:** comparator is documented/testable. **Decision:** EVAL-D14/D15/D16.

### TEST-F009 — Evaluate visible/sample testcases
**Status:** conceptual/audit conflict.

**Inputs:** visibility flag, input, expected output. **Flow:** execute sample → persist outcome → expose approved sample feedback.

**Rules:** conceptual isHidden=false and legacy isSample=true are not assumed equivalent until schema mapping approved. **Validation:** missing visibility must not default to student-visible.

**Edge:** sample changed during rejudge. **Acceptance:** response follows approved visibility classification. **Decision:** EVAL-D10/D12.

### TEST-F010 — Evaluate hidden testcases
**Status:** schema + security-derived.

**Flow:** execute hidden case → persist internal result → calculate score/verdict → return filtered feedback.

**Rules:** hidden cases participate when configured; hidden input/expected output never student-visible; hidden pass/fail/ordinal/runtime policy requires approval.

**Edge:** stderr/compiler output leaks, count/timing side channels. **Acceptance:** all transport outputs respect confidentiality. **Decision:** EVAL-D12/D23.

### TEST-F011 — Mask hidden testcase details
**Status:** critical derived requirement.

**Inputs:** caller role/owner, testcase visibility, raw result. **Flow:** authorize → classify case → apply server-side allowlist/masking → serialize REST/socket.

**Rules:** frontend hiding alone is insufficient. Audit reports hidden actual/expected output masked, but stderr/compileOutput reportedly not masked consistently.

**Validation:** unknown visibility → restrictive response. **Edge:** direct API, socket, log, malicious stderr, compile output. **Acceptance:** hidden input/expected/actual and sensitive diagnostics cannot be fetched by Student. **Decision:** EVAL-D12/D23.

### TEST-F012 — Persist raw JudgeRun data
**Status:** audit-reported; conceptual schema conflict.

**Inputs:** submission, testcase, token, raw response, run attempt. **Flow:** verify correlation → persist raw run according to retention/access policy → maintain history.

**Rules:** raw data is sensitive; not the default Student DTO. Conceptual CodeSubmission vs legacy judge_runs persistence must be reconciled.

**Failure:** DB failure after external execution; retry duplicate run; rerun. **Acceptance:** raw runs traceable, policy-controlled, not overwritten accidentally. **Decision:** EVAL-D21/D23.

### TEST-F013 — Build TestCaseResult payload
**Status:** audit-reported.

**Audit fields:** 1-based index, isSample, status, passed, timeMs, wallTimeMs, memoryKb, exitCode, actualOutput, expectedOutput, stderr, compileOutput.

**Flow:** map raw results → derive pass/fail → convert units → attach testcase identity → filter by visibility/role.

**Rules:** display index is not durable identity; failedTestCaseIndex conflict remains. Missing values cannot be fabricated as pass.

**Edge:** case order changes, missing exit code/output. **Acceptance:** stable safe DTO. **Decision:** EVAL-D11/D12/D23.

### TEST-F014 — Calculate aggregate score
**Status:** audit-reported; conceptual schema lacks weight.

**Legacy formula:** round(100 × passedWeight / totalWeight), else 0 when totalWeight is zero; weight defaults to 1; only Judge0 status 3 contributes. JS Math.round reported. Audit says legacy DTO weight range 1–100; verify source version.

**Flow:** aggregate passed count/weight → compute percentage → persist. **Rules:** do not adopt weighted scoring target until schema/academic owner approves.

**Validation:** invalid/zero weight, missing results, duplicated case. **Acceptance:** deterministic reproducible score; missing cases cannot award credit. **Decision:** EVAL-D09/D10/D18.

### TEST-F015 — Aggregate final verdict
**Status:** audit reports a defect; target semantics open.

**Audit finding:** verdict initialized accepted and first failure changes it; comment suggests priority TLE > MLE > CE > RE > WA, but condition only assigns while verdict remains accepted. Thus reported actual behavior is first failure in result array. The partial verdict condition appears unreachable after failure.

**Flow:** collect all outcomes → apply approved precedence/partial policy → persist verdict separately from score/infrastructure status.

**Rules:** do not adopt first-failure behavior without approval; ACCEPTED requires all required cases pass under policy; PARTIAL is absent from conceptual enum.

**Edge:** mixed failure types, partial credit, incomplete cases. **Acceptance:** deterministic schema-valid verdict. **Decision:** EVAL-D13/D18/D21.

### TEST-F016 — Handle missing/incomplete results
**Status:** derived reliability.

**Triggers:** timeout, missing token, partial batch, worker crash, Judge0 outage. **Flow:** compare expected vs received results → classify incomplete → do not count missing case as pass/WA → recover per Module 18.

**Rules:** legacy partial polling must not produce misleading normal grade. **Data:** operational state requires schema/state decision.

**Edge:** one token pending while others terminal; late result after retry. **Acceptance:** no incomplete evaluation is Accepted/full score. **Decision:** EVAL-D19; Modules 17/18.

### TEST-F017 — Update aggregate submission result
**Status:** audit-reported.

**Audit model:** assessment.submissions updated with status graded, verdict, score, passed/total, testcase result JSON, max runtime/memory, gradedAt. Conceptual CodeSubmission differs.

**Flow:** verify run current → persist per-case records → update aggregate atomically/approved transaction → emit event.

**Rules:** stale retry cannot overwrite newer run; incomplete results cannot be normal graded success. **Edge:** DB/event failure, rerun race.

**Acceptance:** aggregate matches correlated run; event after committed grade. **Decision:** EVAL-D21/D22.

### TEST-F018 — Retrieve Student result
**Status:** RBAC-derived; endpoint is audit reference.

**Flow:** authenticate → verify owner/context → load aggregate → return safe status/score/feedback → mask hidden data.

**Audit endpoint:** GET /api/v1/submissions/:id; not target API contract. **Rules:** own submission only; pending/running distinct from terminal; no hidden diagnostics.

**Edge:** foreign ID, stale poll, rejudge in progress. **Acceptance:** server-side ownership and safe result. **Decision:** EVAL-D12/D23; Module 6.

### TEST-F019 — Retrieve staff result
**Status:** RBAC-derived.

**Actors/scope:** HOD College+Department; Manager assigned Section/group; SUPER_ADMIN global; Finance denied absent grant.

**Flow:** check role/action/scope → authorize fields → return staff DTO. **Rules:** staff result view does not automatically permit raw JudgeRun/token/source access.

**Edge:** staff scope changed, raw diagnostics. **Acceptance:** every result field has authorization. **Decision:** EVAL-D22/D23.

### TEST-F020 — Rejudge submission
**Status:** audit-reported; target policy open.

**Audit:** rerun endpoint POST /api/v1/submissions/:id/rerun, legacy manage:assessment + platform_admin/faculty checks; resets aggregate pending and enqueues fresh execution tokens.

**Flow:** authorize → validate state → record actor/reason if required → run against approved testcase version → update aggregate → trigger approved downstream events.

**Rules:** rerun is not a new Student attempt by default; legacy role mapping not assumed. **Edge:** concurrent rerun, finalized contest, testcase changed.

**Acceptance:** authorized, traceable, version-aware rejudge. **Decision:** EVAL-D22/D20.

### TEST-F021 — Preserve rejudge execution history
**Status:** audit-reported, target decision open.

**Audit:** aggregate submission overwritten, per-testcase JudgeRun rows retained. **Flow:** create run generation → persist new raw run set → link to submission → mark current aggregate generation → retain prior run history per policy.

**Rules:** current result vs prior run clearly distinguished. **Edge:** multiple reruns, failed rerun, retention. **Acceptance:** history explains result changes. **Decision:** EVAL-D21/D22.

### TEST-F022 — Emit grading event
**Status:** audit-reported.

**Audit:** worker emits SubmissionGraded/outbox/Redis; contest consumer reportedly consumes accepted/partial; Course progress consumer listens for SubmissionAccepted, causing event mismatch.

**Flow:** commit result → persist canonical event → publish/retry → consumers process idempotently.

**Rules:** Module 8/9/11 own academic scoring; Module 13 notifications; Module 15 audit. No score/progress side effect before committed grade.

**Edge:** duplicate/missing event, rerun event. **Acceptance:** no double-counting; event mismatch resolved by architecture. **Decision:** EVAL-D24/D25.

### TEST-F023 — Apply testcase identity/order/version
**Status:** schema conflict.

**Conceptual:** TestCase.isHidden/orderIndex. **Audit:** ProblemTestCase.isSample/weight; no explicit ordering reported; result indices one-based.

**Flow:** identify stable testcase/version → approved order → associate run by ID → derive display index.

**Rules:** index is not durable identity; failedTestCaseIndex exists conceptually but audit says absent. **Edge:** reorder/delete during rerun.

**Acceptance:** displayed result corresponds to executed testcase. **Decision:** EVAL-D11/D20/D21.

### TEST-F024 — Protect raw output and diagnostics
**Status:** derived security/privacy.

**Inputs:** stdout/stderr/compiler output/actual/expected/token/raw Judge0 response. **Flow:** classify sensitive artifacts → store under access/retention policy → safe DTO allowlist → redact logs/errors.

**Rules:** hidden data and secrets never reach unauthorized Student; output truncation policy must be approved. Audit reports stderr/compileOutput exposure and no app-side truncation.

**Edge:** code prints hidden input to stderr; large output; raw token leak. **Acceptance:** server-side masking and access control across API/socket/logs. **Decision:** EVAL-D23.

### TEST-F025 — Monitor stuck evaluation
**Status:** derived operational requirement.

**Inputs:** queued/running age, token status, retries, errors, queue depth. **Flow:** monitor lifecycle → detect overdue/incomplete run → surface actionable status → recover per Module 18.

**Rules:** operational failure distinct from code verdict; metrics cannot expose source/testcase secrets. **Edge:** queue backlog, worker crash, Judge0 outage.

**Acceptance:** operations can identify/recover stuck jobs safely. **Decision:** EVAL-D19/D27; Module 18.

## 26. Business rules and validation matrix

| Rule ID | Rule | Evidence/status |
|---|---|---|
| TEST-R01 | Evaluation uses Problem’s configured testcase set | Derived |
| TEST-R02 | Sample/hidden execution follows approved policy | Audit-reported |
| TEST-R03 | One Judge0 execution per testcase in audit | Audit |
| TEST-R04 | Result must correlate to stable testcase identity | Derived |
| TEST-R05 | Legacy status 3 contributes passed weight | Audit |
| TEST-R06 | Legacy weighted score is passed-weight percentage rounded by Math.round | Audit; target TBD |
| TEST-R07 | Conceptual schema has no testcase weight | Source conflict |
| TEST-R08 | Legacy verdict is first-failure order, not stated global precedence | Audit finding |
| TEST-R09 | Legacy PARTIAL condition appears unreachable | Audit finding |
| TEST-R10 | Conceptual and legacy verdict/state models differ | Conflict |
| TEST-R11 | Hidden inputs/expected outputs must not be Student-visible | Security derived |
| TEST-R12 | Hidden stderr/compiler output needs masking | Audit warning |
| TEST-R13 | Incomplete result cannot count as pass/normal WA by default | Derived |
| TEST-R14 | Rejudge is not a new Student attempt by default | Derived |
| TEST-R15 | Historical result requires testcase/version policy | TBD |
| TEST-R16 | Comparator is external/configuration-dependent | Audit |
| TEST-R17 | No float tolerance/custom comparator reported | Audit |
| TEST-R18 | Ordering cannot rely on unspecified DB return order | Derived |
| TEST-R19 | Grading event consumers must be idempotent | Derived |
| TEST-R20 | Raw run history vs aggregate overwrite is not target-approved | Audit/TBD |

| Validation ID | Condition | Expected behavior | Decision |
|---|---|---|---|
| TEST-V01 | Missing Problem/testcase set | Stop; configuration failure | D09 |
| TEST-V02 | Missing testcase ID/visibility | Invalid/restrictive handling | D10/D12 |
| TEST-V03 | Unknown Judge0 status | Never pass; classify unknown | D13/D14 |
| TEST-V04 | Queued token at timeout | Incomplete, not normal grade | D19 |
| TEST-V05 | Token/result count mismatch | Correlation failure | D19/D21 |
| TEST-V06 | Duplicate testcase result | No double scoring | D09/D18 |
| TEST-V07 | Invalid/zero weight | Approved validation | D10 |
| TEST-V08 | Total weight zero | Explicit formula/error policy | D10/D18 |
| TEST-V09 | Output mismatch | Comparator policy | D14/D15 |
| TEST-V10 | Floating-point output | Strict/tolerance policy | D15 |
| TEST-V11 | Multiple failure types | Approved precedence | D18 |
| TEST-V12 | Partial pass set | Approved partial verdict/score | D18 |
| TEST-V13 | Student requests hidden output | Mask/deny | D12/D23 |
| TEST-V14 | Hidden stderr/compiler output | Mask per policy | D12/D23 |
| TEST-V15 | Student requests foreign submission | Deny | RBAC |
| TEST-V16 | Staff requests out-of-scope record | Deny | RBAC |
| TEST-V17 | Unauthorized rerun | Deny | D22 |
| TEST-V18 | Rerun while active | Serialize/reject per policy | D22 |
| TEST-V19 | Problem changes before worker start | Snapshot/version policy | D20 |
| TEST-V20 | DB persistence failure | Recover; no false result/event | D21 |
| TEST-V21 | Duplicate grading event | Idempotent consumer | D24 |
| TEST-V22 | Event before result commit | Prevent | D24 |
| TEST-V23 | Excessive output | Approved truncation/limit | D23 |
| TEST-V24 | Student requests raw JudgeRun | Deny | D23 |
| TEST-V25 | Worker stuck/retry exhausted | Operational recovery | D19/D27 |

## 27. Data and state reference

### Conceptual TestCase
id, problemId, input, expectedOutput, isHidden, orderIndex, createdAt. Schema does not list weight.

### Legacy audit-reported ProblemTestCase
content.problem_test_cases; isSample; weight (reported default 1 and validation 1–100); no explicit orderBy/ordering field reported. Do not merge with conceptual model without decision.

### Legacy audit-reported Submission/JudgeRun
assessment.submissions holds aggregate result/status/score; assessment.judge_runs holds per-testcase raw runs. This differs from conceptual CodeSubmission fields.

### State reference (not canonical)
- Legacy submission: pending → graded; error on worker exception.
- Judge0: in_queue / processing / terminal statuses.
- Evaluation: batch submit → poll → collect → aggregate.
- Conceptual JudgeVerdict enum differs from audit-reported verdict/status strings.
- Rejudge: legacy aggregate reset/overwrite, raw run rows retained; target TBD.

## 28. Decision log status

**EVAL-D01–EVAL-D27 are adopted.** No product/architecture/security decisions remain open in Module 7. Full register: ADOPTED-DECISIONS.md.

| Workstream | Status | Remaining work |
|---|---|---|
| Evaluation policy | Adopted | None |
| ProblemVersion/testcase schema | Adopted through Module 5 | Schema migration/integration |
| Submission/run persistence | Adopted through Module 6 Option B | Schema migration/integration |
| Judge0 comparator/status | Adopted deployment-neutral contract | Pin deployed version/config and contract-test |
| Hidden-data policy | Adopted | Implementation/security test |
| Rejudge/history | Adopted | Implementation/test |
| Grading events | Adopted | Cross-module integration |
| Operations/retention | Policy adopted | Configuration/implementation |
| QA | 200 cases defined | Execute and record evidence |

## 29. Review and readiness checklist

**Product policy — complete**
- [x] EVAL-D01–EVAL-D27 adopted.
- [x] Five evaluation contexts defined.
- [x] ProblemVersion/testcase identity, visibility and ordering aligned to Module 5.
- [x] CodeSubmission/SubmissionRun/JudgeRun aligned to Module 6 Option B.
- [x] Comparator, resource limits, score and verdict policy defined.
- [x] Hidden-data and role-scoped result visibility defined.
- [x] Incomplete execution, rejudge history and grading events defined.
- [x] Retention, output limits, audit and monitoring policies defined.

**Engineering/release — pending**
- [ ] Implement normalized testcase result mapping, aggregation and transaction.
- [ ] Verify schema fields/indexes against Modules 5/6.
- [ ] Pin deployed Judge0 CE version/config and contract-test statuses/comparator behavior.
- [ ] Implement server-side hidden-result masking across REST/WebSocket/events/exports/logs/errors.
- [ ] Implement append-only run history and SUPER_ADMIN rejudge.
- [ ] Integrate Modules 4/8/9/11, 13, 15, 17 and 18.
- [ ] Configure output caps, retention and monitoring thresholds.
- [ ] Execute 200 QA cases plus concurrency/fault/security/load tests.

**Approval boundary:** product decisions are adopted. Schema, runtime, deployment verification, integrations and QA execution remain incomplete.

## 30. Adopted decision register

All EVAL-D01–EVAL-D27 are adopted and recorded in ADOPTED-DECISIONS.md. They resolve contexts, immutable testcase snapshots, equal-weight scoring, verdict precedence, hidden-data confidentiality, role-scoped access, Option B persistence, append-only rejudge history, grading events, retention and monitoring.

## 31. Normative evaluation algorithm

1. Resolve CodeSubmission and its immutable ProblemVersion. Validate its single activity context through the owner module.
2. Resolve the complete ordered ProblemVersionTestCase set. An empty or inconsistent set is a configuration failure.
3. Build one Judge0 execution per testcase using server-controlled source, canonical language key, testcase input/expected output and pinned resource limits.
4. Maintain one JudgeRun per testcase under the active SubmissionRun. Correlate tokens to stable testcase IDs, never array order.
5. Poll all tokens using Module 6 asynchronous policy. All official cases are allowed to finish; one code failure does not short-circuit remaining cases.
6. Only terminal mapped student-code outcomes enter aggregation. Missing, pending, unknown or infrastructure outcomes leave the run incomplete/FAILED with null verdict and score.
7. For complete results, count all official cases equally. scorePercent = Math.round(100 × passedCaseCount / totalOfficialCaseCount), integer 0–100. ACCEPTED requires every official testcase to pass. Otherwise apply TLE > MLE > CE > RE > WA regardless of testcase order. MLE only when deployed mapping verifies it. No PARTIAL verdict exists in v1.
8. Persist normalized JudgeRun outcomes and aggregate SubmissionRun result transactionally. Update parent summary only after commit. Emit SubmissionGraded through the outbox after commit.

## 32. Result privacy, history and access contract

- Visible testcase results may return approved safe details; hidden testcase result entries and metadata are omitted from Student DTOs. Hidden count/order, status, pass/fail, ordinal, timing, memory, input, expected/actual output, stderr and compiler output are not Student-visible.
- Student retrieves own source and safe result only. HOD/Manager receive scoped summary DTOs only. SUPER_ADMIN raw access is privileged and audited. Finance has no Module 7 access.
- REST, WebSocket, exports, events, errors and logs apply the same server-side allowlist/masking.
- CodeSubmission → SubmissionRun → JudgeRun is the only canonical persistence. Historical runs are append-only. Rejudge is SUPER_ADMIN-only, reason-required, uses original ProblemVersion and consumes no Student attempt.
- Failed/incomplete rejudge does not erase the last completed grade. Downstream score/rank/progress effects are handled by activity owners after the canonical event.

## 33. Cross-module and release gates

| Module | Contract |
|---|---|
| 5 — Problem Management | Immutable ProblemVersion, ordered TestCase snapshot, isHidden, resource limits and language keys |
| 6 — Code Submission & Judge0 | CodeSubmission/SubmissionRun/JudgeRun, async tokens/statuses, processing lifecycle, outbox and SUPER_ADMIN rejudge |
| 4 — Course & Enrollment | Validate Course association/enrollment; consume committed SubmissionGraded idempotently for progress |
| 8 — Labs & Lab Scoring | Map problem-level score using Lab-owned maxScore/late/attempt policies |
| 9 — Assessments | Own assessment window, attempt rules, aggregate and result-release policies |
| 10 — Groups | Validate group membership where required |
| 11 — Contests | Own points, penalties, leaderboard and rank updates |
| 13 — Notifications | Resolve recipients/templates/channels for committed graded/regraded events |
| 15 — Audit Logging | Audit privileged raw inspection, rejudge and correction without hidden content |
| 17/18 — State/Edge Cases | Own lifecycle transitions, stuck-job detection, retries, partial batches and recovery |

**Release gate:** pin deployed Judge0 CE version/config; contract-test status mapping, language/config endpoints, batch token mapping and comparator whitespace behavior; then execute all 200 QA cases plus security, concurrency, fault-injection and load tests. These engineering actions remain pending.
