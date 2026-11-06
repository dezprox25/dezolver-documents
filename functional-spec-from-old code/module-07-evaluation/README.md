# Module 7 — Test Case Evaluation & Scoring Engine

## 1. Module Executive Summary

Module 7 governs the algorithmic evaluation, verdict aggregation, and scoring determination mechanisms of the Dezolver platform. While Module 6 manages the ingress, queueing, and dispatch of code to Judge0 CE, Module 7 defines how test case results are evaluated, how execution outputs are compared against baselines, how multi-tier verdicts are resolved across test suites, and how scores are calculated and propagated across platform modalities (Assessments, Practice Rooms, Competitions, and Labs).

A foundational forensic revelation is that **the legacy codebase does not contain a custom Node.js Diff Engine, Custom Checker harness, or floating-point tolerance comparator**. Instead, stdout-to-expected-output diffing is **100% delegated to the Judge0 CE runner**. The evaluation logic within Dezolver resides in the `computeVerdict()` engine inside [`src/worker/consumers/code-execution.consumer.ts`](file:///E:/Dezolver%20project/dezlovers-main/Final%20product/dezlovers-main(cuurent%20update)/src/worker/consumers/code-execution.consumer.ts#L39-L73), which computes weighted proportional scores and applies hardcoded verdict priority rules ($TLE > MLE > CE > RE > WA$).

---

## 2. Directory Layout & Document Navigation

```
docs/functional-spec/module-07-evaluation/
├── README.md               # Executive summary, forensic findings, and navigational guide (this file)
└── SPECIFICATION.md        # Complete 17-Section (A–Q) legacy functional reconstruction
```

- **Detailed Functional Specification:** [`SPECIFICATION.md`](./SPECIFICATION.md)
- **Authoring Standard:** [`FUNCTIONAL-SPECIFICATION-STANDARD.md`](../../FUNCTIONAL-SPECIFICATION-STANDARD.md)
- **Master Index & Registry:** [`INDEX.md`](../INDEX.md)

---

## 3. Evaluation Scope Analysis Across Modalities

| Modality | Implementation Status | Technical Mechanism in Codebase |
|---|---|---|
| **Assessment** | `CONFIRMED LEGACY IMPLEMENTATION` | Fully wired via `assessment.assessments`, `assessment.submissions`, and `assessment.judge_runs`. Evaluates all test cases (sample + hidden) and calculates weighted 0–100% scores. |
| **Practice (Rooms)** | `PARTIALLY IMPLEMENTED` | Coding problems inside curriculum rooms are evaluated as single-question assessments. However, `SubmissionAcceptedConsumer` listens for `SubmissionAccepted` which is never emitted by `CodeExecutionConsumer`, breaking room auto-completion. |
| **Contest / Competition** | `CONFIRMED LEGACY IMPLEMENTATION` | [`SubmissionGradedConsumer`](file:///E:/Dezolver%20project/dezlovers-main/Final%20product/dezlovers-main(cuurent%20update)/src/events/consumers/submission-graded.consumer.ts) inspects graded submissions with `metadata.eventId`. Upserts `events.event_results` and updates Redis ZSET `leaderboard:event:${eventId}` with live rank broadcasting. |
| **Lab & Lab Scoring** | `NOT IMPLEMENTED` | **ZERO CODE EXISTENCE.** No `LabAssignment`, `LabSubmission`, `GroupLab`, or lab scoring services exist in the database or source code. |

---

## 4. Source-Document vs. Codebase Reality Reconciliation

| Conceptual / Documented Requirement | Actual Legacy Codebase Implementation Reality | Status |
|---|---|---|
| **Custom Diff Engine** | **DOES NOT EXIST.** Dezolver sends `expected_output` directly to Judge0 CE; Judge0 performs string comparison and returns status codes. | `NOT IMPLEMENTED` |
| **Custom Checkers (e.g. `testlib.h`)** | **DOES NOT EXIST.** No custom checker scripts, comparator binaries, or interactive test harnesses exist. | `NOT IMPLEMENTED` |
| **Floating-Point Tolerances ($\epsilon$)** | **DOES NOT EXIST.** Output comparison is exact string matching. Any precision divergence results in status 4 (`Wrong Answer`). | `NOT IMPLEMENTED` |
| **Whitespace Normalization** | **DOES NOT EXIST in backend.** Handled strictly by Judge0's native execution flags without pre-processing. | `NOT IMPLEMENTED` |
| **Weighted Scoring Model** | **IMPLEMENTED.** $\text{Score} = \text{round}\left(\frac{\sum_{\text{Passed}} w_i}{\sum_{\text{Total}} w_i} \times 100\right)$ in `computeVerdict`. | `CONFIRMED` |
| **Verdict Precedence** | **IMPLEMENTED.** Strict priority order: $\text{TLE} (5) > \text{MLE} (12) > \text{CE} (6) > \text{RE} (7\text{--}11) > \text{WA} (4)$. | `CONFIRMED` |
| **Partial Credit Verdict** | **IMPLEMENTED.** If $0 < \text{passed} < N$ and overall verdict is accepted, verdict transitions to `'partial'`. | `CONFIRMED` |
| **Sample vs Hidden Masking** | **IMPLEMENTED.** `actualOutput` and `expectedOutput` in `testCaseResults` are populated **only** for test cases where `isSample = true`. | `CONFIRMED` |

---

## 5. Primary Source Code References

### Backend
- **Verdict & Scoring Logic:** [`src/worker/consumers/code-execution.consumer.ts`](file:///E:/Dezolver%20project/dezlovers-main/Final%20product/dezlovers-main(cuurent%20update)/src/worker/consumers/code-execution.consumer.ts#L39-L73) (`computeVerdict`)
- **Judge0 Status & Response Mapping:** [`src/assessment/judge/judge0.client.ts`](file:///E:/Dezolver%20project/dezlovers-main/Final%20product/dezlovers-main(cuurent%20update)/src/assessment/judge/judge0.client.ts#L79-L98)
- **Contest Leaderboard Consumer:** [`src/events/consumers/submission-graded.consumer.ts`](file:///E:/Dezolver%20project/dezlovers-main/Final%20product/dezlovers-main(cuurent%20update)/src/events/consumers/submission-graded.consumer.ts)
- **Curriculum Room Progress Consumer:** [`src/paths/consumers/submission-accepted.consumer.ts`](file:///E:/Dezolver%20project/dezlovers-main/Final%20product/dezlovers-main(cuurent%20update)/src/paths/consumers/submission-accepted.consumer.ts)
- **Submissions Repository:** [`src/assessment/repositories/submissions.repository.ts`](file:///E:/Dezolver%20project/dezlovers-main/Final%20product/dezlovers-main(cuurent%20update)/src/assessment/repositories/submissions.repository.ts)
- **Prisma Schemas:**
  - Assessment: [`prisma/assessment/schema.prisma`](file:///E:/Dezolver%20project/dezlovers-main/Final%20product/dezlovers-main(cuurent%20update)/prisma/assessment/schema.prisma) (`Submission`, `JudgeRun`)
  - Content: [`prisma/content/schema.prisma`](file:///E:/Dezolver%20project/dezlovers-main/Final%20product/dezlovers-main(cuurent%20update)/prisma/content/schema.prisma) (`ProblemTestCase`)
  - Events: [`prisma/events/schema.prisma`](file:///E:/Dezolver%20project/dezlovers-main/Final%20product/dezlovers-main(cuurent%20update)/prisma/events/schema.prisma) (`EventResult`)

### Frontend
- **Verdict Visuals:** [`frontend/src/components/assessment/VerdictBadge.tsx`](file:///E:/Dezolver%20project/dezlovers-main/Final%20product/frontend/src/components/assessment/VerdictBadge.tsx)
- **Verdict Summary Panel:** [`frontend/src/components/assessment/VerdictPanel.tsx`](file:///E:/Dezolver%20project/dezlovers-main/Final%20product/frontend/src/components/assessment/VerdictPanel.tsx)
- **Test Case Results Inspector:** [`frontend/src/components/assessment/JudgeRunCard.tsx`](file:///E:/Dezolver%20project/dezlovers-main/Final%20product/frontend/src/components/assessment/JudgeRunCard.tsx)
- **Submission Detail View:** [`frontend/src/pages/submissions/SubmissionDetailPage.tsx`](file:///E:/Dezolver%20project/dezlovers-main/Final%20product/frontend/src/pages/submissions/SubmissionDetailPage.tsx)

