# Module 7 — Test Case Evaluation: Adopted Decisions

**Document ID:** DEZ-EVAL-DECISIONS-007  
**Version:** 1.0  
**Status:** Adopted target product baseline  
**Decision range:** EVAL-D01–EVAL-D27  
**Cross-module basis:** Module 5 ProblemVersion/TestCase policy; Module 6 Option B architecture.

Adopted by explicit user direction to proceed using the established recommended-default approach. Earlier source/legacy audit descriptions remain historical evidence unless repeated as adopted target rules. README Sections 30–33 are normative.

| ID | Topic | Adopted decision |
|---|---|---|
| EVAL-D01 | Evaluation contexts | One engine for PRACTICE, COURSE, LAB, ASSESSMENT and CONTEST; owner modules validate association, membership, windows and attempts. |
| EVAL-D02 | Partial credit by context | Problem-level percentage is consistent; activity owners map it to max score, pass threshold, penalty or rank. |
| EVAL-D03 | Verdict precedence | Completed outcomes use source-intended TLE > MLE > CE > RE > WA regardless of array order. MLE only if deployment verifies it. |
| EVAL-D04 | PARTIAL verdict | No PARTIAL enum in v1; numeric score carries partial credit. |
| EVAL-D05 | Hidden feedback | Hidden cases count toward grading but no per-case outcome/status/count/order/ordinal/runtime/memory/data is Student-visible. |
| EVAL-D06 | Staff raw access | HOD/Manager summary-only; SUPER_ADMIN raw access audited; Student own safe view; Finance denied. |
| EVAL-D07 | Rejudge authority | SUPER_ADMIN only; reason/audit required; new SubmissionRun under same parent; no new Student attempt. |
| EVAL-D08 | Historical grading history | Append-only SubmissionRun/JudgeRun history; no overwrite; failed rerun does not erase last completed grade. |
| EVAL-D09 | Testcase set and empty set | Use complete ordered snapshot from pinned ProblemVersion; missing/empty set is configuration failure. |
| EVAL-D10 | Schema, visibility and weight | Module 5 TestCase stable ID, ProblemVersion, input, expectedOutput, isHidden and orderIndex. No weight/isSample in v1; equal weighting. |
| EVAL-D11 | Identity/order/index | Stable testcase ID canonical; orderIndex zero-based unique contiguous; visible ordinal one-based; failedTestCaseIndex derived only. |
| EVAL-D12 | Hidden masking | Mask hidden fields/status/count/order/pass/fail/timing/diagnostics across REST, WebSocket, exports, logs and errors. Unknown visibility hidden. |
| EVAL-D13 | Judge0 status mapping | Use deployed /statuses and pinned CE version/config. CE v1.13.1 IDs 7–12 Runtime Error variants, 13 Internal Error, 14 Exec Format Error; no dedicated MLE. |
| EVAL-D14 | Infrastructure failures | Internal/unknown status, partial/missing/duplicate token, timeout, worker error or incomplete set is FAILED/recoverable with null verdict/score. |
| EVAL-D15 | Comparator | Judge0 built-in expected_output comparator only. No app normalization, float tolerance or custom checker. Golden-test whitespace; empty strings valid. |
| EVAL-D16 | Custom checker | No custom comparator/checker/interactive checker in v1. |
| EVAL-D17 | Resource limits | Per-testcase limits from ProblemVersion; CPU ms to seconds, MiB to KiB. Assessment duration is not CPU time. |
| EVAL-D18 | Score | Complete runs: integer Math.round(100 × passed/total), all cases equal. Incomplete/operational failure has null score. |
| EVAL-D19 | Recovery/cancellation | Modules 6/17/18: durable async; 3 pre-handoff retries; 120s default deadline; jittered 1/2/4/5s polling; poll existing tokens; privileged pre-handoff cancellation only. |
| EVAL-D20 | Version snapshot | Submission and every run bind immutable ProblemVersion at acceptance; rejudge uses original; current edits do not affect old runs. |
| EVAL-D21 | Persistence | Module 6 Option B CodeSubmission → SubmissionRun → JudgeRun; one JudgeRun/token per testcase; no legacy Assessment-only tables. |
| EVAL-D22 | Grading event | Emit SubmissionGraded after aggregate commit; safe IDs/context/version/run/summary; dedupe eventId/runId. |
| EVAL-D23 | Retention | Academic history per policy; source/raw separately retained/encrypted; no purge without configured policy. |
| EVAL-D24 | Raw payload | Do not persist full Judge0 HTTP body by default; normalized allowlisted fields/protected outputs only; raw access SUPER_ADMIN-only/audited. |
| EVAL-D25 | Output limits | Student diagnostics 16 KiB/field; protected raw output 64 KiB default configurable; mark truncation; mask hidden diagnostics first. |
| EVAL-D26 | Audit | Module 15 audits raw reads/rejudge/corrections; Module 5 audits testcase/version edits; no hidden data in metadata. |
| EVAL-D27 | Monitoring | Monitor queue/run age, pending tokens, partial batch, retry exhaustion, Judge0 errors/latency, worker heartbeat and unknown statuses; telemetry excludes source/token/hidden data. |

## Source reconciliation
- Business Flow v2 establishes Backend → Judge0 → Evaluation → Result → Score and shows Course, Lab and Contest flows; Page–DB Mapping also identifies Practice and Assessment surfaces.
- Module 5 owns immutable ProblemVersion TestCase snapshots, isHidden/orderIndex and no weight field.
- Legacy audit reported isSample/weight, Assessment-only persistence, intended TLE > MLE > CE > RE > WA priority, first-failure defect, hidden diagnostic gaps and overwrite-style rejudge. These are historical findings, not target design.
- Module 6 Option B owns CodeSubmission/SubmissionRun/JudgeRun, async tokens, processing statuses, SUPER_ADMIN-only rejudge and post-commit SubmissionGraded.
- Official Judge0 CE v1.13.1 published statuses include 7–12 Runtime Error variants and 13 Internal Error; no dedicated MLE. Deployed /statuses and /config_info remain runtime authority.

## Engineering gates
1. Implement normalized testcase results, deterministic score/verdict aggregation and transaction using Module 6 persistence.
2. Verify ProblemVersion snapshots and testcase identity/order against Module 5 migration.
3. Pin deployed Judge0 CE version/config and contract-test statuses and comparator behavior.
4. Implement hidden-result masking and role DTO allowlists across REST/WebSocket/logs/events/exports.
5. Implement append-only run history and privileged rejudge.
6. Integrate activity consumers/events with Modules 4/8/9/11/13/15.
7. Integrate Module 17 lifecycle and Module 18 recovery/monitoring.
8. Configure retention/output caps and execute 200 QA cases plus security/load/fault-injection.

**Approval boundary:** all 27 product decisions adopted. Schema/runtime implementation, deployed Judge0 verification, module integration and QA execution remain incomplete.