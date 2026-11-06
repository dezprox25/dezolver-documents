# Module 9 — Assessments

## 1. Module Executive Summary

Module 9 governs the creation, scheduling, timing, attempt governance, anti-cheat telemetry collection, and student testing workspaces for academic coding assessments on the Dezolver platform.

A forensic investigation of the legacy codebase reveals that Module 9 was implemented as a **lightweight coding execution proxy and anti-cheat signal collector**, rather than a full-fledged academic assessment management system.

Key implementation realities uncovered from source code:
1. **Universal Access / Zero Audience Targeting:** Table `assessment.assessments` contains **no `tenant_id`, `cohort_id`, `group_id`, or `section_id` columns**. All assessments are globally retrievable by any authenticated user holding the `read:cohort` permission.
2. **Missing Lifecycle State in Database:** Table `assessment.assessments` possesses **no `status` column**. While frontend TypeScript types define `'draft' | 'published' | 'archived'`, the frontend API client explicitly strips `status` before dispatching updates, and backend endpoints ignore it. Assessments are active immediately upon creation.
3. **Client-Side Timer Fallacy & CPU Limit Substitution:** The assessment countdown timer is maintained exclusively in student browser `localStorage`. In the backend, `timeLimitMinutes` is not used to enforce a submission deadline; instead, it is multiplied by 60 and passed as the per-test-case `cpuTimeLimit` (in seconds) to the Judge0 sandbox runner!
4. **Single-Problem Architecture:** An assessment references at most one problem via scalar `problemId`. There is no multi-problem join table.
5. **Coding Only — Quizzes Unsupported in Backend:** While the frontend defines MCQ and quiz interfaces, the backend `SubmissionService.submitCode` explicitly rejects non-coding submissions with `400 BadRequestException({ code: 'wrong_submission_kind' })`, and no quiz evaluation endpoint exists.
6. **Critical Faculty RBAC Defect:** `AssessmentsController.create` is guarded with `@RequirePermission('create:content')`, which only `content_manager` and `platform_admin` possess; `faculty` possesses `create:assessment`, inadvertently blocking faculty from creating assessments via the API.
7. **Broken Anti-Cheat Scheduled Cron:** `AntiCheatService.analyzeRecentSubmissions` crashes silently on nonexistent columns `createdAt`, `flagged`, and `metadata`, catching errors and returning `[]`.

---

## 2. Directory Layout & Document Navigation

```
docs/functional-spec/module-09-assessments/
├── README.md               # Executive summary, forensic findings, and navigational guide (this file)
└── SPECIFICATION.md        # Complete 17-Section (A–Q) legacy functional reconstruction & 40-question answers
```

- **Detailed Functional Specification:** [`SPECIFICATION.md`](./SPECIFICATION.md)
- **Authoring Standard:** [`FUNCTIONAL-SPECIFICATION-STANDARD.md`](../../FUNCTIONAL-SPECIFICATION-STANDARD.md)
- **Master Index & Registry:** [`INDEX.md`](../INDEX.md)

---

## 3. Key Forensic Discoveries & Architecture Summary

