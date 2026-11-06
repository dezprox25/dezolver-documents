# Module 6 — Code Submission & Judge0 Integration

## 1. Module Executive Summary

Module 6 provides the core asynchronous code submission, compilation, sandboxed execution, and real-time execution feedback pipeline of the Dezolver platform. It connects student solution code written in Monaco Editor to high-performance, containerized sandbox compilers powered by Judge0 CE.

When a student submits code against an assessment coding problem, the backend validates language compatibility and attempt quotas, persists a pending submission record, and immediately emits an HTTP 202 Accepted response. Execution is offloaded to a distributed BullMQ background queue (`code-execution`), where worker daemons batch-dispatch test cases to the Judge0 CE runner, poll with exponential backoff until terminal execution states are reached, persist granular `JudgeRun` records, calculate deterministic scores and verdicts, and broadcast live verdict notifications to the user interface via Redis Pub/Sub and Socket.IO.

---

## 2. Directory Layout & Document Navigation

```
docs/functional-spec/module-06-submission/
├── README.md               # Executive summary, forensic findings, and navigational guide (this file)
└── SPECIFICATION.md        # Complete 17-Section (A–Q) legacy functional reconstruction
```

- **Detailed Functional Specification:** [`SPECIFICATION.md`](./SPECIFICATION.md)
- **Authoring Standard:** [`FUNCTIONAL-SPECIFICATION-STANDARD.md`](../../FUNCTIONAL-SPECIFICATION-STANDARD.md)
- **Master Index & Registry:** [`INDEX.md`](../INDEX.md)

---

## 3. Key Forensic Discoveries & Architecture Summary

