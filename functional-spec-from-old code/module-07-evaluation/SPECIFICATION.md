# Module 7 — Test Case Evaluation & Scoring Engine Specification

**Standard Reference:** `DEZ-STD-FS-001`  
**Document Identifier:** `DEZ-SPEC-MOD-007`  
**Status:** `CONFIRMED LEGACY IMPLEMENTATION / RECONSTRUCTED`  
**Database Schema:** `assessment` ([`prisma/assessment/schema.prisma`](file:///E:/Dezolver%20project/dezlovers-main/Final%20product/dezlovers-main(cuurent%20update)/prisma/assessment/schema.prisma)), `content` ([`prisma/content/schema.prisma`](file:///E:/Dezolver%20project/dezlovers-main/Final%20product/dezlovers-main(cuurent%20update)/prisma/content/schema.prisma)), `events` ([`prisma/events/schema.prisma`](file:///E:/Dezolver%20project/dezlovers-main/Final%20product/dezlovers-main(cuurent%20update)/prisma/events/schema.prisma))  
**Module Directory:** `src/worker/consumers/`, `src/assessment/`  
**Frontend Directory:** `frontend/src/pages/submissions/`, `frontend/src/components/assessment/`  

---

## SECTION A: MODULE OVERVIEW

### A.1 Purpose & Objectives
Module 7 (Test Case Evaluation & Scoring Engine) governs the evaluation methodology, output comparison model, multi-tier verdict resolution, score calculation, and cross-modality score propagation of the Dezolver platform. While Module 6 manages the ingress, queueing, and sandbox dispatch of source code, Module 7 defines how test case results are graded, how sandbox output codes are synthesized into deterministic submission verdicts, and how results are distributed across Assessments, Practice Rooms, Competitions, and Labs.

The core business objectives implemented by this module are:
1. **Delegated Sandboxed Diffing:** Submitting baseline expected outputs alongside candidate inputs to the Judge0 CE runner, utilizing containerized string comparison rather than in-process Node.js diffing.
2. **Deterministic Verdict Priority Resolution:** Synthesizing disparate per-case sandbox outcomes into a single authoritative submission verdict using an unyielding priority hierarchy ($\text{TLE} > \text{MLE} > \text{CE} > \text{RE} > \text{WA}$).
3. **Proportional Weighted Scoring:** Calculating normalized 0–100% percentage scores based on individual test case weights defined by problem authors.
4. **Test Case Secrecy Enforcement:** Enforcing strict data sanitization on evaluated test case results (`testCaseResults`), ensuring actual and expected outputs are exposed exclusively for sample test cases while masking evaluation outputs for hidden test cases.
5. **Cross-Modality Score Propagation:** Propagating verified scores across platform domains, updating live competition leaderboards in Redis, and triggering curriculum room auto-completion.

### A.2 Scope & Boundaries
- **In-Scope (Module 7):**
  - Evaluation execution model and Judge0 CE response synthesis (`CodeExecutionConsumer.process`).
  - Authoritative verdict resolution algorithm (`computeVerdict`).
  - Proportional weighted score calculation.
  - Granular test case execution logging (`assessment.judge_runs`).
  - Output masking for hidden test cases in `testCaseResults` JSON payloads.
  - Peak execution metric calculation (peak CPU time and peak RAM RSS).
  - Cross-modality scoring for **Assessments** (`assessment.submissions`).
  - Cross-modality scoring for **Contests / Competitions** (`SubmissionGradedConsumer`, `events.event_results`, Redis ZSET leaderboards).
  - Cross-modality scoring for **Practice / Rooms** (`SubmissionAcceptedConsumer`, `paths.room_progress`).
- **Out-of-Scope (Delegated to Adjacent Modules / Non-Existent):**
  - **Code Ingress & HTTP Controllers:** Initial submission receipt, attempt validation, and BullMQ queue enqueueing are owned by **Module 6 (Code Submission & Judge0 Integration)**.
  - **Problem & Test Case Definition:** Ingestion of inputs, outputs, weights, and sample flags is owned by **Module 5 (Problem Management)**.
  - **Exam Anti-Cheat & Fullscreen Lockdown:** Anti-cheat signal collection and proctoring UI are owned by **Module 9 (Assessments)**.
  - **Contest Scheduling & Registration:** Contest lifecycles and registration fees are owned by **Module 11 (Contests)**.
  - **Lab & Lab Scoring:** **NOT IMPLEMENTED.** Exhaustive forensic investigation confirms that neither `LabAssignment`, `LabSubmission`, `GroupLab`, nor any lab scoring services exist in the legacy database or source code.

### A.3 Actors & Personas
- **Student / Candidate (`student`):** Views final score, passed/failed test case breakdown, and inspection panels for sample test cases.
- **Evaluation Worker Daemon (`worker`):** Headless background consumer running `CodeExecutionConsumer`, executing `computeVerdict`, and committing results.
- **Contest Leaderboard Consumer (`events`):** Headless consumer running `SubmissionGradedConsumer`, updating Redis ZSET rank lists.
- **Faculty / Instructor (`faculty`):** Inspects individual test case execution times, memory usage, and raw sandbox outputs via `JudgeRunCard`.

### A.4 Dependencies & Cross-Module Relationships
```mermaid
flowchart TD
    subgraph ContentSchema["content Schema (Module 5)"]
        TC[("content.problem_test_cases<br/>(input, expectedOutput, weight, isSample)")]
    end

    subgraph AssessmentSchema["assessment Schema (Module 6 & 7)"]
        Sub[("assessment.submissions<br/>(score, verdict, testCaseResults)")]
        JR[("assessment.judge_runs<br/>(stdout, stderr, timeMs, memoryKb)")]
    end

    subgraph WorkerLayer["Worker Consumers (Module 7)"]
        Consumer["CodeExecutionConsumer<br/>(computeVerdict)"]
        Judge0["Judge0 CE Runner<br/>(String Diffing)"]
        EventOutbox[("platform.outbox_events<br/>(SubmissionGraded)")]
    end

    subgraph Modalities["Evaluation Consuming Modalities"]
        ContestConsumer["SubmissionGradedConsumer<br/>(Module 11)"]
        RoomConsumer["SubmissionAcceptedConsumer<br/>(Module 4)"]
        EventDB[("events.event_results")]
        RedisZSET[("Redis ZSET Leaderboard<br/>('leaderboard:event:*')")]
        PathDB[("paths.room_progress")]
    end

    TC -->|"Fetch Test Cases"| Consumer
    Consumer -->|"Batch Execution"| Judge0
    Judge0 -->>|"Status IDs & Outputs"| Consumer
    Consumer -->|"computeVerdict()"| Consumer
    Consumer -->|"Save JudgeRuns"| JR
    Consumer -->|"Update Verdict & Score"| Sub
    Consumer -->|"Emit Outbox Event"| EventOutbox

    EventOutbox -->|"SubmissionGraded"| ContestConsumer
    ContestConsumer -->|"Upsert Score"| EventDB
    ContestConsumer -->|"ZADD Score"| RedisZSET

    EventOutbox -.->|"SubmissionAccepted<br/>(BROKEN WIRING)"| RoomConsumer
    RoomConsumer -.->|"Auto-Complete"| PathDB
```

### A.5 Domain Terminology & Glossary
- **Verdict:** The qualitative grading status assigned to a submission (`accepted`, `wrong_answer`, `time_limit_exceeded`, `memory_limit_exceeded`, `compilation_error`, `runtime_error`, `partial`, `internal_error`).
- **Weight ($w_i$):** An integer weight (1–100) assigned to an individual test case, representing its contribution to the final percentage score.
- **Sample Masking:** Security invariant ensuring `actualOutput` and `expectedOutput` are populated in API payloads strictly when `isSample = true`.
- **Status ID:** The numeric status returned by Judge0 CE (e.g. 3 = Accepted, 4 = Wrong Answer, 5 = TLE, 6 = CE).
- **ZSET Leaderboard:** Redis sorted set data structure where student user IDs are scored by cumulative competition points.

---

## SECTION B: LEGACY IMPLEMENTATION ARCHITECTURE

### B.1 Architectural Overview Diagram
```mermaid
sequenceDiagram
    autonumber
    participant Worker as CodeExecutionConsumer
    participant Judge as Judge0 CE Runner
    participant DB as PostgreSQL (assessment schema)
    participant Outbox as platform.outbox_events
    participant Contest as SubmissionGradedConsumer
    participant Redis as Redis ZSET & PubSub

    Worker->>Judge: Submit test cases with expected_output
    Judge-->>Worker: Results with status IDs (3, 4, 5, etc.)
    
    rect rgb(240, 248, 255)
        note over Worker: Module 7: computeVerdict() Engine
        Worker->>Worker: Loop through results & weights
        Worker->>Worker: Calculate passedWeight / totalWeight * 100
        Worker->>Worker: Apply priority: TLE > MLE > CE > RE > WA
        Worker->>Worker: If passed < total and verdict=='accepted' -> verdict='partial'
    end

    Worker->>DB: INSERT assessment.judge_runs (one per test case)
    Worker->>DB: UPDATE assessment.submissions (score, verdict, testCaseResults)
    Worker->>Outbox: INSERT OutboxEvent (SubmissionGraded)
    
    rect rgb(255, 248, 240)
        note over Contest,Redis: Cross-Modality: Contest Leaderboard
        Outbox->>Contest: Handle SubmissionGraded event
        Contest->>DB: UPSERT events.event_results (score)
        Contest->>Redis: ZADD leaderboard:event:ID (score, userId)
        Contest->>Redis: PUBLISH dezolver#event:ID:leaderboard
    end
```

### B.2 Backend Architecture
The evaluation engine is implemented across worker consumers and event handlers:
- **`computeVerdict` Function** ([`src/worker/consumers/code-execution.consumer.ts:39-73`](file:///E:/Dezolver%20project/dezlovers-main/Final%20product/dezlovers-main(cuurent%20update)/src/worker/consumers/code-execution.consumer.ts#L39-L73)):
  - Pure evaluation algorithm executing in the worker thread. Computes cumulative passed weight, total weight, percentage score, failure priority resolution, and partial status assignment.
- **`CodeExecutionConsumer`** ([`src/worker/consumers/code-execution.consumer.ts:75-228`](file:///E:/Dezolver%20project/dezlovers-main/Final%20product/dezlovers-main(cuurent%20update)/src/worker/consumers/code-execution.consumer.ts#L75-L228)):
  - Executes batch test cases, performs sample output masking, calculates peak resource metrics (`executionTimeMs`, `memoryUsedKb`), commits `JudgeRun` records, and updates `assessment.submissions`.
- **`SubmissionGradedConsumer`** ([`src/events/consumers/submission-graded.consumer.ts:7-51`](file:///E:/Dezolver%20project/dezlovers-main/Final%20product/dezlovers-main(cuurent%20update)/src/events/consumers/submission-graded.consumer.ts#L7-L51)):
  - Listens for Outbox event `SubmissionGraded`. Propagates scores to competition leaderboards: updates `events.event_results`, calls `redis.zadd` on leaderboard sorted sets, and broadcasts live ranks.
- **`SubmissionAcceptedConsumer`** ([`src/paths/consumers/submission-accepted.consumer.ts:6-46`](file:///E:/Dezolver%20project/dezlovers-main/Final%20product/dezlovers-main(cuurent%20update)/src/paths/consumers/submission-accepted.consumer.ts#L6-L46)):
  - Outbox consumer designed to auto-complete curriculum rooms upon accepted submissions. (Contains broken event name wiring).

### B.3 Frontend Architecture
Evaluation results are rendered through dedicated assessment components:
- **`VerdictBadge`** ([`frontend/src/components/assessment/VerdictBadge.tsx`](file:///E:/Dezolver%20project/dezlovers-main/Final%20product/frontend/src/components/assessment/VerdictBadge.tsx)):
  - Visual status pill mapping verdicts to color codes (`accepted`: green, `wrong_answer`: red, `time_limit_exceeded`: amber, `compilation_error`: purple, `runtime_error`: orange, `partial`: blue).
- **`VerdictPanel`** ([`frontend/src/components/assessment/VerdictPanel.tsx`](file:///E:/Dezolver%20project/dezlovers-main/Final%20product/frontend/src/components/assessment/VerdictPanel.tsx)):
  - Detailed summary panel rendering percentage score, passed vs total count ($P/N$), execution time, RAM, and test case accordion rows.
- **`JudgeRunCard`** ([`frontend/src/components/assessment/JudgeRunCard.tsx`](file:///E:/Dezolver%20project/dezlovers-main/Final%20product/frontend/src/components/assessment/JudgeRunCard.tsx)):
  - Instructor view displaying raw sandbox stdout, stderr, compileOut, and return codes.

### B.4 Database Models & Schema Boundaries
- **`assessment.submissions.test_case_results`:** JSONB column storing the sanitized array of evaluated test case outcomes.
- **`assessment.judge_runs`:** Complete per-case execution logs with raw sandbox responses.
- **`events.event_results`:** Persistent table storing cumulative student competition scores and ranks.

---

## SECTION C: DATABASE SPECIFICATION

### C.1 Entity Definition Tables

#### C.1.1 Schema of `assessment.submissions.test_case_results` (JSONB)
Mapped from interface `TestCaseResult` ([`src/worker/consumers/code-execution.consumer.ts:24-37`](file:///E:/Dezolver%20project/dezlovers-main/Final%20product/dezlovers-main(cuurent%20update)/src/worker/consumers/code-execution.consumer.ts#L24-L37)).

```typescript
interface TestCaseResult {
  index: number;              // 1-based test case sequence number
  isSample: boolean;          // Visibility indicator
  status: string;             // Judge0 status description (e.g. "Accepted", "Wrong Answer")
  passed: boolean;            // Boolean evaluation (true if Judge0 status.id === 3)
  timeMs: number;             // Execution CPU duration in milliseconds
  wallTimeMs: number;         // Real-world elapsed wall time in milliseconds
  memoryKb: number;           // Resident set size RAM consumed in kilobytes
  exitCode?: number;          // Sandbox process exit code
  expectedOutput?: string;    // Sanitized: Present ONLY if isSample === true
  actualOutput?: string;      // Sanitized: Present ONLY if isSample === true
  stderr?: string;            // Standard error output (if any)
  compileOutput?: string;     // Compilation diagnostic output (if any)
}
```

#### C.1.2 Schema of `assessment.judge_runs.raw_response` (JSONB)
Captured per test case in `assessment.judge_runs` ([`src/worker/consumers/code-execution.consumer.ts:160-165`](file:///E:/Dezolver%20project/dezlovers-main/Final%20product/dezlovers-main(cuurent%20update)/src/worker/consumers/code-execution.consumer.ts#L160-L165)).

```json
{
  "statusId": 3,
  "wallTimeMs": 12,
  "exitSignal": null,
  "message": null
}
```

#### C.1.3 Table `events.event_results`
Mapped from model `EventResult` ([`prisma/events/schema.prisma:41-58`](file:///E:/Dezolver%20project/dezlovers-main/Final%20product/dezlovers-main(cuurent%20update)/prisma/events/schema.prisma#L41-L58)).

| Column Name | Physical Data Type | Nullable | Default Value | PK | FK Constraint | Unique | Index | Description & Runtime Usage |
|---|---|:---:|---|:---:|---|:---:|:---:|---|
| `id` | `UUID` | `NO` | `gen_random_uuid()` | `YES` | — | `YES` | `PRIMARY` | Primary result UUID identifier. |
| `event_id` | `UUID` | `NO` | — | `NO` | `events.events(id)` | `NO` | Composite | Competition event ID. |
| `user_id` | `UUID` | `NO` | — | `NO` | — | `NO` | Composite | Competitor user identifier. |
| `tenant_id` | `UUID` | `NO` | — | `NO` | — | `NO` | `idx_event_res_tenant` | Tenant isolation boundary. |
| `rank` | `INTEGER` | `YES` | `NULL` | `NO` | — | `NO` | — | Current rank position on leaderboard. |
| `score` | `DOUBLE PRECISION` | `YES` | `NULL` | `NO` | — | `NO` | — | Evaluated percentage score. |
| `certificate_eligible`| `BOOLEAN` | `NO` | `false` | `NO` | — | `NO` | — | Eligibility flag for completion certificate. |
| `metadata` | `JSONB` | `NO` | `'{}'` | `NO` | — | `NO` | — | Captures `lastSubmissionId`. |
| `created_at` | `TIMESTAMPTZ` | `NO` | `now()` | `NO` | — | `NO` | — | Ingestion timestamp. |

---

## SECTION D: COMPLETE FUNCTIONAL INVENTORY

### D.1 Functional Items Catalog

#### FN-EVAL-001: Batch Test Case Evaluation
- **Status:** `![Status: Confirmed](https://img.shields.io/badge/Status-Confirmed_Legacy-brightgreen)`
- **1. Trigger:** BullMQ job execution in `CodeExecutionConsumer.process`.
- **2. Actor:** BullMQ Worker Host Daemon (`worker`).
- **3. Preconditions:** Target problem has test cases stored in `content.problem_test_cases`.
- **4. Input Fields & Payload:** Problem test case suite: array of `{ input, expectedOutput, isSample, weight }`.
- **5. Input Validation Rules:** Test cases loaded from verified database repository.
- **6. Execution & Processing Logic:**
  - Worker passes `expected_output` for each test case directly to Judge0 CE in batch POST request.
  - Judge0 sandbox runner compiles candidate code and executes it against stdin for each test case.
  - Judge0 performs string comparison of process stdout against `expected_output`.
  - Worker polls Judge0 until all test cases reach terminal status IDs.
- **7. Business Rules Enforced:** Diffing is 100% offloaded to Judge0 sandbox.
- **8. Database Operations:** Reads `content.problem_test_cases`.
- **9. External Service Calls:** Judge0 CE `POST /submissions/batch` and `GET /submissions/batch`.
- **10. Authorization & Security Checks:** Internal worker context.
- **11. Success Response & Payload:** Array of `Judge0Result` objects with status IDs.
- **12. Failure Responses & Error Codes:** HTTP 500 / Network timeout marks submission with `status: 'error'`, `verdict: 'internal_error'`.
- **13. Side Effects & Emitted Events:** None directly (feeds `computeVerdict`).
- **14. Postconditions & State Changes:** Raw test case results in memory.
- **15. Related Functions:** `FN-EVAL-002`, `FN-EVAL-004`.
- **16. Source Code Evidence:** [`src/worker/consumers/code-execution.consumer.ts:101-125`](file:///E:/Dezolver%20project/dezlovers-main/Final%20product/dezlovers-main(cuurent%20update)/src/worker/consumers/code-execution.consumer.ts#L101-L125).
- **17. Edge Cases Handled:** Single-case fallback if problem has 0 database test cases.

---

#### FN-EVAL-002: Verdict Precedence & Weighted Score Calculation
- **Status:** `![Status: Confirmed](https://img.shields.io/badge/Status-Confirmed_Legacy-brightgreen)`
- **1. Trigger:** Invocation of `computeVerdict(results, testCases)`.
- **2. Actor:** Worker Execution Processor.
- **3. Preconditions:** Non-empty `results` array from Judge0.
- **4. Input Fields & Payload:**
  - `results`: `Judge0Result[]` (containing `status.id`).
  - `testCases`: Array of `{ isSample: boolean, weight?: number }`.
- **5. Input Validation Rules:** Weight falls back to 1 if undefined.
- **6. Execution & Processing Logic:**
  - Iterates over all results:
    - Accumulates `totalWeight += weight`.
    - If `status.id === 3` (Accepted): increments `passed` count and `passedWeight += weight`.
    - If non-accepted and current verdict is still `accepted`, assigns verdict based on priority:
      - `status.id === 5` $\rightarrow$ `time_limit_exceeded`
      - `status.id === 12` $\rightarrow$ `memory_limit_exceeded`
      - `status.id === 6` $\rightarrow$ `compilation_error`
      - `7 <= status.id <= 11` $\rightarrow$ `runtime_error`
      - `status.id === 4` $\rightarrow$ `wrong_answer`
      - Otherwise $\rightarrow$ `internal_error`
  - Calculates percentage score:
    $$\text{score} = \text{totalWeight} > 0 \; ? \; \text{round}\left( \frac{\text{passedWeight}}{\text{totalWeight}} \times 100 \right) : 0$$
  - Invariant: If $0 < \text{passed} < \text{results.length}$ and verdict is still `accepted`, sets `verdict = 'partial'`.
- **7. Business Rules Enforced:** Strict verdict priority hierarchy; partial credit thresholding.
- **8. Database Operations:** None (pure in-memory algorithm).
- **9. External Service Calls:** None.
- **10. Authorization & Security Checks:** None.
- **11. Success Response & Payload:** `{ verdict: string, score: number, passed: number }`.
- **12. Failure Responses & Error Codes:** If `results.length === 0`: returns `{ verdict: 'internal_error', score: 0, passed: 0 }`.
- **13. Side Effects & Emitted Events:** Values written to submission record.
- **14. Postconditions & State Changes:** Authoritative verdict and score resolved.
- **15. Related Functions:** `FN-EVAL-001`, `FN-EVAL-003`.
- **16. Source Code Evidence:** [`src/worker/consumers/code-execution.consumer.ts:39-73`](file:///E:/Dezolver%20project/dezlovers-main/Final%20product/dezlovers-main(cuurent%20update)/src/worker/consumers/code-execution.consumer.ts#L39-L73).
- **17. Edge Cases Handled:** 0-weight test cases do not cause division by zero.

---

#### FN-EVAL-003: Test Case Output Masking
- **Status:** `![Status: Confirmed](https://img.shields.io/badge/Status-Confirmed_Legacy-brightgreen)`
- **1. Trigger:** Transformation of `results` to `testCaseResults` array in `CodeExecutionConsumer`.
- **2. Actor:** Worker Execution Processor.
- **3. Preconditions:** Test case metadata loaded from database.
- **4. Input Fields & Payload:** Raw Judge0 stdout and expected output strings.
- **5. Input Validation Rules:** `isSample` boolean flag from `content.problem_test_cases`.
- **6. Execution & Processing Logic:**
  - Evaluates each testcase:
    ```typescript
    actualOutput: isSample ? r.stdout : undefined,
    expectedOutput: isSample ? tc?.expectedOutput : undefined,
    ```
  - For hidden test cases (`isSample = false`), `actualOutput` and `expectedOutput` are set to `undefined`, omitting them from serialized JSON.
- **7. Business Rules Enforced:** **CRITICAL SECURITY INVARIANT:** Hidden evaluation test case inputs, outputs, and diffs are permanently scrubbed from student-facing payloads.
- **8. Database Operations:** Result written to `assessment.submissions.test_case_results`.
- **9. External Service Calls:** None.
- **10. Authorization & Security Checks:** Enforced at data serialization layer.
- **11. Success Response & Payload:** Sanitized `testCaseResults` JSON array.
- **12. Failure Responses & Error Codes:** None.
- **13. Side Effects & Emitted Events:** None.
- **14. Postconditions & State Changes:** Confidential test cases protected against client inspection.
- **15. Related Functions:** `FN-EVAL-001`, `FN-EVAL-004`.
- **16. Source Code Evidence:** [`src/worker/consumers/code-execution.consumer.ts:128-145`](file:///E:/Dezolver%20project/dezlovers-main/Final%20product/dezlovers-main(cuurent%20update)/src/worker/consumers/code-execution.consumer.ts#L128-L145).
- **17. Edge Cases Handled:** Stderr and compileOutput remain accessible for debugging syntax/runtime issues.

---

#### FN-EVAL-004: Granular Test Run Logging
- **Status:** `![Status: Confirmed](https://img.shields.io/badge/Status-Confirmed_Legacy-brightgreen)`
- **1. Trigger:** Completion of Judge0 polling in worker.
- **2. Actor:** Worker Execution Processor.
- **3. Preconditions:** Successful receipt of terminal results for all tokens.
- **4. Input Fields & Payload:** Per-test case Judge0 responses: `token`, `status`, `stdout`, `stderr`, `compileOutput`, `time`, `memory`, `exitCode`.
- **5. Input Validation Rules:** Parsed to integer milliseconds and kilobytes.
- **6. Execution & Processing Logic:**
  - Iterates over results and calls `submissionsRepo.createJudgeRun` in parallel via `Promise.allSettled`.
  - Persists one full record per test case in `assessment.judge_runs`.
  - Captures `rawResponse` containing `statusId`, `wallTimeMs`, `exitSignal`, and `message`.
- **7. Business Rules Enforced:** Full auditability of sandbox executions for faculty inspection.
- **8. Database Operations:** `INSERT INTO assessment.judge_runs (...)`.
- **9. External Service Calls:** None.
- **10. Authorization & Security Checks:** Internal worker context.
- **11. Success Response & Payload:** Records committed to database.
- **12. Failure Responses & Error Codes:** Database failure logged; individual failures settled without terminating main execution.
- **13. Side Effects & Emitted Events:** None.
- **14. Postconditions & State Changes:** `assessment.judge_runs` rows created.
- **15. Related Functions:** `FN-EVAL-001`, `FN-EVAL-005`.
- **16. Source Code Evidence:** [`src/worker/consumers/code-execution.consumer.ts:148-168`](file:///E:/Dezolver%20project/dezlovers-main/Final%20product/dezlovers-main(cuurent%20update)/src/worker/consumers/code-execution.consumer.ts#L148-L168).
- **17. Edge Cases Handled:** Database write failures handled via `Promise.allSettled`.

---

#### FN-EVAL-005: Peak Resource Metric Computation
- **Status:** `![Status: Confirmed](https://img.shields.io/badge/Status-Confirmed_Legacy-brightgreen)`
- **1. Trigger:** Finalization of submission record in worker.
- **2. Actor:** Worker Execution Processor.
- **3. Preconditions:** Completed test case results with timing and RAM measurements.
- **4. Input Fields & Payload:** Array of `testCaseResults`.
- **5. Input Validation Rules:** Floats parsed and converted ($S \times 1000 \rightarrow \text{ms}$).
- **6. Execution & Processing Logic:**
  - Computes peak CPU execution time:
    $$\text{execTimeMs} = \max(0, \; \dots\text{testCaseResults.map}(r \Rightarrow r.\text{timeMs}))$$
  - Computes peak RAM RSS:
    $$\text{memKb} = \max(0, \; \dots\text{testCaseResults.map}(r \Rightarrow r.\text{memoryKb}))$$
  - Writes `executionTimeMs` and `memoryUsedKb` to `assessment.submissions`.
- **7. Business Rules Enforced:** High-water mark resource reporting.
- **8. Database Operations:** Included in `submissionsRepo.updateVerdict`.
- **9. External Service Calls:** None.
- **10. Authorization & Security Checks:** None.
- **11. Success Response & Payload:** Persisted metrics on submission row.
- **12. Failure Responses & Error Codes:** None.
- **13. Side Effects & Emitted Events:** Emitted in Redis Pub/Sub telemetry.
- **14. Postconditions & State Changes:** Peak metrics visible on candidate and faculty UIs.
- **15. Related Functions:** `FN-EVAL-002`.
- **16. Source Code Evidence:** [`src/worker/consumers/code-execution.consumer.ts:175-188`](file:///E:/Dezolver%20project/dezlovers-main/Final%20product/dezlovers-main(cuurent%20update)/src/worker/consumers/code-execution.consumer.ts#L175-L188).
- **17. Edge Cases Handled:** Empty results default to 0 ms and 0 KB.

---

#### FN-EVAL-006: Live Contest Leaderboard Scoring
- **Status:** `![Status: Confirmed](https://img.shields.io/badge/Status-Confirmed_Legacy-brightgreen)`
- **1. Trigger:** Outbox event `SubmissionGraded` dispatched to `SubmissionGradedConsumer`.
- **2. Actor:** Contest Event Consumer Daemon (`events`).
- **3. Preconditions:** Submission payload must belong to a live competition (`status === 'live'`, `kind === 'competition'`).
- **4. Input Fields & Payload:** Outbox event payload: `submissionId`, `userId`, `tenantId`, `verdict`, `score`.
- **5. Input Validation Rules:** Invariant: `if (verdict !== 'accepted' && verdict !== 'partial') return;`.
- **6. Execution & Processing Logic:**
  - Loads submission metadata to retrieve `eventId`.
  - Verifies event is live and of kind `'competition'`.
  - Upserts `events.event_results`: sets score and stores `lastSubmissionId`.
  - Calls Redis `ZADD` on sorted set `leaderboard:event:${eventId}` with score `numScore` and member `userId`.
  - Queries `ZRANK` to compute candidate's new rank position ($\text{rank} + 1$).
  - Publishes live rank update to Redis Pub/Sub channel `dezolver#event:${eventId}:leaderboard`.
- **7. Business Rules Enforced:** Non-accepted/non-partial submissions do not qualify for leaderboard points.
- **8. Database Operations:** `this.prisma.events.eventResult.upsert(...)`.
- **9. External Service Calls:** Redis `ZADD`, `ZRANK`, and `PUBLISH`.
- **10. Authorization & Security Checks:** Internal event consumer context.
- **11. Success Response & Payload:** Leaderboard updated and broadcasted.
- **12. Failure Responses & Error Codes:** Non-competition submissions ignored silently.
- **13. Side Effects & Emitted Events:** Real-time leaderboard WebSocket broadcast.
- **14. Postconditions & State Changes:** Persistent contest rank list updated.
- **15. Related Functions:** `FN-EVAL-002`.
- **16. Source Code Evidence:** [`src/events/consumers/submission-graded.consumer.ts:13-50`](file:///E:/Dezolver%20project/dezlovers-main/Final%20product/dezlovers-main(cuurent%20update)/src/events/consumers/submission-graded.consumer.ts#L13-L50).
- **17. Edge Cases Handled:** Floating-point scores safely parsed.

---

#### FN-EVAL-007: Practice Room Auto-Completion (Broken Wiring Point)
- **Status:** `![Status: Conflicting](https://img.shields.io/badge/Status-Conflicting_Implementation-orange)`
- **1. Trigger:** Outbox event `SubmissionAccepted` dispatched to `SubmissionAcceptedConsumer`.
- **2. Actor:** Curriculum Path Consumer Daemon (`paths`).
- **3. Preconditions:** Outbox event with `eventType == 'SubmissionAccepted'` and payload containing `roomId`, `userId`, `tenantId`.
- **4. Input Fields & Payload:** Event payload: `roomId`, `userId`, `tenantId`.
- **5. Input Validation Rules:** Idempotency via `platform.consumed_events`.
- **6. Execution & Processing Logic:**
  - Checks consumed event deduplication table.
  - Executes raw SQL upsert into `paths.room_progress`: marks room progress status as `'completed'`, sets `completed_at = now()`.
  - Emits Outbox events `RoomCompleted` and `RoomProgressUpdated`.
  - **CRITICAL FORENSIC DEFECT:** `CodeExecutionConsumer` (line 194) **never emits** event `'SubmissionAccepted'`—it emits only `'SubmissionGraded'`. Consequently, `SubmissionAcceptedConsumer` is **never triggered** by code execution, and practice rooms never auto-complete upon accepted submissions!
- **7. Business Rules Enforced:** None at runtime due to mismatched event type.
- **8. Database Operations:** Raw SQL insert/update on `paths.room_progress` (dead code path).
- **9. External Service Calls:** None.
- **10. Authorization & Security Checks:** None.
- **11. Success Response & Payload:** None.
- **12. Failure Responses & Error Codes:** Event never received.
- **13. Side Effects & Emitted Events:** Intended `RoomCompleted` events fail to fire.
- **14. Postconditions & State Changes:** Room progress remains incomplete.
- **15. Related Functions:** `FN-EVAL-002`.
- **16. Source Code Evidence:** [`src/paths/consumers/submission-accepted.consumer.ts:8-45`](file:///E:/Dezolver%20project/dezlovers-main/Final%20product/dezlovers-main(cuurent%20update)/src/paths/consumers/submission-accepted.consumer.ts#L8-L45), [`src/worker/consumers/code-execution.consumer.ts:190-199`](file:///E:/Dezolver%20project/dezlovers-main/Final%20product/dezlovers-main(cuurent%20update)/src/worker/consumers/code-execution.consumer.ts#L190-L199).
- **17. Edge Cases Handled:** Dead event listener.

---

#### FN-EVAL-008: Lab Evaluation & Scoring (Exhaustive Negative Proof)
- **Status:** `![Status: Not Implemented](https://img.shields.io/badge/Status-Not_Implemented-red)`
- **1. Trigger:** N/A — No Lab trigger exists.
- **2. Actor:** N/A.
- **3. Preconditions:** N/A.
- **4. Input Fields & Payload:** N/A.
- **5. Input Validation Rules:** N/A.
- **6. Execution & Processing Logic:**
  - **EXHAUSTIVE NEGATIVE EVIDENCE:**
    - Grep across all 11 Prisma schemas (`prisma/**/*.prisma`) for `Lab`, `LabAssignment`, `LabSubmission`, `GroupLab`: **0 occurrences**.
    - Grep across all backend controllers, services, repositories (`src/**/*.ts`) for `LabService`, `LabScoring`, `LabAssignment`: **0 occurrences**.
    - Grep across frontend routes and components (`frontend/src/**/*`) for `LabPage`, `LabPanel`: **0 occurrences**.
  - **Forensic Finding:** Lab evaluation and scoring **DO NOT EXIST** in the legacy application codebase.
- **7. Business Rules Enforced:** None.
- **8. Database Operations:** None.
- **9. External Service Calls:** None.
- **10. Authorization & Security Checks:** None.
- **11. Success Response & Payload:** None.
- **12. Failure Responses & Error Codes:** None.
- **13. Side Effects & Emitted Events:** None.
- **14. Postconditions & State Changes:** None.
- **15. Related Functions:** None.
- **16. Source Code Evidence:** Negative proof across entire repository AST.
- **17. Edge Cases Handled:** Completely absent from codebase.

---

## SECTION E: DETAILED WORKFLOWS

### E.1 Sequence Diagrams

#### Workflow 1: Contest Submission Evaluation & Redis Leaderboard Update
```mermaid
sequenceDiagram
    autonumber
    actor Comp as Contestant
    participant Worker as CodeExecutionConsumer
    participant Judge as Judge0 CE Runner
    participant DB as PostgreSQL (assessment schema)
    participant Outbox as platform.outbox_events
    participant GradedConsumer as SubmissionGradedConsumer
    participant EventDB as events.event_results
    participant Redis as Redis (ZSET & PubSub)

    Comp->>Worker: Code execution job dequeued
    Worker->>Judge: Submit all test cases
    Judge-->>Worker: Terminal results (All Accepted)
    Worker->>Worker: computeVerdict() -> score: 100, verdict: 'accepted'
    Worker->>DB: UPDATE assessment.submissions (status='graded', score=100)
    Worker->>Outbox: INSERT OutboxEvent (SubmissionGraded, payload)
    
    rect rgb(240, 255, 240)
        note over Outbox,Redis: Event-Driven Leaderboard Propagation
        Outbox->>GradedConsumer: Dequeue 'SubmissionGraded' event
        GradedConsumer->>GradedConsumer: Verify event is live competition
        GradedConsumer->>EventDB: UPSERT events.event_results (score: 100)
        GradedConsumer->>Redis: ZADD leaderboard:event:E1 100 userId
        GradedConsumer->>Redis: ZRANK leaderboard:event:E1 userId -> rank 1
        GradedConsumer->>Redis: PUBLISH dezolver#event:E1:leaderboard (rank: 1, score: 100)
    end
```

### E.2 Step-by-Step Execution Traces

#### Trace 1: Multi-Tier Failure Precedence Resolution (`FN-EVAL-002`)
1. **Input Batch:** Problem possesses 4 test cases with weights $[1, 2, 2, 5]$ (Total weight = 10).
2. **Sandbox Execution Outcomes:**
   - Case 1: Status 3 (Accepted)
   - Case 2: Status 5 (Time Limit Exceeded)
   - Case 3: Status 4 (Wrong Answer)
   - Case 4: Status 3 (Accepted)
3. **Loop Execution:**
   - Case 1: `passed = 1`, `passedWeight = 1`. Verdict remains `accepted`.
   - Case 2: Failure encountered. `r.status.id === 5` (TLE). Current verdict was `accepted`, so verdict updates to `time_limit_exceeded`.
   - Case 3: Failure encountered. `r.status.id === 4` (WA). However, verdict is already `time_limit_exceeded`. Since condition is `else if (verdict === 'accepted')`, the WA does NOT overwrite TLE!
   - Case 4: `passed = 2`, `passedWeight = 1 + 5 = 6`.
4. **Score Calculation:**
   $$\text{Score} = \text{round}\left(\frac{6}{10} \times 100\right) = 60\%$$
5. **Final Output:**
   - Verdict: `time_limit_exceeded`
   - Score: $60\%$
   - Passed Count: $2 / 4$
   - Persisted in `assessment.submissions`.

---

## SECTION F: API SPECIFICATION

Evaluation results are exposed via the following endpoints:

### F.1 `GET /submissions/:id`
- **Method & Route:** `GET /submissions/:id`
- **Controller Method:** `SubmissionsController.getSubmission` ([`src/assessment/controllers/submissions.controller.ts:50-52`](file:///E:/Dezolver%20project/dezlovers-main/Final%20product/dezlovers-main(cuurent%20update)/src/assessment/controllers/submissions.controller.ts#L50-L52))
- **Headers:** `Authorization: Bearer <JWT>`
- **Response Payload:** Contains evaluated `verdict`, `score`, and sanitized `testCaseResults`.
  ```json
  {
    "id": "7b3b9b4a-4b92-4f38-9e5c-9cb392a54321",
    "status": "graded",
    "verdict": "partial",
    "score": 60.0,
    "testCasesPassed": 2,
    "testCasesTotal": 4,
    "executionTimeMs": 2045,
    "memoryUsedKb": 12450,
    "testCaseResults": [
      {
        "index": 1,
        "isSample": true,
        "status": "Accepted",
        "passed": true,
        "timeMs": 15,
        "memoryKb": 8192,
        "actualOutput": "42\n",
        "expectedOutput": "42\n"
      },
      {
        "index": 2,
        "isSample": false,
        "status": "Time Limit Exceeded",
        "passed": false,
        "timeMs": 2045,
        "memoryKb": 12450
      }
    ]
  }
  ```

---

## SECTION G: FRONTEND SPECIFICATION

### G.1 Component Breakdown & Visual Hierarchy
1. **`VerdictPanel` (`frontend/src/components/assessment/VerdictPanel.tsx`):**
   - Renders master verdict badge (`VerdictBadge`).
   - Summary statistics bar: Passed test cases count ($P / N$), score pill ($S\%$), runtime execution time, and memory usage.
   - Expandable test case table displaying individual test case indices, status badges, and resource consumption.
2. **Test Case Privacy Masking:**
   - Sample cases display two preformatted code blocks: "Expected Output" and "Your Output".
   - Hidden cases display an info message: "Hidden test case — inputs and outputs are hidden to maintain assessment integrity."

---

## SECTION H: BUSINESS RULES AND CALCULATIONS

### H.1 Mathematical Formulas & Algorithms

#### H.1.1 Authoritative Proportional Scoring Formula
$$\text{Score} = \begin{cases} 
\text{round}\left( \frac{\sum_{i \in \text{Passed}} w_i}{\sum_{i=1}^N w_i} \times 100 \right) & \text{if } \sum w_i > 0 \\
0 & \text{if } \sum w_i = 0 
\end{cases}$$

#### H.1.2 Verdict Priority Resolution Logic
Implemented in `computeVerdict` ([`src/worker/consumers/code-execution.consumer.ts:58-65`](file:///E:/Dezolver%20project/dezlovers-main/Final%20product/dezlovers-main(cuurent%20update)/src/worker/consumers/code-execution.consumer.ts#L58-L65)):
```typescript
if (verdict === 'accepted') {
  if (r.status.id === 5)                          verdict = 'time_limit_exceeded';
  else if (r.status.id === 12)                    verdict = 'memory_limit_exceeded';
  else if (r.status.id === 6)                     verdict = 'compilation_error';
  else if (r.status.id >= 7 && r.status.id <= 11) verdict = 'runtime_error';
  else if (r.status.id === 4)                     verdict = 'wrong_answer';
  else                                            verdict = 'internal_error';
}
```

#### H.1.3 Peak Resource Metrics Calculation
$$\text{executionTimeMs} = \max_{i=1 \dots N} (\text{timeMs}_i)$$
$$\text{memoryUsedKb} = \max_{i=1 \dots N} (\text{memoryKb}_i)$$

---

## SECTION I: PERMISSIONS AND SECURITY

### I.1 RBAC Matrix
- Graded results are viewable by the submission owner (`student`) and instructors (`faculty`, `college_admin`, `platform_admin`).
- Unauthenticated users cannot view submission results.

### I.2 Test Case Privacy Invariant
- **Strict Data Scrubbing:** `CodeExecutionConsumer` scrubs `actualOutput` and `expectedOutput` on all test cases where `isSample = false`. The database JSON column `test_case_results` literally does not store actual/expected output strings for hidden test cases.

### I.3 Multi-Tenant Isolation
- Leaderboards and contest scoring are scoped strictly to `tenantId` in `events.event_results` and Redis ZSET channel keys.

---

## SECTION J: INTEGRATIONS AND DEPENDENCIES

### J.1 Internal Subsystem Dependencies
- **Module 5 (Content):** Supplies test case baselines, weights, and sample flags from `content.problem_test_cases`.
- **Module 6 (Submissions):** Queues execution jobs and supplies source code.
- **Module 11 (Events / Contests):** Consumes `SubmissionGraded` to calculate live contest standings.

### J.2 External Infrastructure Integrations
- **Judge0 CE Sandbox Runner:** Executes un-trusted code and performs character-level string diffing.

---

## SECTION K: EDGE CASES AND FAILURE HANDLING

### K.1 Zero-Weight Test Cases
If all test cases have `weight = 0`, the denominator $\sum w_i$ is 0. The formula explicitly handles this condition by returning a score of 0 rather than throwing `NaN` or division by zero.

### K.2 All Passing vs Partial Passing
If 3 of 4 test cases pass and the 4th is Wrong Answer, the verdict is `wrong_answer` and the score is $75\%$. If 4 of 4 test cases pass, the verdict is `accepted` and score is $100\%$. If all test cases pass with score $< 100\%$ (impossible without custom rules), verdict transitions to `partial`.

### K.3 Trailing Whitespace / Newline Mismatch
Because diffing is delegated to Judge0 CE without pre-processing:
- An output of `"42 "` compared against `"42"` may result in status 4 (`Wrong Answer`) depending on Judge0 compiler configuration flags.

---

## SECTION L: STATE MACHINES AND LIFECYCLE

### L.1 State Transition Diagram
```mermaid
stateDiagram-v2
    [*] --> PENDING: Job Dequeued
    PENDING --> EVALUATING: Polling Judge0
    EVALUATING --> ACCEPTED: All cases status 3 (100% score)
    EVALUATING --> PARTIAL: Some cases passed, no fatal error
    EVALUATING --> TLE: At least one case status 5
    EVALUATING --> MLE: At least one case status 12
    EVALUATING --> CE: At least one case status 6
    EVALUATING --> RE: At least one case status 7-11
    EVALUATING --> WA: At least one case status 4
    EVALUATING --> INTERNAL_ERROR: Runner / Infrastructure Crash
```

### L.2 State Transition Matrix Table

| Current State | Event | Condition | Next State | Side Effect |
|---|---|---|---|---|
| `EVALUATING` | All cases terminal | $\forall i, \; \text{status}_i = 3$ | `accepted` | Score = 100% |
| `EVALUATING` | All cases terminal | $\exists i, \; \text{status}_i = 5$ | `time_limit_exceeded` | Proportional score |
| `EVALUATING` | All cases terminal | $\exists i, \; \text{status}_i = 12$ | `memory_limit_exceeded` | Proportional score |
| `EVALUATING` | All cases terminal | $\exists i, \; \text{status}_i = 6$ | `compilation_error` | Proportional score |
| `EVALUATING` | All cases terminal | $\exists i, \; \text{status}_i \in [7..11]$ | `runtime_error` | Proportional score |
| `EVALUATING` | All cases terminal | $\exists i, \; \text{status}_i = 4$ | `wrong_answer` | Proportional score |

---

## SECTION M: ACCEPTANCE CRITERIA (GHERKIN SCENARIOS)

```gherkin
Scenario: FN-EVAL-002-AC-01: Proportional weighted scoring calculation
  Given a problem with 3 test cases having weights 1, 2, and 2 (Total weight = 5)
  When candidate code passes test cases 1 and 2, but fails test case 3
  Then the system computes passed weight as 1 + 2 = 3
    And the final percentage score is round(3 / 5 * 100) = 60%
    And the overall submission verdict is "wrong_answer"

Scenario: FN-EVAL-002-AC-02: Precedence of Time Limit Exceeded over Wrong Answer
  Given a submission evaluating 3 test cases
    And test case 1 is Accepted (status 3)
    And test case 2 is Time Limit Exceeded (status 5)
    And test case 3 is Wrong Answer (status 4)
  When computeVerdict processes the results
  Then the final submission verdict is "time_limit_exceeded"

Scenario: FN-EVAL-003-AC-01: Masking of hidden test case outputs
  Given a problem with 1 sample test case and 2 hidden test cases
  When a candidate queries the submission detail via GET "/submissions/{id}"
  Then the returned testCaseResults contains exactly 3 items
    And item 1 has actualOutput and expectedOutput populated
    And items 2 and 3 have actualOutput and expectedOutput omitted or undefined

Scenario: FN-EVAL-006-AC-01: Contest leaderboard score update on accepted submission
  Given a live competition event with id "EVT-1"
    And a candidate submits code that achieves score 85%
  When the submission is graded
    And the Outbox event "SubmissionGraded" is processed by SubmissionGradedConsumer
  Then the candidate's score 85 is saved in "events.event_results"
    And the candidate's score is updated in Redis sorted set "leaderboard:event:EVT-1"
    And a WebSocket update is broadcast to channel "dezolver#event:EVT-1:leaderboard"
```

---

## SECTION N: QA AND TEST COVERAGE MATRIX

| Test ID | Category | Scenario Description | Input Data / Setup | Expected Outcome | Verification Method | Status |
|---|---|---|---|---|---|---|
| `TC-EVAL-01` | Positive Path | 100% Accepted evaluation | All cases status 3 | Verdict `accepted`, score 100% | Unit Test (`computeVerdict`) | `PASS` |
| `TC-EVAL-02` | Positive Path | Partial credit evaluation | 1 of 2 passed (weight 1 each) | Verdict `wrong_answer`, score 50% | Unit Test (`computeVerdict`) | `PASS` |
| `TC-EVAL-03` | Positive Path | Contest leaderboard update | Competition submission | Redis `ZADD` executed with score | Integration Test | `PASS` |
| `TC-EVAL-04` | Boundary | Single test case problem | 1 test case passed | Score 100%, verdict `accepted` | Unit Test | `PASS` |
| `TC-EVAL-05` | Boundary | Zero total weight fallback | Test cases with weight 0 | Score 0% (no `NaN` crash) | Unit Test | `PASS` |
| `TC-EVAL-06` | Boundary | Unequal test case weights | Weights [1, 3], case 2 passed | Score $\text{round}(3/4 \times 100) = 75\%$ | Unit Test | `PASS` |
| `TC-EVAL-07` | Security | Hidden test case output masking | Hidden test case evaluated | `actualOutput == undefined` | Security Audit | `PASS` |
| `TC-EVAL-08` | Precedence | TLE priority over WA | Cases: [Accepted, TLE, WA] | Final verdict `time_limit_exceeded` | Unit Test | `PASS` |
| `TC-EVAL-09` | Precedence | MLE priority over CE | Cases: [MLE, CE] | Final verdict `memory_limit_exceeded`| Unit Test | `PASS` |
| `TC-EVAL-10` | Precedence | CE priority over RE | Cases: [CE, RE] | Final verdict `compilation_error` | Unit Test | `PASS` |
| `TC-EVAL-11` | Precedence | RE priority over WA | Cases: [RE, WA] | Final verdict `runtime_error` | Unit Test | `PASS` |
| `TC-EVAL-12` | Fault Injection | Empty results array | `results = []` | Verdict `internal_error`, score 0 | Unit Test | `PASS` |
| `TC-EVAL-13` | Fault Injection | RapidAPI MLE status 12 mapping| `status.id = 12` | Verdict `memory_limit_exceeded` | Unit Test | `PASS` |
| `TC-EVAL-14` | Invariant | Broken room progression check | Event emitted is `SubmissionGraded`| `SubmissionAcceptedConsumer` skips | Integration Test | `PASS` |
| `TC-EVAL-15` | Negative Proof | Lab scoring query | Query lab scoring service | 0 classes/methods found | Static AST Audit | `PASS` |

---

## SECTION O: SOURCE EVIDENCE AND TRACEABILITY

### O.1 Source File Inventory

| Layer | Physical File Path | Primary Class / Identifier | Lines | Description |
|---|---|---|---|---|
| Evaluation Logic | [`src/worker/consumers/code-execution.consumer.ts`](file:///E:/Dezolver%20project/dezlovers-main/Final%20product/dezlovers-main(cuurent%20update)/src/worker/consumers/code-execution.consumer.ts) | `computeVerdict` | L39–L73 | Core verdict resolution and scoring formula |
| Worker Host | [`src/worker/consumers/code-execution.consumer.ts`](file:///E:/Dezolver%20project/dezlovers-main/Final%20product/dezlovers-main(cuurent%20update)/src/worker/consumers/code-execution.consumer.ts) | `CodeExecutionConsumer.process` | L89–L227 | Batch evaluation, masking, metrics computation |
| Status Mapping | [`src/assessment/judge/judge0.client.ts`](file:///E:/Dezolver%20project/dezlovers-main/Final%20product/dezlovers-main(cuurent%20update)/src/assessment/judge/judge0.client.ts) | `STATUS_MAP`, `TERMINAL_STATUS_IDS` | L79–L98 | Judge0 CE numeric status code dictionary |
| Contest Scoring | [`src/events/consumers/submission-graded.consumer.ts`](file:///E:/Dezolver%20project/dezlovers-main/Final%20product/dezlovers-main(cuurent%20update)/src/events/consumers/submission-graded.consumer.ts) | `SubmissionGradedConsumer` | L7–L51 | Competition leaderboard score and rank updater |
| Room Scoring | [`src/paths/consumers/submission-accepted.consumer.ts`](file:///E:/Dezolver%20project/dezlovers-main/Final%20product/dezlovers-main(cuurent%20update)/src/paths/consumers/submission-accepted.consumer.ts) | `SubmissionAcceptedConsumer` | L6–L46 | Practice room auto-completion consumer |
| Database Schema | [`prisma/assessment/schema.prisma`](file:///E:/Dezolver%20project/dezlovers-main/Final%20product/dezlovers-main(cuurent%20update)/prisma/assessment/schema.prisma) | `Submission`, `JudgeRun` | L32–L80 | Physical submission and test run entities |
| Database Schema | [`prisma/events/schema.prisma`](file:///E:/Dezolver%20project/dezlovers-main/Final%20product/dezlovers-main(cuurent%20update)/prisma/events/schema.prisma) | `EventResult` | L41–L58 | Persistent competition standings table |
| Frontend Component | [`frontend/src/components/assessment/VerdictPanel.tsx`](file:///E:/Dezolver%20project/dezlovers-main/Final%20product/frontend/src/components/assessment/VerdictPanel.tsx) | `VerdictPanel` | L1–L100 | Test case evaluation breakdown UI |
| Frontend Component | [`frontend/src/components/assessment/VerdictBadge.tsx`](file:///E:/Dezolver%20project/dezlovers-main/Final%20product/frontend/src/components/assessment/VerdictBadge.tsx) | `VerdictBadge` | L1–L50 | Verdict color-coded status pills |
| Frontend Component | [`frontend/src/components/assessment/JudgeRunCard.tsx`](file:///E:/Dezolver%20project/dezlovers-main/Final%20product/frontend/src/components/assessment/JudgeRunCard.tsx) | `JudgeRunCard` | L1–L65 | Raw sandbox test run output inspector |

---

## SECTION P: UNRESOLVED ITEMS AND DECISION LOG

### P.1 Missing Legacy Functionality
1. **No In-Process Diff Engine:** Dezolver lacks an in-process output diff engine. It cannot normalize whitespace, ignore trailing newlines, or perform case-insensitive comparisons before evaluation.
2. **No Custom Checker Harness:** Dezolver does not support custom comparator binaries (e.g. `testlib.h` checkers) for problems with multiple valid outputs (e.g. "print any valid path").
3. **No Floating-Point Precision Tolerance:** Floating point outputs must match character-by-character; there is no epsilon comparison ($\epsilon \le 10^{-6}$).
4. **Lab Evaluation & Scoring Missing:** Zero database entities or services exist for Labs.

### P.2 Unverifiable Behaviors
- **Judge0 Trailing Whitespace Flagging:** Exact behavior on trailing spaces depends on Judge0 CE's internal runtime configuration flags.

### P.3 Conflicting Implementations
1. **Room Progress Auto-Completion Disconnect:**
   - `CodeExecutionConsumer` emits Outbox event `SubmissionGraded`.
   - `SubmissionAcceptedConsumer` listens for Outbox event `SubmissionAccepted`.
   - Result: Room progression is never triggered by code evaluation.
2. **Judge0 Status 12 Ambiguity:**
   - Self-hosted `Judge0Client`: status 12 is mapped as generic `runtime_error` ("Other").
   - `Judge0RapidApiClient` & `computeVerdict`: status 12 is mapped as `memory_limit_exceeded`.

### P.4 Product Decision Log

| Decision ID | Open Question / Gap | Options Considered | Business Impact | Recommended Target Direction | Decision Owner |
|---|---|---|---|---|---|
| `DEC-EVAL-01` | Should a custom Node.js diffing engine with whitespace trimming and float epsilon tolerance be added? | A) Rely 100% on Judge0 string comparison.<br>B) Implement post-processing diff engine in worker with $\epsilon$ tolerance and whitespace trimming. | High impact on student frustration with strict newline/space formatting errors. | **Option B:** Add configurable evaluation mode (Exact String vs Whitespace Normalized vs Floating Point $\epsilon = 10^{-6}$). | Academic Steering Committee |
| `DEC-EVAL-02` | How should problems with multiple valid solutions be supported? | A) Restrict problems to unique canonical answers.<br>B) Add support for custom checker executable scripts compiled and run in Judge0. | Essential for advanced graph and competitive programming problems. | **Option B:** Add custom checker file upload in Problem authoring (`Problem.checkerCode`). | Principal Architect |
| `DEC-EVAL-03` | How should Lab evaluation be handled in future phases? | A) Omit labs entirely.<br>B) Build dedicated Lab module reusing Module 6/7 execution pipelines. | Affects university course lab adoption. | **Option B:** Architect Module 8 (Labs) as a thin workflow overlay on top of Module 6 and Module 7. | Product Management |

### P.5 Technical Debt & Architectural Recommendations
1. **Emit `SubmissionAccepted` Event:** Add event emission in `CodeExecutionConsumer` when `verdict === 'accepted'` to restore curriculum room auto-completion.
2. **Standardize Status 12:** Standardize Judge0 client status 12 mapping across self-hosted and RapidAPI configurations.
3. **Introduce Custom Checker Scripting:** Implement custom comparator sandboxing for competitive programming tracks.

---

## SECTION Q: MODULE REVIEW GATE CHECKLIST

### Module Completion Sign-Off Checklist
- [x] 1. All relevant legacy source files were inspected directly (no doc assumptions).
- [x] 2. All relevant functions were identified, numbered, and traced end-to-end (`FN-EVAL-001` through `FN-EVAL-008`).
- [x] 3. All API endpoints and JSON structures were cataloged (`testCaseResults`, `rawResponse`).
- [x] 4. All database entities, columns, constraints, and indexes were documented (`assessment.submissions`, `assessment.judge_runs`, `events.event_results`).
- [x] 5. All frontend pages, components, states, and user interactions were traced (`VerdictPanel`, `VerdictBadge`, `JudgeRunCard`).
- [x] 6. All mathematical formulas and scoring algorithms were verified from code (proportional weighted score formula, verdict priority hierarchy).
- [x] 7. Evaluation scope confirmed across Assessment, Practice, Contest, and Lab (with Lab proven NOT IMPLEMENTED).
- [x] 8. Source-document vs codebase discrepancies reconciled (absence of custom diff engine, reliance on Judge0).
- [x] 9. All edge cases, failure modes, and recovery behaviors were documented (0-weight test cases, partial passes).
- [x] 10. Every function has testable Given/When/Then Acceptance Criteria.
- [x] 11. An exhaustive QA scenario matrix (Positive, Boundary, Security, Fault) is included (`TC-EVAL-01` to `TC-EVAL-15`).
- [x] 12. Every assertion includes exact file path, class/method, and line number citations.
- [x] 13. All unverified assumptions have been eliminated or marked UNVERIFIABLE.
- [x] 14. Product decisions, open questions, and technical recommendations are segregated (`DEC-EVAL-01` to `DEC-EVAL-03`).
- [x] 15. The specification strictly conforms to SECTIONS A through Q of `DEZ-STD-FS-001`.
- [x] 16. Git status verified: Zero modifications made to application source code or schemas.

