# DEZOLVER — 11.2 Contest Submission

**Document ID:** CON-SUB-FS-001 | **Version:** 0.2 | **Status:** Detailed draft; decisions open  
**Governing standard:** [Functional Specification Standard v1.0](../../FUNCTIONAL-SPECIFICATION-STANDARD.md)  
**Parent module:** [Module 11 Contests](../README.md)  
**Core contest management:** [Function-Level Specifications](../FUNCTION-SPECIFICATIONS.md)

> Source facts, derived behavior, recommendations, and TBD decisions remain separate. This draft does not approve proposed behavior.

## Evidence and boundaries
Business Flow shows Contest Problem → Student participation → Code Submission → Judge0 → evaluation. Page mapping includes Contest, ContestProblem, ContestParticipant, Problem, CodeSubmission. CodeSubmission conceptual fields: problemId, studentId, language, code, judge0Token, verdict, executionTimeMs, memoryUsageKb, failedTestCaseIndex, stdout, stderr, submittedAt, startedAt, completedAt. Schema lacks contestId/contestProblemId/participantId. Module 6 owns submission/Judge0; Module 7 owns evaluation.

## Permissions
Student submits as self and only when eligible. Staff access follows RBAC. Group membership does not authorize another Student’s code. Hidden testcase policy Modules 5–7.

### CSUB-F001 — View Contest Problem
- **Entry:** Contest detail/problem selector.
- **Inputs:** contestId, contestProblemId/problemId; Student auth context.
- **Flow:** validate Contest/Problem association, participation/eligibility, time and Problem visibility → retrieve Module 5 allowed statement → display.
- **Rules:** Problem pre-start visibility, versioning/freeze, post-contest access CON-D06/11/24.
- **Validation:** foreign Contest/Problem, missing association, nonparticipant, hidden/archived Problem, time window.
- **Data:** read only.
- **Success:** permitted statement; no hidden testcase.
- **Acceptance:** linked Problem and authorization required; viewing does not permit submission.
- **QA:** CON-QA-059–063.

### CSUB-F002 — Submit code in Contest context
- **Status:** submission flow is source-confirmed; contest-specific linkage/storage policy TBD.
- **Entry:** Contest Problem workspace → Submit.
- **Inputs:** contestId, contestProblemId/problemId, language, code; Student ID auth-derived. Module 6 owns submission contract.
- **Preconditions:** active eligible participant, linked Problem, allowed time/language, submission policy.
- **Flow:** submit → authorize Contest/participant/problem → check window/limits → validate using Module 6 → persist submission and Contest context per approved design → dispatch evaluation → return submission status.
- **Rules:** linkage CON-D12; limits/rate/attempt/late rules CON-D13; Judge0 CPU vs Contest duration CON-D14. Do not invent fields.
- **Validation:** nonparticipant, foreign problem, outside window, unsupported language, rate limit, duplicate request, Judge0 outage.
- **Data:** CodeSubmission via Module 6 and approved context link; no schema assumption.
- **Success:** accepted/queued; evaluation remains pending.
- **Failure:** judge operational failure is not automatically wrong answer or zero; Module 18.
- **Acceptance:** server validates contest context; no cross-contest contamination; pending and judge error distinct.
- **QA:** CON-QA-064–071.

### CSUB-F003 — View submission history/result
- **Entry:** Contest Problem → submission history.
- **Inputs:** contestId/submissionId; identity auth-derived.
- **Flow:** authorize Contest, submission, ownership → retrieve Module 6 record and Module 7 verdict → return allowed details.
- **Rules:** Student own-only; staff scope; result release and stdout/stderr/failed testcase visibility CON-D24.
- **Validation:** foreign submission/Student/Contest, pending evaluation.
- **Data:** read.
- **Failure/edge:** Judge pending/error, contest ended, rejudge/score update.
- **Acceptance:** own record restriction; verdict visibility follows Modules 6/7; pending is not final.
- **QA:** CON-QA-072–076.

### Historical cross-check — legacy Event evidence

**Evidence status: LEGACY-SPEC-REPORTED.** The old-code report says POST /events/:id/submissions was a stub and actual event submissions used POST /assessments/:id/submissions with metadata.eventId. This is historical Event behavior, not the target contract. Module 6 adopted one activity context per CodeSubmission: contextType=CONTEST, contextId=Contest.id, persisted as CodeSubmission → SubmissionRun → JudgeRun. Module 11 validates ContestParticipant, ContestProblem and schedule eligibility; Module 6 owns intake/Judge0; Module 7 owns evaluation. Do not retain the Assessment proxy. Raw old source was not independently inspected.

**Decision IDs:** CON-D05/06/11–14/24.