| Aspect | Legacy Source Code Implementation Reality |
|---|---|
| **Database Schema** | `assessment` schema ([`prisma/assessment/schema.prisma`](file:///E:/Dezolver%20project/dezlovers-main/Final%20product/dezlovers-main(cuurent%20update)/prisma/assessment/schema.prisma)): `Submission` (`assessment.submissions`) and `JudgeRun` (`assessment.judge_runs`). |
| **Ingress Pattern** | Asynchronous HTTP 202 Accepted. `POST /assessments/:id/submissions` persists submission in `pending` status, captures attempt number, and enqueues job to BullMQ queue `code-execution`. |
| **Max Attempts Gate** | Enforces `maxAttempts` from assessment definition: `countByUserAndAssessment >= maxAttempts -> throw ForbiddenException({ code: 'max_attempts_reached' })`. |
| **Judge0 Sandbox Architecture** | Self-hosted Judge0 CE client ([`src/assessment/judge/judge0.client.ts`](file:///E:/Dezolver%20project/dezlovers-main/Final%20product/dezlovers-main(cuurent%20update)/src/assessment/judge/judge0.client.ts)) targeting `https://judge.truenooklms.org` (with alternative `Judge0RapidApiClient`). |
| **Language Mapping** | `LANGUAGE_IDS` mapping 41 compiler strings to Judge0 CE runtime IDs (e.g. `python`: 71, `cpp`: 54, `java`: 62, `javascript`: 63, `c`: 50, `rust`: 73). |
| **Execution Protocol** | Batch submission via `POST /submissions/batch?base64_encoded=true`. Input, source code, and expected output are base64-encoded. |
| **Polling Mechanism** | Exponential backoff polling in `pollUntilDone`: starts at 1000 ms, steps +500 ms, caps at 4000 ms, with 60,000 ms hard deadline. |
| **Per-Case Persistence** | Persists exactly one `JudgeRun` record per test case executed, capturing stdout, stderr, compileOut, timeMs, memoryKb, exitCode, and raw JSON response. |
| **Verdict Precedence** | Priority resolution in `computeVerdict`: $\text{TLE} (5) > \text{MLE} (12) > \text{CE} (6) > \text{RE} (7\text{--}11) > \text{WA} (4)$. |
| **Weighted Scoring** | $\text{Score} = \text{round}\left(\frac{\sum_{\text{Passed}} \text{weight}_i}{\sum_{\text{Total}} \text{weight}_i} \times 100\right)$. If $0 < \text{passed} < \text{total}$ and verdict is accepted, overall verdict becomes `'partial'`. |
| **Real-Time Distribution** | Worker publishes to Redis channels `dezolver#submission:${id}` and `dezolver#user:${userId}`. Socket.IO gateway (`SubmissionsGateway`) relays live events to clients. |
| **Rerun Capability** | Privileged endpoint `POST /submissions/:id/rerun` resets status to `pending` and re-enqueues evaluation. |
| **Broken Anti-Cheat Cron** | `AntiCheatService.analyzeRecentSubmissions` crashes silently on nonexistent columns `createdAt`, `flagged`, and `metadata`, catching errors and returning `[]`. |
| **Broken Event Dispatch** | `CodeExecutionConsumer` emits outbox event `'SubmissionGraded'`. However, `SubmissionAcceptedConsumer` expects `'SubmissionAccepted'`, leaving room auto-completion disconnected. |

---

## 4. Primary Source Code References

### Backend
- **Prisma Schema:** [`prisma/assessment/schema.prisma`](file:///E:/Dezolver%20project/dezlovers-main/Final%20product/dezlovers-main(cuurent%20update)/prisma/assessment/schema.prisma) (`Submission`, `JudgeRun`, `Assessment`)
- **Controller:** [`src/assessment/controllers/submissions.controller.ts`](file:///E:/Dezolver%20project/dezlovers-main/Final%20product/dezlovers-main(cuurent%20update)/src/assessment/controllers/submissions.controller.ts)
- **Service:** [`src/assessment/services/submission.service.ts`](file:///E:/Dezolver%20project/dezlovers-main/Final%20product/dezlovers-main(cuurent%20update)/src/assessment/services/submission.service.ts)
- **Repository:** [`src/assessment/repositories/submissions.repository.ts`](file:///E:/Dezolver%20project/dezlovers-main/Final%20product/dezlovers-main(cuurent%20update)/src/assessment/repositories/submissions.repository.ts)
- **Judge0 Clients:**
  - Self-hosted CE: [`src/assessment/judge/judge0.client.ts`](file:///E:/Dezolver%20project/dezlovers-main/Final%20product/dezlovers-main(cuurent%20update)/src/assessment/judge/judge0.client.ts)
  - RapidAPI CE: [`src/assessment/judge/judge0-rapidapi.client.ts`](file:///E:/Dezolver%20project/dezlovers-main/Final%20product/dezlovers-main(cuurent%20update)/src/assessment/judge/judge0-rapidapi.client.ts)
- **Background Worker:** [`src/worker/consumers/code-execution.consumer.ts`](file:///E:/Dezolver%20project/dezlovers-main/Final%20product/dezlovers-main(cuurent%20update)/src/worker/consumers/code-execution.consumer.ts)
- **WebSocket Gateway:** [`src/websocket/gateways/submissions.gateway.ts`](file:///E:/Dezolver%20project/dezlovers-main/Final%20product/dezlovers-main(cuurent%20update)/src/websocket/gateways/submissions.gateway.ts)
- **Anti-Cheat Service:** [`src/assessment/services/anti-cheat.service.ts`](file:///E:/Dezolver%20project/dezlovers-main/Final%20product/dezlovers-main(cuurent%20update)/src/assessment/services/anti-cheat.service.ts)

### Frontend
- **Submission Pages:** [`frontend/src/pages/submissions/`](file:///E:/Dezolver%20project/dezlovers-main/Final%20product/frontend/src/pages/submissions/)
  - [`SubmissionsPage.tsx`](file:///E:/Dezolver%20project/dezlovers-main/Final%20product/frontend/src/pages/submissions/SubmissionsPage.tsx)
  - [`SubmissionDetailPage.tsx`](file:///E:/Dezolver%20project/dezlovers-main/Final%20product/frontend/src/pages/submissions/SubmissionDetailPage.tsx)
  - [`FlaggedSubmissionsPage.tsx`](file:///E:/Dezolver%20project/dezlovers-main/Final%20product/frontend/src/pages/submissions/FlaggedSubmissionsPage.tsx)
- **Assessment Components:**
  - [`CodeEditor.tsx`](file:///E:/Dezolver%20project/dezlovers-main/Final%20product/frontend/src/components/assessment/CodeEditor.tsx)
  - [`SubmissionTable.tsx`](file:///E:/Dezolver%20project/dezlovers-main/Final%20product/frontend/src/components/assessment/SubmissionTable.tsx)
  - [`SubmissionCard.tsx`](file:///E:/Dezolver%20project/dezlovers-main/Final%20product/frontend/src/components/assessment/SubmissionCard.tsx)
  - [`JudgeRunCard.tsx`](file:///E:/Dezolver%20project/dezlovers-main/Final%20product/frontend/src/components/assessment/JudgeRunCard.tsx)
- **Hooks & Services:**
  - [`useSubmissions.ts`](file:///E:/Dezolver%20project/dezlovers-main/Final%20product/frontend/src/hooks/useSubmissions.ts)
  - [`submission.service.ts`](file:///E:/Dezolver%20project/dezlovers-main/Final%20product/frontend/src/services/api/submission.service.ts)
  - [`judge.service.ts`](file:///E:/Dezolver%20project/dezlovers-main/Final%20product/frontend/src/services/api/judge.service.ts)