| Component / Subsystem | Conceptual Specification | Actual Legacy Implementation Reality | Forensic Status |
|---|---|---|:---:|
| **Audience Assignment** | Assigned to Groups, Sections, Departments | No assignment entities exist; all assessments are open to any authenticated student with `read:cohort`. | `NOT FOUND` |
| **`groupId` Field** | Scopes assessment access to a student group | Zero instances of `groupId` exist in backend schemas or frontend code. | `NOT FOUND` |
| **Lifecycle States** | `DRAFT`, `PUBLISHED`, `ARCHIVED`, `SCHEDULED` | Backend has no status column; assessments are immediately active on creation. Frontend strips `status` on update. | `CONFLICTING` |
| **Timer Enforcement** | Server-side start/end enforcement with auto-submit | Purely client-side countdown in `localStorage`; no server submission cutoff. Backend substitutes duration for Judge0 CPU limit. | `PARTIALLY IMPLEMENTED` |
| **Attempts Policy** | Configurable attempt limit with Best/Latest score rule | `maxAttempts` enforced in DB; no final score selection logic exists (append-only log). | `PARTIALLY IMPLEMENTED` |
| **Problem Association** | Multi-problem assessments with weights & ordering | Scalar `problemId` references at most 1 problem; weights exist only at test-case level. | `PARTIALLY IMPLEMENTED` |
| **Scoring Engine** | Comprehensive rubric with `passingPercentage` | Proportional test-case weights via `computeVerdict`; `passingPercentage` is absent. | `PARTIALLY IMPLEMENTED` |
| **Anti-Cheating** | Browser tracking, fullscreen lock, plagiarism check | Tab blur/paste tracking sent in payload; cron checks score, but schema lacks `flagged` col. | `PARTIALLY IMPLEMENTED` |
| **Results Release** | Embargoed until staff release or exam window close | Immediate real-time Redis/WebSocket broadcast upon grading; sample outputs visible. | `CONFIRMED` |
| **Staff Score Overrides** | HOD/Manager manual score editing with audit log | Manual score editing does not exist; only anti-cheat flag reviews can be recorded. | `NOT FOUND` |

---

## 4. Primary Source Code References

### Backend
- **Prisma Schema:** [`prisma/assessment/schema.prisma`](file:///E:/Dezolver%20project/dezlovers-main/Final%20product/dezlovers-main(cuurent%20update)/prisma/assessment/schema.prisma) (`Assessment`, `Submission`, `JudgeRun`)
- **Controller:** [`src/assessment/controllers/assessments.controller.ts`](file:///E:/Dezolver%20project/dezlovers-main/Final%20product/dezlovers-main(cuurent%20update)/src/assessment/controllers/assessments.controller.ts)
- **Submissions Controller:** [`src/assessment/controllers/submissions.controller.ts`](file:///E:/Dezolver%20project/dezlovers-main/Final%20product/dezlovers-main(cuurent%20update)/src/assessment/controllers/submissions.controller.ts)
- **Repository:** [`src/assessment/repositories/assessments.repository.ts`](file:///E:/Dezolver%20project/dezlovers-main/Final%20product/dezlovers-main(cuurent%20update)/src/assessment/repositories/assessments.repository.ts)
- **Service:** [`src/assessment/services/submission.service.ts`](file:///E:/Dezolver%20project/dezlovers-main/Final%20product/dezlovers-main(cuurent%20update)/src/assessment/services/submission.service.ts)
- **Anti-Cheat Service:** [`src/assessment/services/anti-cheat.service.ts`](file:///E:/Dezolver%20project/dezlovers-main/Final%20product/dezlovers-main(cuurent%20update)/src/assessment/services/anti-cheat.service.ts)
- **RBAC Matrix:** [`src/shared/auth/permission.guard.ts`](file:///E:/Dezolver%20project/dezlovers-main/Final%20product/dezlovers-main(cuurent%20update)/src/shared/auth/permission.guard.ts#L11-L38)

### Frontend
- **Assessment Pages:** [`frontend/src/pages/assessments/`](file:///E:/Dezolver%20project/dezlovers-main/Final%20product/frontend/src/pages/assessments/)
  - `AssessmentsPage.tsx`, `AssessmentDetailPage.tsx`, `AssessmentCreatePage.tsx`, `AssessmentWorkspacePage.tsx`, `AssessmentAnalyticsPage.tsx`
- **Assessment Components:** [`frontend/src/components/assessment/`](file:///E:/Dezolver%20project/dezlovers-main/Final%20product/frontend/src/components/assessment/)
  - `AssessmentTimer.tsx`, `CodeEditor.tsx`, `ProblemPanel.tsx`, `VerdictPanel.tsx`
- **API Service:** [`frontend/src/services/api/assessment.service.ts`](file:///E:/Dezolver%20project/dezlovers-main/Final%20product/frontend/src/services/api/assessment.service.ts)
- **Hooks:** [`frontend/src/hooks/useAssessments.ts`](file:///E:/Dezolver%20project/dezlovers-main/Final%20product/frontend/src/hooks/useAssessments.ts)

