# Module 6 — Code Submission & Judge0: Adopted Decisions

**Document ID:** DEZ-SUB-DECISIONS-006  
**Version:** 1.0  
**Status:** Adopted target product baseline  
**Architecture:** Option B — CodeSubmission + SubmissionRun + JudgeRun  
**Decision range:** SUB-D01–SUB-D28

Adopted by explicit user direction on 2026-11-07. Option B selected and recommended defaults applied to all remaining decisions. README Sections 25–29 are normative.

| ID | Topic | Adopted decision |
|---|---|---|
| SUB-D01 | Judge0 deployment | Deployment-neutral adapter; private/self-hosted CE-compatible Judge0 preferred; separate environments and secret-managed credentials. |
| SUB-D02 | API/auth/version | Official CE async submit/batch/token retrieval; base64 text; pin deployed version/config; server-side auth; individual POST fallback. |
| SUB-D03 | Delivery | Durable async outbox/worker, server-side polling, websocket + REST polling; no formal sync wait. |
| SUB-D04 | Language IDs | Stable DEZOLVER language keys mapped to active Judge0 ID/version; pin per run; no hard-coded legacy IDs. |
| SUB-D05 | Creation/state | Atomic parent + run 1 + attempt + outbox; HTTP 202 after commit; QUEUED initially, verdict null. |
| SUB-D06 | Persistence | Option B: CodeSubmission parent, SubmissionRun per generation, JudgeRun per testcase/token. |
| SUB-D07 | Activity | Exactly one PRACTICE/COURSE/LAB/ASSESSMENT/CONTEST; Practice contextId null; owner modules validate. |
| SUB-D08 | Code validation | UTF-8 non-empty code; 100 KiB default configurable maximum; no custom stdin/compiler args v1. |
| SUB-D09 | Attempts | Practice/Course no product cap; Lab/Assessment/Contest owner modules enforce limits; rate guard remains. |
| SUB-D10 | Idempotency | Required 16–128 printable ASCII key; Student/context scoped; same payload replay 24h; changed payload 409. |
| SUB-D11 | Retries | Up to 3 pre-handoff retries with exponential backoff+jitter; after tokens persist, poll/reconcile only. |
| SUB-D12 | Timeout/poll | 120-second default deadline configurable; 1/2/4/5-second jittered polls. |
| SUB-D13 | State mapping | ProcessingStatus distinct from JudgeVerdict; deployed /statuses mapping; IDs 1/2 nonterminal. |
| SUB-D14 | Infrastructure | Internal/unknown status, missing/duplicate tokens, partial batch, timeouts and worker failure are operational, never Wrong Answer. |
| SUB-D15 | Metrics | UTC; CPU ms→seconds; memoryLimitMb as MiB→KB ×1024; max per-case metrics; failed index one-based display. |
| SUB-D16 | Output privacy | Visible feedback only; hidden details/count/order/outputs masked; 16 KiB default Student diagnostic cap. |
| SUB-D17 | Permissions | Student own; HOD Department summary; Manager Section summary; SUPER_ADMIN raw audited; Finance denied; staff source hidden v1. |
| SUB-D18 | Cancellation | Student cancel disabled; SUPER_ADMIN/Ops can cancel pre-handoff only. |
| SUB-D19 | Retention | Academic history per institution policy; source/raw separate retention/encryption; no auto-purge without policy. |
| SUB-D20 | Rejudge | SUPER_ADMIN+reason; new run on same ProblemVersion; preserve old runs; no new attempt; score reconciliation explicit. |
| SUB-D21 | Scratchpad | Run Code/custom stdin/scratchpad deferred from v1. |
| SUB-D22 | Rate/fairness | 10 accepted/minute and 2 in-flight/Student defaults; tenant quotas configurable; weighted fair scheduling. |
| SUB-D23 | Security | TLS, encryption at rest, ID-only queue, no code/token/hidden data in logs/events; Judge0 credentials server-only. |
| SUB-D24 | Events | Durable accepted/started/graded/failed/rejudge events; dedupe eventId/runId; payload excludes code/token/hidden data. |
| SUB-D25 | Push/poll | DB authoritative; websocket hint + 3-second polling fallback backed off to 10 seconds; Module 13 owns notices. |
| SUB-D26 | API | POST /api/v1/submissions + Idempotency-Key; 202 after commit; safe GET detail/history; SUPER_ADMIN rejudge. |
| SUB-D27 | Activity owners | Course Module 4; Lab Module 8 (+Group Module 10); Assessment Module 9; Contest Module 11; Practice Module 5. |
| SUB-D28 | Attempt atomicity | Atomic per Student+Problem+context; owner caps; replay no extra attempt; rejudge is not an attempt. |

## Source reconciliation
- Business Flow original/v2 shows Course/Problem, Lab/Problem and Contest/Problem Submit flows through DEZOLVER/Judge0.
- Page–DB Mapping associates CodeSubmission with HOD/Manager pages and Student Assignments, My Labs, Practice Problems, Contests and My Courses; it is not a full API permission matrix.
- RBAC grants SUPER_ADMIN View/Manage, HOD View, Manager View*, Student Create/View-own; Finance has no Code Submission grant.
- Conceptual schema has one parent judge0Token and no context/per-testcase token model.
- Legacy audit reports Assessment-bound submissions and per-testcase judge runs; it is not canonical target persistence.
- Module 5 owns immutable ProblemVersion/language/runtime; Module 7 owns testcase evaluation.
- Official Judge0 CE docs describe async tokenized submit, batch create/retrieve and language/status/config endpoints; documented v1.13.1. Deployed version/config must be verified.

## Architecture
One CodeSubmission represents one Student formal submission with one immutable ProblemVersion and one context. Each execution is a SubmissionRun; each testcase execution is a JudgeRun with its own token. Rejudge creates a new run and preserves prior history without incrementing Student attempt.

## Engineering gates
1. Add parent context/version/attempt/processing/idempotency fields and SubmissionRun/JudgeRun/outbox/idempotency structures.
2. Backfill old single-token rows only when testcase correlation is proven.
3. Implement atomic attempts, outbox and idempotency.
4. Pin Judge0 adapter/version/status/language/config.
5. Integrate Module 7 immutable testcase snapshots and result aggregation.
6. Integrate activity authorization Modules 4/8/9/10/11.
7. Integrate Modules 17/18/13/15.
8. Execute 180 QA cases plus load/security/fault-injection testing.

**Approval boundary:** all 28 product decisions are adopted. Schema, runtime, deployed Judge0 verification, integrations and QA execution remain incomplete pending evidence.