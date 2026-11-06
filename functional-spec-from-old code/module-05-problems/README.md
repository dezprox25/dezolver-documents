# Module 5 — Problem Management

## 1. Module Executive Summary

Module 5 provides the coding challenge and technical assessment question repository for the Dezolver platform. It manages the lifecycle of coding problems, Markdown problem statements, input/output specifications, mathematical constraints, difficulty tiers, topic tagging, execution parameters (CPU time and RAM limits), and associated verification test cases (both public sample cases and hidden evaluation cases).

A foundational forensic discovery is that **Problems exist exclusively as a Global / Platform-Wide Catalog** within the `content` database schema. There is no `tenant_id` column on `content.problems`, meaning institutional colleges and faculty cannot author private, tenant-isolated problems in the legacy codebase. Furthermore, **the backend completely lacks Problem Update (`PATCH`/`PUT`), Problem Deletion (`DELETE`), and Test Case Modification endpoints**, rendering problems immutable once drafted.

---

## 2. Directory Layout & Document Navigation

```
docs/functional-spec/module-05-problems/
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
| **Database Schema** | `content` schema ([`prisma/content/schema.prisma`](file:///E:/Dezolver%20project/dezlovers-main/Final%20product/dezlovers-main(cuurent%20update)/prisma/content/schema.prisma)): `Problem` (`content.problems`) and `ProblemTestCase` (`content.problem_test_cases`). |
| **Catalog Scope** | **GLOBAL ONLY.** No `tenant_id` exists on `Problem`. All problems are platform-wide and authored by global content managers or platform admins. |
| **Test Case Secrecy** | Strict API-level masking. [`ProblemsRepository.findBySlug`](file:///E:/Dezolver%20project/dezlovers-main/Final%20product/dezlovers-main(cuurent%20update)/src/content/repositories/problems.repository.ts#L39-L49) filters `testCases: { where: { isSample: true } }`. Hidden test cases are never leaked to public endpoints. |
| **Publication Invariant** | A problem cannot transition to `published` unless it possesses at least one test case (`testCases.length === 0 -> throw BadRequestException({ code: 'no_test_cases' })`). |
| **Execution Limits** | Explicit resource caps: `timeLimitMs` (default 2000 ms, min 100, max 60000) and `memoryLimitMb` (default 256 MB, min 8, max 512). |
| **Language Restrictions** | String array `allowedLanguages` stored per problem, restricting acceptable Judge0 compilers (e.g. `['python', 'java', 'cpp']`). |
| **Immutable Design** | **CRITICAL LEGACY GAP.** Zero endpoints exist to update or delete problems or test cases after initial creation. |
| **Cross-Content Search** | Full-text search via `POST /search` using PostgreSQL `ILIKE` insensitive matching over titles and statements. |
| **Frontend Suite** | Complete problem catalog, detail view with Markdown/KaTeX statement rendering, and multi-step creation form ([`frontend/src/pages/content/problems/`](file:///E:/Dezolver%20project/dezlovers-main/Final%20product/frontend/src/pages/content/problems/)). |

---

## 4. Primary Source Code References

### Backend
- **Prisma Schema:** [`prisma/content/schema.prisma`](file:///E:/Dezolver%20project/dezlovers-main/Final%20product/dezlovers-main(cuurent%20update)/prisma/content/schema.prisma) (`Problem`, `ProblemTestCase`)
- **Controller:** [`src/content/controllers/problems.controller.ts`](file:///E:/Dezolver%20project/dezlovers-main/Final%20product/dezlovers-main(cuurent%20update)/src/content/controllers/problems.controller.ts)
- **Repository:** [`src/content/repositories/problems.repository.ts`](file:///E:/Dezolver%20project/dezlovers-main/Final%20product/dezlovers-main(cuurent%20update)/src/content/repositories/problems.repository.ts)
- **Service:** [`src/content/services/content.service.ts`](file:///E:/Dezolver%20project/dezlovers-main/Final%20product/dezlovers-main(cuurent%20update)/src/content/services/content.service.ts#L40-L105)
- **DTOs:** [`src/content/dto/content.dto.ts`](file:///E:/Dezolver%20project/dezlovers-main/Final%20product/dezlovers-main(cuurent%20update)/src/content/dto/content.dto.ts#L15-L100)
- **Search Controller:** [`src/content/controllers/search.controller.ts`](file:///E:/Dezolver%20project/dezlovers-main/Final%20product/dezlovers-main(cuurent%20update)/src/content/controllers/search.controller.ts)

### Frontend
- **Problem Pages:** [`frontend/src/pages/content/problems/`](file:///E:/Dezolver%20project/dezlovers-main/Final%20product/frontend/src/pages/content/problems/)
  - [`ProblemsPage.tsx`](file:///E:/Dezolver%20project/dezlovers-main/Final%20product/frontend/src/pages/content/problems/ProblemsPage.tsx)
  - [`ProblemDetailPage.tsx`](file:///E:/Dezolver%20project/dezlovers-main/Final%20product/frontend/src/pages/content/problems/ProblemDetailPage.tsx)
  - [`ProblemCreatePage.tsx`](file:///E:/Dezolver%20project/dezlovers-main/Final%20product/frontend/src/pages/content/problems/ProblemCreatePage.tsx)
- **Hooks:** [`frontend/src/hooks/useProblems.ts`](file:///E:/Dezolver%20project/dezlovers-main/Final%20product/frontend/src/hooks/useProblems.ts)
- **Assessment Components:** [`frontend/src/components/assessment/ProblemPanel.tsx`](file:///E:/Dezolver%20project/dezlovers-main/Final%20product/frontend/src/components/assessment/ProblemPanel.tsx), [`ProblemHeader.tsx`](file:///E:/Dezolver%20project/dezlovers-main/Final%20product/frontend/src/components/assessment/ProblemHeader.tsx), [`ProblemExamples.tsx`](file:///E:/Dezolver%20project/dezlovers-main/Final%20product/frontend/src/components/assessment/ProblemExamples.tsx)
- **API Service:** [`frontend/src/services/api/content.service.ts`](file:///E:/Dezolver%20project/dezlovers-main/Final%20product/frontend/src/services/api/content.service.ts#L155-L191)

