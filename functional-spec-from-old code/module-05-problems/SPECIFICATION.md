# Module 5 — Problem Management Specification

**Standard Reference:** `DEZ-STD-FS-001`  
**Document Identifier:** `DEZ-SPEC-MOD-005`  
**Status:** `CONFIRMED LEGACY IMPLEMENTATION / RECONSTRUCTED`  
**Database Schema:** `content` ([`prisma/content/schema.prisma`](file:///E:/Dezolver%20project/dezlovers-main/Final%20product/dezlovers-main(cuurent%20update)/prisma/content/schema.prisma))  
**Module Directory:** `src/content/`  
**Frontend Directory:** `frontend/src/pages/content/problems/`, `frontend/src/components/assessment/`  

---

## SECTION A: MODULE OVERVIEW

### A.1 Purpose & Objectives
Module 5 (Problem Management) serves as the centralized technical challenge and algorithmic coding question repository of the Dezolver platform. It provides the domain abstractions, storage models, input/output specifications, execution resource limits, and test case data required to define, browse, and execute coding challenges across platform assessments, student practice rooms, and competitive programming contests.

The core business objectives implemented by this module are:
1. **Standardized Problem Modeling:** Defining programming challenges with structured Markdown statements, formatted input/output descriptions, computational constraints, topic taxonomies, and difficulty tiers (`easy`, `medium`, `hard`).
2. **Deterministic Execution Parameterization:** Associating precise CPU execution time limits (`time_limit_ms`) and RAM consumption caps (`memory_limit_mb`) to govern code evaluation environments (Judge0 CE).
3. **Multi-Tier Test Case Management:** Partitioning test cases into public sample cases (`is_sample = true`) exposed to students for local verification, and hidden evaluation cases (`is_sample = false`) strictly sequestered for backend grading.
4. **Content Publication Integrity:** Enforcing strict publication state invariants requiring at least one verified test case before a problem can transition from draft to published status.
5. **Universal Content Discoverability:** Providing cursor-paginated problem catalogs and full-text substring search across platform-wide challenges.

### A.2 Scope & Boundaries
- **In-Scope (Module 5):**
  - Problem draft authoring and metadata initialization (`POST /problems`).
  - Automatic URL slug generation and explicit slug assignment.
  - Test case ingestion and weighting (`POST /problems/:id/test-cases`).
  - Publication gating and state transition (`POST /problems/:id/publish`).
  - Public problem catalog exploration with cursor pagination (`GET /problems`).
  - Public problem statement retrieval with sample-only test case masking (`GET /problems/:slug`).
  - Cross-content full-text search integration (`POST /search`).
  - Problem UI views (Catalog list, Detail tabs, Multi-step creation form, Student assessment panel).
- **Out-of-Scope (Delegated to Adjacent Modules):**
  - **Code Execution & Sandboxing:** Isolated compiler invocation and Judge0 runner dispatch are owned by **Module 6 (Code Submission & Judge0 Integration)**.
  - **Test Case Evaluation & Verdict Calculation:** Batch execution of hidden test cases, runtime resource measurement, diff generation, and verdict scoring are owned by **Module 7 (Test Case Evaluation & Scoring Engine)**.
  - **Assessment Exam Orchestration:** Grouping problems into exams, timer cutoffs, anti-cheat signaling, and attempt limits are owned by **Module 9 (Assessments)**.
  - **Contest & Leaderboard Management:** Contest schedules, registration, penalty scoring, and live rank lists are owned by **Module 11 (Contests)**.
  - **Room Curriculum Stitching:** Sequential room paths and learning tracks embedding problems are owned by **Module 4 (Course & Curriculum Management)**.

### A.3 Actors & Personas
- **Content Manager (`content_manager`):** Authorized to author problem drafts, ingest test cases, configure compiler language constraints, and publish challenges globally.
- **Platform Administrator (`platform_admin`):** Superuser with unrestricted wildcard access (`*`) across all problem authoring, test case configuration, and status transitions.
- **Platform Moderator (`platform_moderator`):** Listed in service-level manager roles, though restricted by controller permissions in legacy code.
- **Student / Candidate (`student`):** Public consumer who browses published problems, views Markdown statements, inspects sample test cases, and submits solutions.
- **Assessment Engine (Machine Actor):** Automated system that fetches problem execution limits (`timeLimitMs`, `memoryLimitMb`, `allowedLanguages`) and full test case suites for headless grading.

### A.4 Dependencies & Cross-Module Relationships
```mermaid
flowchart TD
    subgraph Module5["Module 5: Problem Management"]
        ProbCtrl["ProblemsController<br/>(src/content/controllers/problems.controller.ts)"]
        SrchCtrl["SearchController<br/>(src/content/controllers/search.controller.ts)"]
        ContentSvc["ContentService<br/>(src/content/services/content.service.ts)"]
        ProbRepo["ProblemsRepository<br/>(src/content/repositories/problems.repository.ts)"]
        DB_Prob[("content.problems")]
        DB_TC[("content.problem_test_cases")]
    end

    subgraph Module6_7["Module 6 & 7: Execution & Evaluation"]
        SubSvc["SubmissionService<br/>(Module 6)"]
        EvalWorker["EvaluationWorker<br/>(Module 7)"]
    end

    subgraph Module9["Module 9: Assessments"]
        AssessModel[("assessment.assessments<br/>(problem_id FK)")]
    end

    ProbCtrl --> ContentSvc
    SrchCtrl --> ContentSvc
    ContentSvc --> ProbRepo
    ProbRepo --> DB_Prob
    ProbRepo --> DB_TC

    DB_Prob -.->|"Foreign Reference"| AssessModel
    DB_Prob -.->|"Fetch Limits"| SubSvc
    DB_TC -.->|"Fetch Hidden Test Cases"| EvalWorker
```

### A.5 Domain Terminology & Glossary
- **Problem (`Problem`):** An algorithmic or programming challenge entity stored in `content.problems` with metadata, Markdown statement, and execution boundaries.
- **ProblemTestCase (`ProblemTestCase`):** An input-output pair associated with a problem in `content.problem_test_cases`, carrying an evaluation weight and a sample visibility flag.
- **Sample Test Case:** A test case with `isSample = true`, visible to all students via public APIs to demonstrate expected input and output formatting.
- **Hidden Test Case:** A test case with `isSample = false`, strictly suppressed from public endpoints, utilized exclusively by background grading workers to score submissions.
- **Slug:** A unique, URL-safe string identifier (e.g., `reverse-a-linked-list`) automatically derived from the problem title or explicitly provided by an author.
- **Time Limit (`timeLimitMs`):** Maximum CPU execution duration (in milliseconds) permitted for a solution before Judge0 terminates execution with `Time Limit Exceeded` (default: 2000 ms).
- **Memory Limit (`memoryLimitMb`):** Maximum RAM allocation (in megabytes) permitted for a solution before Judge0 terminates execution with `Memory Limit Exceeded` (default: 256 MB).
- **Global Content Catalog:** Architecture design where problems belong to the platform at large rather than being isolated to specific institutional tenants.

---

## SECTION B: LEGACY IMPLEMENTATION ARCHITECTURE

### B.1 Architectural Overview Diagram
```mermaid
sequenceDiagram
    autonumber
    actor Author as Content Manager
    participant UI as ProblemCreatePage.tsx
    participant API as ProblemsController
    participant Guard as PermissionGuard
    participant Svc as ContentService
    participant Repo as ProblemsRepository
    participant DB as PostgreSQL (content schema)

    Author->>UI: Fill Title, Statement, Limits, Sample & Hidden Cases
    UI->>API: POST /problems (Title, Markdown, Limits, Languages)
    API->>Guard: Verify JWT & 'create:content' permission
    Guard-->>API: Authorized
    API->>Svc: createProblem(dto, principal)
    Svc->>Svc: toSlug(dto.title)
    Svc->>Repo: create({ ...dto, slug, authorUserId, status: 'draft' })
    Repo->>DB: INSERT INTO content.problems
    DB-->>Repo: Problem Record Created
    Repo-->>Svc: Problem Record
    Svc-->>API: Problem Record
    API-->>UI: HTTP 201 Created (Problem ID & Slug)

    loop For Each Test Case
        UI->>API: POST /problems/:id/test-cases (Input, ExpectedOutput, isSample, weight)
        API->>Guard: Verify 'create:content'
        API->>Svc: addTestCase(id, dto, principal)
        Svc->>Repo: findById(id)
        Repo-->>Svc: Problem (Status: draft)
        Svc->>Repo: addTestCase(data)
        Repo->>DB: INSERT INTO content.problem_test_cases
        DB-->>API: HTTP 201 Created
    end

    Author->>UI: Click "Publish"
    UI->>API: POST /problems/:id/publish
    API->>Guard: Verify 'publish:content'
    API->>Svc: publishProblem(id, principal)
    Svc->>Repo: getTestCases(id, includeHidden=true)
    Repo-->>Svc: Test Cases Count > 0
    Svc->>Repo: updateStatus(id, 'published')
    Repo->>DB: UPDATE content.problems SET status='published', published_at=NOW()
    DB-->>API: HTTP 200 OK
    API-->>UI: Problem Published Successfully
```

### B.2 Backend Architecture
The backend is structured under the NestJS `ContentModule` ([`src/content/content.module.ts`](file:///E:/Dezolver%20project/dezlovers-main/Final%20product/dezlovers-main(cuurent%20update)/src/content/content.module.ts)):
- **Controllers:**
  - `ProblemsController` ([`src/content/controllers/problems.controller.ts`](file:///E:/Dezolver%20project/dezlovers-main/Final%20product/dezlovers-main(cuurent%20update)/src/content/controllers/problems.controller.ts)): Exposes listing, slug retrieval, draft creation, test case ingestion, and publishing endpoints.
  - `SearchController` ([`src/content/controllers/search.controller.ts`](file:///E:/Dezolver%20project/dezlovers-main/Final%20product/dezlovers-main(cuurent%20update)/src/content/controllers/search.controller.ts)): Exposes cross-content full-text search.
- **Service Layer:**
  - `ContentService` ([`src/content/services/content.service.ts`](file:///E:/Dezolver%20project/dezlovers-main/Final%20product/dezlovers-main(cuurent%20update)/src/content/services/content.service.ts#L40-L105)): Implements slugification, role verification, test-case publication invariants, and search aggregation.
- **Data Access Layer:**
  - `ProblemsRepository` ([`src/content/repositories/problems.repository.ts`](file:///E:/Dezolver%20project/dezlovers-main/Final%20product/dezlovers-main(cuurent%20update)/src/content/repositories/problems.repository.ts)): Executes Prisma queries against `prisma.content.problem` and `prisma.content.problemTestCase`.
- **Interceptors & Guards:**
  - `PermissionGuard` ([`src/shared/auth/permission.guard.ts`](file:///E:/Dezolver%20project/dezlovers-main/Final%20product/dezlovers-main(cuurent%20update)/src/shared/auth/permission.guard.ts)): Enforces RBAC permissions (`create:content`, `publish:content`, `read:cohort`).
  - `EnvelopeInterceptor` ([`src/shared/interceptors/envelope.interceptor.ts`](file:///E:/Dezolver%20project/dezlovers-main/Final%20product/dezlovers-main(cuurent%20update)/src/shared/interceptors/envelope.interceptor.ts)): Wraps API responses into standardized `{ data, ... }` envelopes.

### B.3 Frontend Architecture
The client application is built with React 18, React Router v6, TanStack Query v5, React Hook Form, Zod, and Tailwind CSS / shadcn/ui:
- **Routes & Pages:**
  - `/content/problems`: Handled by `ProblemsPage` ([`frontend/src/pages/content/problems/ProblemsPage.tsx`](file:///E:/Dezolver%20project/dezlovers-main/Final%20product/frontend/src/pages/content/problems/ProblemsPage.tsx)). Catalog with debounced search, difficulty filter, status filter, and infinite cursor pagination.
  - `/content/problems/create`: Handled by `ProblemCreatePage` ([`frontend/src/pages/content/problems/ProblemCreatePage.tsx`](file:///E:/Dezolver%20project/dezlovers-main/Final%20product/frontend/src/pages/content/problems/ProblemCreatePage.tsx)). Form with multi-test case authoring, tag pills, and language selectors.
  - `/content/problems/:slug`: Handled by `ProblemDetailPage` ([`frontend/src/pages/content/problems/ProblemDetailPage.tsx`](file:///E:/Dezolver%20project/dezlovers-main/Final%20product/frontend/src/pages/content/problems/ProblemDetailPage.tsx)). Tabs for Statement, Test Cases, and Runtime Configuration.
- **Assessment Components:**
  - `ProblemPanel` ([`frontend/src/components/assessment/ProblemPanel.tsx`](file:///E:/Dezolver%20project/dezlovers-main/Final%20product/frontend/src/components/assessment/ProblemPanel.tsx)): Embedded left-hand problem viewer in student assessment interfaces.
  - `ProblemHeader` ([`frontend/src/components/assessment/ProblemHeader.tsx`](file:///E:/Dezolver%20project/dezlovers-main/Final%20product/frontend/src/components/assessment/ProblemHeader.tsx)): Title, difficulty badge, resource limits, and topic tags.
  - `ProblemExamples` ([`frontend/src/components/assessment/ProblemExamples.tsx`](file:///E:/Dezolver%20project/dezlovers-main/Final%20product/frontend/src/components/assessment/ProblemExamples.tsx)): Side-by-side Input/Output code blocks for sample cases.
- **Data Fetching Hooks:**
  - `useProblems`, `useProblem`, `useCreateProblem`, `useAddTestCase`, `usePublishProblem` ([`frontend/src/hooks/useProblems.ts`](file:///E:/Dezolver%20project/dezlovers-main/Final%20product/frontend/src/hooks/useProblems.ts)).

### B.4 Database Models & Schema Boundaries
The entities reside exclusively in the `content` PostgreSQL schema ([`prisma/content/schema.prisma`](file:///E:/Dezolver%20project/dezlovers-main/Final%20product/dezlovers-main(cuurent%20update)/prisma/content/schema.prisma)):
- Table `content.problems`: Core problem definitions.
- Table `content.problem_test_cases`: Test cases linked by `problem_id` with `onDelete: Cascade`.
- Cross-schema linkage: `assessment.assessments.problem_id` ([`prisma/assessment/schema.prisma`](file:///E:/Dezolver%20project/dezlovers-main/Final%20product/dezlovers-main(cuurent%20update)/prisma/assessment/schema.prisma#L20)) references `content.problems.id` logically across schema boundaries.

---

## SECTION C: DATABASE SPECIFICATION

### C.1 Entity Definition Tables

#### C.1.1 Table `content.problems`
Mapped from model `Problem` ([`prisma/content/schema.prisma:15-40`](file:///E:/Dezolver%20project/dezlovers-main/Final%20product/dezlovers-main(cuurent%20update)/prisma/content/schema.prisma#L15-L40)).

| Column Name | Physical Data Type | Nullable | Default Value | PK | FK Constraint | Unique | Index | Description & Runtime Usage |
|---|---|:---:|---|:---:|---|:---:|:---:|---|
| `id` | `UUID` | `NO` | `gen_random_uuid()` | `YES` | — | `YES` | `PRIMARY` | Primary problem UUID identifier. |
| `slug` | `VARCHAR(255)` | `NO` | — | `NO` | — | `YES` | `problems_slug_key` | URL-safe slug for routing and SEO. |
| `title` | `VARCHAR(255)` | `NO` | — | `NO` | — | `NO` | — | Human-readable problem title. |
| `statement_md` | `TEXT` | `NO` | — | `NO` | — | `NO` | — | Markdown problem description and prompt. |
| `input_format_md` | `TEXT` | `YES` | `NULL` | `NO` | — | `NO` | — | Markdown input structure and data types. |
| `output_format_md` | `TEXT` | `YES` | `NULL` | `NO` | — | `NO` | — | Markdown output specification. |
| `constraints_md` | `TEXT` | `YES` | `NULL` | `NO` | — | `NO` | — | Mathematical constraints (e.g., $1 \le N \le 10^5$). |
| `difficulty` | `VARCHAR(20)` | `NO` | `'medium'` | `NO` | — | `NO` | Composite | Difficulty tier (`easy`, `medium`, `hard`). |
| `topics` | `VARCHAR(64)[]` | `NO` | `ARRAY[]` | `NO` | — | `NO` | — | Categorical taxonomy tags (e.g., `dp`, `graphs`). |
| `companies` | `VARCHAR(64)[]` | `NO` | `ARRAY[]` | `NO` | — | `NO` | — | Historical company association tags. |
| `allowed_languages` | `TEXT[]` | `NO` | `ARRAY[]` | `NO` | — | `NO` | — | Compiler language whitelist (e.g., `['python', 'cpp']`). |
| `time_limit_ms` | `INTEGER` | `NO` | `2000` | `NO` | — | `NO` | — | Execution time limit in milliseconds. |
| `memory_limit_mb` | `INTEGER` | `NO` | `256` | `NO` | — | `NO` | — | Execution memory limit in megabytes. |
| `status` | `VARCHAR(20)` | `NO` | `'draft'` | `NO` | — | `NO` | Composite | Problem lifecycle status (`draft`, `published`). |
| `author_user_id` | `UUID` | `YES` | `NULL` | `NO` | — | `NO` | — | UUID of creating user from identity schema. |
| `published_at` | `TIMESTAMPTZ` | `YES` | `NULL` | `NO` | — | `NO` | — | Timestamp of publication transition. |
| `created_at` | `TIMESTAMPTZ` | `NO` | `now()` | `NO` | — | `NO` | — | Creation timestamp. |
| `updated_at` | `TIMESTAMPTZ` | `NO` | Auto-updated | `NO` | — | `NO` | — | Last modification timestamp. |

#### C.1.2 Table `content.problem_test_cases`
Mapped from model `ProblemTestCase` ([`prisma/content/schema.prisma:42-55`](file:///E:/Dezolver%20project/dezlovers-main/Final%20product/dezlovers-main(cuurent%20update)/prisma/content/schema.prisma#L42-L55)).

| Column Name | Physical Data Type | Nullable | Default Value | PK | FK Constraint | Unique | Index | Description & Runtime Usage |
|---|---|:---:|---|:---:|---|:---:|:---:|---|
| `id` | `UUID` | `NO` | `gen_random_uuid()` | `YES` | — | `YES` | `PRIMARY` | Primary test case UUID identifier. |
| `problem_id` | `UUID` | `NO` | — | `NO` | `content.problems(id) ON DELETE CASCADE` | `NO` | Composite | Parent problem reference. |
| `input` | `TEXT` | `NO` | — | `NO` | — | `NO` | — | Raw test case input delivered to stdin. |
| `expected_output` | `TEXT` | `NO` | — | `NO` | — | `NO` | — | Expected stdout baseline for evaluation. |
| `is_sample` | `BOOLEAN` | `NO` | `false` | `NO` | — | `NO` | Composite | Visibility flag: `true`=public, `false`=hidden. |
| `weight` | `INTEGER` | `NO` | `1` | `NO` | — | `NO` | — | Evaluation scoring weight (1–100). |

### C.2 Constraints, Relationships & Cascade Behaviors
- **Foreign Key Cascade:** `content.problem_test_cases.problem_id` enforces `ON DELETE CASCADE`. If a problem record is deleted, all associated test cases are automatically purged by PostgreSQL.
- **Unique Constraints:**
  - `content.problems(slug)`: Guarantees global uniqueness of URL slugs.
- **Indexes:**
  - `content.problems`: `@@index([difficulty, status])` optimizes filtered catalog queries.
  - `content.problem_test_cases`: `@@index([problem_id, isSample])` optimizes sample test case filtering during public problem fetches.

### C.3 Soft-Delete & Data Retention Mechanics
- **No Soft Delete:** Neither `content.problems` nor `content.problem_test_cases` possesses `deleted_at` or `is_deleted` columns.
- **Immutable Deletion Policy:** The backend exposes no HTTP endpoint or service method to delete problems or test cases. Records remain permanently in the database once inserted.

---

## SECTION D: COMPLETE FUNCTIONAL INVENTORY

### D.1 Functional Items Catalog

#### FN-PRB-001: List Problems
- **Status:** `![Status: Confirmed](https://img.shields.io/badge/Status-Confirmed_Legacy-brightgreen)`
- **1. Trigger:** HTTP `GET /problems`
- **2. Actor:** Student, Anonymous Candidate, Faculty, Administrator (Public endpoint).
- **3. Preconditions:** None.
- **4. Input Fields & Payload:**
  - Query parameters: `difficulty` (optional string), `q` (optional title substring query), `limit` (optional string/number, default: 20), `cursor` (optional base64 encoded cursor).
- **5. Input Validation Rules:** Query parsing parses `limit` to integer; `decodeCursor` decodes base64 payload.
- **6. Execution & Processing Logic:**
  - Calls `ContentService.listProblems` ([`src/content/services/content.service.ts:42-62`](file:///E:/Dezolver%20project/dezlovers-main/Final%20product/dezlovers-main(cuurent%20update)/src/content/services/content.service.ts#L42-L62)).
  - Repository defaults `status: 'published'` unless explicitly filtered by internal callers ([`src/content/repositories/problems.repository.ts:20`](file:///E:/Dezolver%20project/dezlovers-main/Final%20product/dezlovers-main(cuurent%20update)/src/content/repositories/problems.repository.ts#L20)).
  - Queries `take: limit + 1` ordered by `createdAt: 'desc'`.
  - Computes `hasMore = items.length > limit`. If true, slices to `limit` and encodes next cursor from `page[page.length - 1].id`.
- **7. Business Rules Enforced:** Hidden draft problems are excluded from standard catalog responses.
- **8. Database Operations:** `this.prisma.content.problem.findMany(...)` selecting `id, slug, title, difficulty, topics, companies, status, createdAt`.
- **9. External Service Calls:** None.
- **10. Authorization & Security Checks:** Decorated with `@Public()`. No JWT or roles required.
- **11. Success Response & Payload:** HTTP 200 OK with `{ items: ProblemSummary[], nextCursor: string | null }`.
- **12. Failure Responses & Error Codes:** None under normal conditions.
- **13. Side Effects & Emitted Events:** None.
- **14. Postconditions & State Changes:** None (Read-only).
- **15. Related Functions:** `FN-PRB-002`, `FN-PRB-006`.
- **16. Source Code Evidence:** [`src/content/controllers/problems.controller.ts:23-39`](file:///E:/Dezolver%20project/dezlovers-main/Final%20product/dezlovers-main(cuurent%20update)/src/content/controllers/problems.controller.ts#L23-L39), [`src/content/services/content.service.ts:42-62`](file:///E:/Dezolver%20project/dezlovers-main/Final%20product/dezlovers-main(cuurent%20update)/src/content/services/content.service.ts#L42-L62), [`src/content/repositories/problems.repository.ts:8-37`](file:///E:/Dezolver%20project/dezlovers-main/Final%20product/dezlovers-main(cuurent%20update)/src/content/repositories/problems.repository.ts#L8-L37).
- **17. Edge Cases Handled:** Invalid base64 cursor gracefully ignored; empty search returns empty array.

---

#### FN-PRB-002: Get Problem by Slug
- **Status:** `![Status: Confirmed](https://img.shields.io/badge/Status-Confirmed_Legacy-brightgreen)`
- **1. Trigger:** HTTP `GET /problems/:slug`
- **2. Actor:** Student, Anonymous Candidate, Faculty, Administrator (Public endpoint).
- **3. Preconditions:** Problem with the specified slug must exist.
- **4. Input Fields & Payload:**
  - Path parameter: `slug` (`string`, required).
- **5. Input Validation Rules:** Non-empty string.
- **6. Execution & Processing Logic:**
  - Calls `ContentService.getProblemBySlug(slug)` ([`src/content/services/content.service.ts:64-68`](file:///E:/Dezolver%20project/dezlovers-main/Final%20product/dezlovers-main(cuurent%20update)/src/content/services/content.service.ts#L64-L68)).
  - Calls `ProblemsRepository.findBySlug(slug)` ([`src/content/repositories/problems.repository.ts:39-49`](file:///E:/Dezolver%20project/dezlovers-main/Final%20product/dezlovers-main(cuurent%20update)/src/content/repositories/problems.repository.ts#L39-L49)).
  - Includes test cases with strict filter `where: { isSample: true }`, selecting `id, input, expectedOutput, isSample`.
  - If null, throws `NotFoundException({ code: 'not_found' })`.
- **7. Business Rules Enforced:** **CRITICAL SECURITY INVARIANT:** Hidden test cases (`isSample = false`) are permanently excluded from this query and never delivered over HTTP.
- **8. Database Operations:** `this.prisma.content.problem.findUnique({ where: { slug }, include: { testCases: { where: { isSample: true } } } })`.
- **9. External Service Calls:** None.
- **10. Authorization & Security Checks:** Decorated with `@Public()`.
- **11. Success Response & Payload:** HTTP 200 OK with full Problem entity and `testCases` array containing only sample cases.
- **12. Failure Responses & Error Codes:** HTTP 404 Not Found `{"statusCode": 404, "message": {"code": "not_found"}}`.
- **13. Side Effects & Emitted Events:** None.
- **14. Postconditions & State Changes:** None (Read-only).
- **15. Related Functions:** `FN-PRB-001`, `FN-PRB-005`.
- **16. Source Code Evidence:** [`src/content/controllers/problems.controller.ts:41-48`](file:///E:/Dezolver%20project/dezlovers-main/Final%20product/dezlovers-main(cuurent%20update)/src/content/controllers/problems.controller.ts#L41-L48), [`src/content/services/content.service.ts:64-68`](file:///E:/Dezolver%20project/dezlovers-main/Final%20product/dezlovers-main(cuurent%20update)/src/content/services/content.service.ts#L64-L68), [`src/content/repositories/problems.repository.ts:39-49`](file:///E:/Dezolver%20project/dezlovers-main/Final%20product/dezlovers-main(cuurent%20update)/src/content/repositories/problems.repository.ts#L39-L49).
- **17. Edge Cases Handled:** Slugs with special characters or nonexistent slugs return clean 404.

---

#### FN-PRB-003: Create Problem Draft
- **Status:** `![Status: Confirmed](https://img.shields.io/badge/Status-Confirmed_Legacy-brightgreen)`
- **1. Trigger:** HTTP `POST /problems`
- **2. Actor:** Content Manager, Platform Admin, Platform Moderator.
- **3. Preconditions:** Caller must hold `create:content` permission claim and belong to `CONTENT_MANAGER_ROLES`.
- **4. Input Fields & Payload:**
  - Body (`CreateProblemDto`):
    - `title` (string, min 3 chars, required)
    - `slug` (string, optional)
    - `statementMd` (string, required)
    - `inputFormatMd` (string, optional)
    - `outputFormatMd` (string, optional)
    - `constraintsMd` (string, optional)
    - `difficulty` (string: `'easy' | 'medium' | 'hard'`, optional)
    - `topics` (string[], optional)
    - `companies` (string[], optional)
    - `allowedLanguages` (string[], optional)
    - `timeLimitMs` (number: 100–60000, optional)
    - `memoryLimitMb` (number: 8–512, optional)
- **5. Input Validation Rules:** Enforced by class-validator ([`src/content/dto/content.dto.ts:15-78`](file:///E:/Dezolver%20project/dezlovers-main/Final%20product/dezlovers-main(cuurent%20update)/src/content/dto/content.dto.ts#L15-L78)).
- **6. Execution & Processing Logic:**
  - Controller validates JWT (`@RequireAuth()`) and checks permission (`@RequirePermission('create:content')`).
  - Calls `ContentService.createProblem(dto, principal)` ([`src/content/services/content.service.ts:70-77`](file:///E:/Dezolver%20project/dezlovers-main/Final%20product/dezlovers-main(cuurent%20update)/src/content/services/content.service.ts#L70-L77)).
  - Verifies actor role against `CONTENT_MANAGER_ROLES = ['content_manager', 'platform_admin', 'platform_moderator']`. Throws `ForbiddenException({ code: 'forbidden' })` if mismatched.
  - Derives slug: `dto.slug ?? toSlug(dto.title)` via regex `toLowerCase().replace(/[^a-z0-9]+/g, '-').replace(/^-|-$/g, '')`.
  - Invokes repository `create` setting `status: 'draft'`, `authorUserId: actor.userId`.
- **7. Business Rules Enforced:** Problems always initialize in `draft` status.
- **8. Database Operations:** `this.prisma.content.problem.create({ data: { ...data, status: 'draft' } })`.
- **9. External Service Calls:** None.
- **10. Authorization & Security Checks:** Guard verifies `create:content`; service checks role membership; audit interceptor logs `problem.create`.
- **11. Success Response & Payload:** HTTP 201 Created with full created Problem record.
- **12. Failure Responses & Error Codes:**
  - HTTP 400 Bad Request: DTO validation failures.
  - HTTP 401 Unauthorized: Missing or invalid JWT.
  - HTTP 403 Forbidden: Missing `create:content` or not in `CONTENT_MANAGER_ROLES`.
  - HTTP 409 Conflict / 500: Duplicate slug violation on `problems_slug_key`.
- **13. Side Effects & Emitted Events:** Audit log record emitted via `@AuditAction('problem.create')`.
- **14. Postconditions & State Changes:** New problem record committed in `content.problems` with status `'draft'`.
- **15. Related Functions:** `FN-PRB-004`, `FN-PRB-005`.
- **16. Source Code Evidence:** [`src/content/controllers/problems.controller.ts:50-63`](file:///E:/Dezolver%20project/dezlovers-main/Final%20product/dezlovers-main(cuurent%20update)/src/content/controllers/problems.controller.ts#L50-L63), [`src/content/services/content.service.ts:70-77`](file:///E:/Dezolver%20project/dezlovers-main/Final%20product/dezlovers-main(cuurent%20update)/src/content/services/content.service.ts#L70-L77), [`src/content/repositories/problems.repository.ts:55-71`](file:///E:/Dezolver%20project/dezlovers-main/Final%20product/dezlovers-main(cuurent%20update)/src/content/repositories/problems.repository.ts#L55-L71).
- **17. Edge Cases Handled:** Missing explicit slug automatically derived from title.

---

#### FN-PRB-004: Ingest Test Case
- **Status:** `![Status: Confirmed](https://img.shields.io/badge/Status-Confirmed_Legacy-brightgreen)`
- **1. Trigger:** HTTP `POST /problems/:id/test-cases`
- **2. Actor:** Content Manager, Platform Admin.
- **3. Preconditions:** Target problem must exist and its status must NOT be `'published'`.
- **4. Input Fields & Payload:**
  - Path parameter: `id` (UUID, required).
  - Body (`AddTestCaseDto`):
    - `input` (string, required)
    - `expectedOutput` (string, required)
    - `isSample` (boolean, optional, default: false)
    - `weight` (number: 1–100, optional, default: 1)
- **5. Input Validation Rules:** Enforced by class-validator ([`src/content/dto/content.dto.ts:80-100`](file:///E:/Dezolver%20project/dezlovers-main/Final%20product/dezlovers-main(cuurent%20update)/src/content/dto/content.dto.ts#L80-L100)).
- **6. Execution & Processing Logic:**
  - Controller checks `@RequirePermission('create:content')`.
  - Calls `ContentService.addTestCase(problemId, dto, principal)` ([`src/content/services/content.service.ts:79-88`](file:///E:/Dezolver%20project/dezlovers-main/Final%20product/dezlovers-main(cuurent%20update)/src/content/services/content.service.ts#L79-L88)).
  - Loads problem via `problemsRepo.findById(problemId)`. If null, throws `NotFoundException({ code: 'not_found' })`.
  - Invariant check: `if (problem.status === 'published') throw new BadRequestException({ code: 'problem_published' });`.
  - Invokes repository `addTestCase` to insert test case.
- **7. Business Rules Enforced:** **IMMUTABILITY INVARIANT:** Once a problem is published, no new test cases may be added.
- **8. Database Operations:** `this.prisma.content.problemTestCase.create({ data: { problemId, ...dto } })`.
- **9. External Service Calls:** None.
- **10. Authorization & Security Checks:** Guard verifies `create:content`; audit action `problem.test_case.add`.
- **11. Success Response & Payload:** HTTP 201 Created with created `ProblemTestCase` record.
- **12. Failure Responses & Error Codes:**
  - HTTP 400 Bad Request: DTO validation error or `code: 'problem_published'`.
  - HTTP 401 Unauthorized: Unauthenticated request.
  - HTTP 403 Forbidden: Lacks `create:content`.
  - HTTP 404 Not Found: `code: 'not_found'`.
- **13. Side Effects & Emitted Events:** Audit log record emitted via `@AuditAction('problem.test_case.add')`.
- **14. Postconditions & State Changes:** New record created in `content.problem_test_cases`.
- **15. Related Functions:** `FN-PRB-003`, `FN-PRB-005`.
- **16. Source Code Evidence:** [`src/content/controllers/problems.controller.ts:65-83`](file:///E:/Dezolver%20project/dezlovers-main/Final%20product/dezlovers-main(cuurent%20update)/src/content/controllers/problems.controller.ts#L65-L83), [`src/content/services/content.service.ts:79-88`](file:///E:/Dezolver%20project/dezlovers-main/Final%20product/dezlovers-main(cuurent%20update)/src/content/services/content.service.ts#L79-L88), [`src/content/repositories/problems.repository.ts:80-88`](file:///E:/Dezolver%20project/dezlovers-main/Final%20product/dezlovers-main(cuurent%20update)/src/content/repositories/problems.repository.ts#L80-L88).
- **17. Edge Cases Handled:** Attempting to mutate published problem blocked with explicit error.

---

#### FN-PRB-005: Publish Problem
- **Status:** `![Status: Confirmed](https://img.shields.io/badge/Status-Confirmed_Legacy-brightgreen)`
- **1. Trigger:** HTTP `POST /problems/:id/publish`
- **2. Actor:** Content Manager, Platform Admin.
- **3. Preconditions:**
  - Problem must exist.
  - Problem status must NOT be `'published'`.
  - Problem MUST have at least one test case associated.
- **4. Input Fields & Payload:**
  - Path parameter: `id` (UUID, required).
- **5. Input Validation Rules:** UUID string.
- **6. Execution & Processing Logic:**
  - Controller checks `@RequirePermission('publish:content')`.
  - Calls `ContentService.publishProblem(id, principal)` ([`src/content/services/content.service.ts:90-104`](file:///E:/Dezolver%20project/dezlovers-main/Final%20product/dezlovers-main(cuurent%20update)/src/content/services/content.service.ts#L90-L104)).
  - Loads problem via `problemsRepo.findById(id)`. If null, throws `NotFoundException({ code: 'not_found' })`.
  - Invariant check 1: `if (problem.status === 'published') throw new BadRequestException({ code: 'already_published' });`.
  - Invariant check 2: `const testCases = await this.problemsRepo.getTestCases(id, true); if (testCases.length === 0) throw new BadRequestException({ code: 'no_test_cases' });`.
  - Updates problem status to `'published'` and sets `published_at = NOW()`.
- **7. Business Rules Enforced:** Zero-test-case problems can never be published to the platform.
- **8. Database Operations:** `this.prisma.content.problem.update({ where: { id }, data: { status: 'published', publishedAt: new Date() } })`.
- **9. External Service Calls:** None.
- **10. Authorization & Security Checks:** Guard verifies `publish:content`; audit action `problem.publish`.
- **11. Success Response & Payload:** HTTP 200 OK with updated Problem record.
- **12. Failure Responses & Error Codes:**
  - HTTP 400 Bad Request: `code: 'already_published'` or `code: 'no_test_cases'`.
  - HTTP 401 Unauthorized: Unauthenticated request.
  - HTTP 403 Forbidden: Lacks `publish:content`.
  - HTTP 404 Not Found: `code: 'not_found'`.
- **13. Side Effects & Emitted Events:** Audit log record emitted via `@AuditAction('problem.publish')`.
- **14. Postconditions & State Changes:** `content.problems.status` set to `'published'`, `published_at` set to current timestamp.
- **15. Related Functions:** `FN-PRB-001`, `FN-PRB-004`.
- **16. Source Code Evidence:** [`src/content/controllers/problems.controller.ts:85-98`](file:///E:/Dezolver%20project/dezlovers-main/Final%20product/dezlovers-main(cuurent%20update)/src/content/controllers/problems.controller.ts#L85-L98), [`src/content/services/content.service.ts:90-104`](file:///E:/Dezolver%20project/dezlovers-main/Final%20product/dezlovers-main(cuurent%20update)/src/content/services/content.service.ts#L90-L104), [`src/content/repositories/problems.repository.ts:73-78`](file:///E:/Dezolver%20project/dezlovers-main/Final%20product/dezlovers-main(cuurent%20update)/src/content/repositories/problems.repository.ts#L73-L78).
- **17. Edge Cases Handled:** Re-publishing published problem prevented with `already_published` error.

---

#### FN-PRB-006: Cross-Content Full-Text Search
- **Status:** `![Status: Confirmed](https://img.shields.io/badge/Status-Confirmed_Legacy-brightgreen)`
- **1. Trigger:** HTTP `POST /search`
- **2. Actor:** Authenticated User holding `read:cohort`.
- **3. Preconditions:** User authenticated with JWT.
- **4. Input Fields & Payload:**
  - Body (`SearchDto`): `q` (string, required), `kind` (optional string: `'all' | 'room' | 'problem' | 'course'`, default: `'all'`).
- **5. Input Validation Rules:** Enforced by `SearchDto` class-validator.
- **6. Execution & Processing Logic:**
  - Calls `ContentService.search(q, kind)` ([`src/content/services/content.service.ts:227-239`](file:///E:/Dezolver%20project/dezlovers-main/Final%20product/dezlovers-main(cuurent%20update)/src/content/services/content.service.ts#L227-L239)).
  - When `kind === 'all' || kind === 'problem'`, calls `ProblemsRepository.searchFts(q)` ([`src/content/repositories/problems.repository.ts:99-110`](file:///E:/Dezolver%20project/dezlovers-main/Final%20product/dezlovers-main(cuurent%20update)/src/content/repositories/problems.repository.ts#L99-L110)).
  - Executes query: `where: { status: 'published', OR: [{ title: { contains: q, mode: 'insensitive' } }] }, take: 20`.
  - Returns selected fields: `id, slug, title, difficulty`.
- **7. Business Rules Enforced:** Only published problems appear in search results; search queries match against titles case-insensitively.
- **8. Database Operations:** `this.prisma.content.problem.findMany(...)`.
- **9. External Service Calls:** None.
- **10. Authorization & Security Checks:** Guard verifies `read:cohort`.
- **11. Success Response & Payload:** HTTP 200 OK with `{ problems: ProblemSearchResult[], rooms?: [...], courses?: [...] }`.
- **12. Failure Responses & Error Codes:** HTTP 401 Unauthorized; HTTP 403 Forbidden.
- **13. Side Effects & Emitted Events:** None.
- **14. Postconditions & State Changes:** None (Read-only).
- **15. Related Functions:** `FN-PRB-001`.
- **16. Source Code Evidence:** [`src/content/controllers/search.controller.ts:14-22`](file:///E:/Dezolver%20project/dezlovers-main/Final%20product/dezlovers-main(cuurent%20update)/src/content/controllers/search.controller.ts#L14-L22), [`src/content/services/content.service.ts:227-239`](file:///E:/Dezolver%20project/dezlovers-main/Final%20product/dezlovers-main(cuurent%20update)/src/content/services/content.service.ts#L227-L239), [`src/content/repositories/problems.repository.ts:99-110`](file:///E:/Dezolver%20project/dezlovers-main/Final%20product/dezlovers-main(cuurent%20update)/src/content/repositories/problems.repository.ts#L99-L110).
- **17. Edge Cases Handled:** Substring matching handles mixed-case searches seamlessly.

---

#### FN-PRB-007: Background CSV Problem Import
- **Status:** `![Status: Conflicting](https://img.shields.io/badge/Status-Conflicting_Implementation-orange)`
- **1. Trigger:** BullMQ job dequeue in `CsvImportConsumer` ([`src/worker/consumers/csv-import.consumer.ts:160-186`](file:///E:/Dezolver%20project/dezlovers-main/Final%20product/dezlovers-main(cuurent%20update)/src/worker/consumers/csv-import.consumer.ts#L160-L186)).
- **2. Actor:** Background Worker Daemon (`worker`).
- **3. Preconditions:** S3 CSV file uploaded with header rows for bulk problem import.
- **4. Input Fields & Payload:** Streamed CSV rows with columns `title`, `difficulty`, `description`, `topics`.
- **5. Input Validation Rules:** `parseProblemRow` parses comma-separated topics.
- **6. Execution & Processing Logic:**
  - Iterates over parsed CSV rows and invokes `this.prisma.content.problem.create`.
  - **CRITICAL FORENSIC DEFECT:** Passes `tenantId`, `description`, and omits `slug` and `statementMd`.
- **7. Business Rules Enforced:** None (fails at runtime).
- **8. Database Operations:** Attempted `this.prisma.content.problem.create(...)`.
- **9. External Service Calls:** S3 stream read.
- **10. Authorization & Security Checks:** None at worker execution time.
- **11. Success Response & Payload:** Intended log: `Problem import done`.
- **12. Failure Responses & Error Codes:** Prisma runtime exception: Unknown arguments `tenantId`, `description`, missing required arguments `slug`, `statementMd`.
- **13. Side Effects & Emitted Events:** Fails job and routes to DLQ if retries exceed maximum.
- **14. Postconditions & State Changes:** No records created; job crashes.
- **15. Related Functions:** `FN-PRB-003`.
- **16. Source Code Evidence:** [`src/worker/consumers/csv-import.consumer.ts:160-186`](file:///E:/Dezolver%20project/dezlovers-main/Final%20product/dezlovers-main(cuurent%20update)/src/worker/consumers/csv-import.consumer.ts#L160-L186).
- **17. Edge Cases Handled:** Unhandled schema mismatch defect.

---

## SECTION E: DETAILED WORKFLOWS

### E.1 Sequence Diagrams

#### Workflow 1: Problem Authoring & Multi-Step Ingestion
```mermaid
sequenceDiagram
    autonumber
    actor Mgr as Content Manager
    participant UI as ProblemCreatePage.tsx
    participant API as ProblemsController
    participant Svc as ContentService
    participant Repo as ProblemsRepository
    participant DB as content.problems / test_cases

    Mgr->>UI: Inputs Problem Statement & Configuration
    Mgr->>UI: Adds 2 Sample Cases and 3 Hidden Cases
    Mgr->>UI: Clicks "Save Problem"
    UI->>API: POST /problems (CreateProblemDto)
    API->>Svc: createProblem()
    Svc->>Repo: create() -> INSERT content.problems (status: draft)
    Repo-->>API: Problem Created (id: UUID-A, slug: 'two-sum')
    API-->>UI: HTTP 201 Created

    rect rgb(240, 248, 255)
        note right of UI: Non-Atomic Sequential Ingestion
        loop For each test case i in [1..5]
            UI->>API: POST /problems/UUID-A/test-cases (input, output, isSample, weight)
            API->>Svc: addTestCase()
            Svc->>Repo: addTestCase() -> INSERT content.problem_test_cases
            Repo-->>API: Test Case Created
            API-->>UI: HTTP 201 Created
        end
    end

    UI->>UI: Redirects to /content/problems/two-sum
```

#### Workflow 2: Student Problem Exploration & Test Case Masking
```mermaid
sequenceDiagram
    autonumber
    actor Stud as Student
    participant UI as ProblemDetailPage.tsx
    participant API as ProblemsController
    participant Repo as ProblemsRepository
    participant DB as PostgreSQL (content schema)

    Stud->>UI: Navigates to /content/problems/two-sum
    UI->>API: GET /problems/two-sum
    API->>Repo: findBySlug('two-sum')
    Repo->>DB: SELECT * FROM content.problems WHERE slug = 'two-sum'
    Repo->>DB: SELECT * FROM content.problem_test_cases WHERE problem_id = ID AND is_sample = true
    note over DB,Repo: Hidden test cases (is_sample = false) are filtered out at the DB level!
    DB-->>Repo: Problem data + 2 Sample Test Cases
    Repo-->>API: Sanitized Problem Payload
    API-->>UI: HTTP 200 OK (Sample Cases Only)
    UI->>Stud: Renders Markdown Statement & Sample Test Cases (Zero Hidden Leak)
```

### E.2 Step-by-Step Execution Traces

#### Trace 1: Problem Drafting (`FN-PRB-003`)
1. **User Action:** Content Manager completes form in `ProblemCreatePage.tsx` and clicks "Save Problem".
2. **Client Validation:** Form inputs validated against Zod schema `createProblemSchema` ([`frontend/src/lib/schemas/content.schemas.ts`](file:///E:/Dezolver%20project/dezlovers-main/Final%20product/frontend/src/lib/schemas/content.schemas.ts)).
3. **HTTP Dispatch:** Axios client dispatches `POST /problems` with JSON body and Bearer JWT.
4. **Ingress & Guards:**
   - `AuthGuard` verifies Bearer JWT RS256 signature and attaches `req.principal`.
   - `PermissionGuard` checks `principal.roles` for permission `'create:content'`.
5. **Controller Processing:** `ProblemsController.create` receives validated `CreateProblemDto`.
6. **Service Logic:**
   - `ContentService.createProblem` verifies actor role membership in `['content_manager', 'platform_admin', 'platform_moderator']`.
   - Auto-generates slug: `toSlug(dto.title)` if `dto.slug` omitted.
7. **Database Mutation:** `ProblemsRepository.create` inserts row into `content.problems` with `status: 'draft'`.
8. **HTTP Response:** Returns HTTP 201 Created with problem entity.
9. **UI Reaction:** Frontend captures `createdProblemId`, then executes asynchronous loop to add test cases.

---

## SECTION F: API SPECIFICATION

### F.1 Endpoint Inventory Table

| HTTP Method | Route | Controller Method | Guards Applied | Required Permission | Status |
|---|---|---|---|---|---|
| `GET` | `/problems` | `ProblemsController.list` | None (`@Public()`) | Public | `CONFIRMED` |
| `GET` | `/problems/:slug` | `ProblemsController.getBySlug` | None (`@Public()`) | Public | `CONFIRMED` |
| `POST` | `/problems` | `ProblemsController.create` | `AuthGuard`, `PermissionGuard` | `create:content` | `CONFIRMED` |
| `POST` | `/problems/:id/test-cases` | `ProblemsController.addTestCase` | `AuthGuard`, `PermissionGuard` | `create:content` | `CONFIRMED` |
| `POST` | `/problems/:id/publish` | `ProblemsController.publish` | `AuthGuard`, `PermissionGuard` | `publish:content` | `CONFIRMED` |
| `POST` | `/search` | `SearchController.search` | `AuthGuard`, `PermissionGuard` | `read:cohort` | `CONFIRMED` |

---

### F.2 Endpoint Deep-Dive Specifications

#### F.2.1 `GET /problems`
- **Method & Route:** `GET /problems`
- **Controller Method:** `ProblemsController.list` ([`src/content/controllers/problems.controller.ts:27-39`](file:///E:/Dezolver%20project/dezlovers-main/Final%20product/dezlovers-main(cuurent%20update)/src/content/controllers/problems.controller.ts#L27-L39))
- **Headers:** `Accept: application/json`
- **Query Parameters:**
  - `difficulty` (`string`, optional): Filter by `'easy'`, `'medium'`, `'hard'`.
  - `q` (`string`, optional): Case-insensitive substring match on problem title.
  - `limit` (`string`, optional): Maximum records to return (parsed to integer, default: 20).
  - `cursor` (`string`, optional): Base64-encoded pagination cursor (`{"id":"<uuid>"}`).
- **Success Response (HTTP 200 OK):**
  ```json
  {
    "items": [
      {
        "id": "7b3b9b4a-4b92-4f38-9e5c-9cb392a54321",
        "slug": "two-sum",
        "title": "Two Sum",
        "difficulty": "easy",
        "topics": ["arrays", "hash-table"],
        "companies": ["google", "amazon"],
        "status": "published",
        "createdAt": "2026-06-01T12:00:00.000Z"
      }
    ],
    "nextCursor": "eyJpZCI6IjdiM2I5YjRhLTRiOTItNGYzOC05ZTVjLTljYjM5MmE1NDMyMSJ9"
  }
  ```

#### F.2.2 `GET /problems/:slug`
- **Method & Route:** `GET /problems/:slug`
- **Controller Method:** `ProblemsController.getBySlug` ([`src/content/controllers/problems.controller.ts:46-48`](file:///E:/Dezolver%20project/dezlovers-main/Final%20product/dezlovers-main(cuurent%20update)/src/content/controllers/problems.controller.ts#L46-L48))
- **Headers:** `Accept: application/json`
- **Path Parameters:**
  - `slug` (`string`, required): Problem slug identifier.
- **Success Response (HTTP 200 OK):**
  ```json
  {
    "id": "7b3b9b4a-4b92-4f38-9e5c-9cb392a54321",
    "slug": "two-sum",
    "title": "Two Sum",
    "statementMd": "Given an array of integers `nums` and an integer `target`...",
    "inputFormatMd": "First line contains N. Second line contains N integers.",
    "outputFormatMd": "Output two indices.",
    "constraintsMd": "2 <= nums.length <= 10^4",
    "difficulty": "easy",
    "topics": ["arrays", "hash-table"],
    "companies": ["google"],
    "allowedLanguages": ["python", "cpp", "java"],
    "timeLimitMs": 2000,
    "memoryLimitMb": 256,
    "status": "published",
    "authorUserId": "11111111-2222-3333-4444-555555555555",
    "publishedAt": "2026-06-01T12:30:00.000Z",
    "createdAt": "2026-06-01T12:00:00.000Z",
    "updatedAt": "2026-06-01T12:30:00.000Z",
    "testCases": [
      {
        "id": "99999999-8888-7777-6666-555555555555",
        "input": "4\n2 7 11 15\n9",
        "expectedOutput": "0 1",
        "isSample": true
      }
    ]
  }
  ```
- **Error Responses:**
  | Status Code | Error Code | Reason |
  |---|---|---|
  | `404 Not Found` | `not_found` | No problem exists matching the provided slug |

#### F.2.3 `POST /problems`
- **Method & Route:** `POST /problems`
- **Controller Method:** `ProblemsController.create` ([`src/content/controllers/problems.controller.ts:61-63`](file:///E:/Dezolver%20project/dezlovers-main/Final%20product/dezlovers-main(cuurent%20update)/src/content/controllers/problems.controller.ts#L61-L63))
- **Headers:** `Authorization: Bearer <JWT>`, `Content-Type: application/json`
- **Request Body Schema (`CreateProblemDto`):**
  ```json
  {
    "title": "Median of Two Sorted Arrays",
    "slug": "median-of-two-sorted-arrays",
    "statementMd": "Given two sorted arrays nums1 and nums2...",
    "inputFormatMd": "Array size followed by elements.",
    "outputFormatMd": "The median value as float.",
    "constraintsMd": "nums1.length <= 1000",
    "difficulty": "hard",
    "topics": ["binary-search", "arrays"],
    "companies": ["amazon", "microsoft"],
    "allowedLanguages": ["python", "cpp"],
    "timeLimitMs": 2000,
    "memoryLimitMb": 256
  }
  ```
- **Validation Rules (`CreateProblemDto`):**
  - `title`: `@IsString()`, `@MinLength(3)`
  - `slug`: `@IsOptional()`, `@IsString()`
  - `statementMd`: `@IsString()`
  - `difficulty`: `@IsOptional()`, `@IsIn(['easy', 'medium', 'hard'])`
  - `topics`, `companies`, `allowedLanguages`: `@IsOptional()`, `@IsArray()`
  - `timeLimitMs`: `@IsOptional()`, `@IsNumber()`, `@Min(100)`, `@Max(60000)`
  - `memoryLimitMb`: `@IsOptional()`, `@IsNumber()`, `@Min(8)`, `@Max(512)`
- **Success Response (HTTP 201 Created):** Full created `Problem` record.
- **Error Responses:**
  | Status Code | Error Code | Reason |
  |---|---|---|
  | `400 Bad Request` | Validation Error | Payload violates class-validator constraints |
  | `401 Unauthorized` | Missing JWT | Bearer token omitted or invalid |
  | `403 Forbidden` | `insufficient_permission` / `forbidden` | Missing `create:content` or not in `CONTENT_MANAGER_ROLES` |

#### F.2.4 `POST /problems/:id/test-cases`
- **Method & Route:** `POST /problems/:id/test-cases`
- **Controller Method:** `ProblemsController.addTestCase` ([`src/content/controllers/problems.controller.ts:77-83`](file:///E:/Dezolver%20project/dezlovers-main/Final%20product/dezlovers-main(cuurent%20update)/src/content/controllers/problems.controller.ts#L77-L83))
- **Headers:** `Authorization: Bearer <JWT>`, `Content-Type: application/json`
- **Path Parameters:** `id` (UUID, required)
- **Request Body Schema (`AddTestCaseDto`):**
  ```json
  {
    "input": "5\n1 2 3 4 5",
    "expectedOutput": "15",
    "isSample": false,
    "weight": 5
  }
  ```
- **Validation Rules (`AddTestCaseDto`):**
  - `input`: `@IsString()`
  - `expectedOutput`: `@IsString()`
  - `isSample`: `@IsOptional()`, `@IsBoolean()`
  - `weight`: `@IsOptional()`, `@IsNumber()`, `@Min(1)`, `@Max(100)`
- **Success Response (HTTP 201 Created):** Created `ProblemTestCase` record.
- **Error Responses:**
  | Status Code | Error Code | Reason |
  |---|---|---|
  | `400 Bad Request` | `problem_published` | Cannot add test cases to an already published problem |
  | `401 Unauthorized` | Missing JWT | Unauthenticated caller |
  | `403 Forbidden` | `insufficient_permission` | Lacks `create:content` |
  | `404 Not Found` | `not_found` | Problem with given UUID does not exist |

#### F.2.5 `POST /problems/:id/publish`
- **Method & Route:** `POST /problems/:id/publish`
- **Controller Method:** `ProblemsController.publish` ([`src/content/controllers/problems.controller.ts:96-98`](file:///E:/Dezolver%20project/dezlovers-main/Final%20product/dezlovers-main(cuurent%20update)/src/content/controllers/problems.controller.ts#L96-L98))
- **Headers:** `Authorization: Bearer <JWT>`
- **Path Parameters:** `id` (UUID, required)
- **Success Response (HTTP 200 OK):** Updated `Problem` record with `status: 'published'`.
- **Error Responses:**
  | Status Code | Error Code | Reason |
  |---|---|---|
  | `400 Bad Request` | `already_published` | Problem is already published |
  | `400 Bad Request` | `no_test_cases` | Problem has 0 test cases and cannot be published |
  | `401 Unauthorized` | Missing JWT | Unauthenticated caller |
  | `403 Forbidden` | `insufficient_permission` | Lacks `publish:content` |
  | `404 Not Found` | `not_found` | Problem ID not found |

---

## SECTION G: FRONTEND SPECIFICATION

### G.1 Page & Route Inventory
| Route Path | Page Component | Guard / Access Level | Data Fetching Hook | Key UI Views |
|---|---|---|---|---|
| `/content/problems` | `ProblemsPage.tsx` | Authenticated | `useProblems` | Search bar, Level filter, Status filter, Problem table, Pagination |
| `/content/problems/create` | `ProblemCreatePage.tsx` | `manage:problem` | `useCreateProblem`, `useAddTestCase` | Title, Markdown textareas, Test case builder, Tags, Limits |
| `/content/problems/:slug` | `ProblemDetailPage.tsx` | Authenticated | `useProblem`, `usePublishProblem` | Header meta, Markdown renderer, Test case cards, Config table |

### G.2 Component Breakdown & Visual Hierarchy
1. **Catalog View (`ProblemsPage.tsx`):**
   - Header: Title, description, "New Problem" action button (conditionally rendered if `canAuthor = usePermissions('manage:problem')`).
   - Filter Toolbar: Debounced search input (`useDebounce(search, 350)`), difficulty dropdown (`all`, `easy`, `medium`, `hard`), status dropdown (`all`, `draft`, `published`), refresh button.
   - Table List: Iterates over problems with `ContentStatusBadge`, `DifficultyBadge`, topic badges, and timestamp.
2. **Authoring Interface (`ProblemCreatePage.tsx`):**
   - Core Details Card: Title, difficulty select, time/memory limit inputs.
   - Markdown Section Cards: Problem statement, input format, output format, constraints.
   - Dynamic Test Case Card Array: Repeater inputs for `input`, `expectedOutput`, `explanation`, and `isSample` switch.
   - Topic Pills & Language Checkboxes: Tag management and compiler whitelisting.
3. **Detail & Audit View (`ProblemDetailPage.tsx`):**
   - Header Strip: Problem title, publish button (enabled when `canManage && problem.status === 'draft'`), status badge, difficulty badge.
   - Tabs Container:
     - `statement`: Pre-formatted Markdown blocks for statement, input, output, and constraints.
     - `test-cases`: Sample test case input/output grids and hidden test case indicator.
     - `config`: Two-column definition list detailing `timeLimitMs`, `memoryLimitMb`, `allowedLanguages`, `createdAt`, and `id`.
4. **Student Assessment View (`ProblemPanel.tsx`):**
   - Compact vertical layout designed for split-screen coding exams.
   - Renders `ProblemHeader`, Markdown statement, constraints code block, and `ProblemExamples` side-by-side blocks.

### G.3 State Management & Data Fetching
- **Server State (TanStack Query):**
  - Query keys defined in `QUERY_KEYS.PROBLEMS` (`['problems']`).
  - Cache invalidation executed via `qc.invalidateQueries({ queryKey: QUERY_KEYS.PROBLEMS })` upon problem creation, test case ingestion, and publication.
  - Stale time configured to 2 minutes (`staleTime: 2 * 60 * 1000`).

---

## SECTION H: BUSINESS RULES AND CALCULATIONS

### H.1 Mathematical Formulas & Algorithms

#### H.1.1 Automatic Slug Derivation Algorithm
When an author leaves the `slug` field empty during problem creation, the backend derives a clean URL slug via regex substitution ([`src/content/services/content.service.ts:22-27`](file:///E:/Dezolver project/dezlovers-main/Final product/dezlovers-main(cuurent update)/src/content/services/content.service.ts#L22-L27)):
$$\text{Slug}(T) = \text{trim}(\text{replace}(\text{replace}(\text{lower}(T), \; \neg[a\text{-}z0\text{-}9]+, \; \text{'-'}), \; \text{'-'}\$, \; \text{''}))$$
Implemented as:
```typescript
function toSlug(title: string): string {
  return title
    .toLowerCase()
    .replace(/[^a-z0-9]+/g, '-')
    .replace(/^-|-$/g, '');
}
```

#### H.1.2 Base64 Pagination Cursor Encoding & Decoding
Cursor-based pagination utilizes base64-serialized JSON objects containing the last evaluated problem UUID:
$$\text{Cursor}_{\text{encoded}} = \text{base64}(\text{JSON}(\{ \text{id}: \text{lastItemId} \}))$$
$$\text{Cursor}_{\text{decoded}} = \text{JSON.parse}(\text{base64\_decode}(S))$$

### H.2 Validation Rules & Boundary Conditions
- **Problem Title:** Length must be $\ge 3$ characters (`@MinLength(3)`).
- **Execution Time Limit (`timeLimitMs`):**
  $$100 \le \text{timeLimitMs} \le 60000 \quad (\text{Default: } 2000 \text{ ms})$$
- **Execution Memory Limit (`memoryLimitMb`):**
  $$8 \le \text{memoryLimitMb} \le 512 \quad (\text{Default: } 256 \text{ MB})$$
- **Test Case Weight:**
  $$1 \le \text{weight} \le 100 \quad (\text{Default: } 1)$$
- **Test Case Ingestion Invariant:**
  $$\text{Problem.status} \ne \text{'published'}$$
- **Publication Invariant:**
  $$\text{count}(\text{ProblemTestCase}) \ge 1$$

---

## SECTION I: PERMISSIONS AND SECURITY

### I.1 Role-Based Access Control (RBAC) Matrix

| Operation | Required Permission | Allowed Platform Roles | Allowed College Roles | Student | Guest |
|---|---|---|---|:---:|:---:|
| List Published Problems (`GET /problems`) | None (`@Public()`) | All | All | ✅ | ✅ |
| View Problem Details (`GET /problems/:slug`) | None (`@Public()`) | All | All | ✅ | ✅ |
| Draft Problem (`POST /problems`) | `create:content` | `content_manager`, `platform_admin` | ❌ | ❌ | ❌ |
| Add Test Case (`POST /problems/:id/test-cases`) | `create:content` | `content_manager`, `platform_admin` | ❌ | ❌ | ❌ |
| Publish Problem (`POST /problems/:id/publish`) | `publish:content` | `content_manager`, `platform_admin` | ❌ | ❌ | ❌ |
| Cross-Content Search (`POST /search`) | `read:cohort` | All Authenticated Roles | All Authenticated Roles | ✅ | ❌ |

### I.2 Multi-Tenant Isolation Enforcement
- **Platform-Global Scope:** Table `content.problems` contains **NO `tenant_id` column**.
- **Cross-Tenant Access Reality:** Any problem created on Dezolver is universally visible across all institutional tenants, colleges, and students.
- **Absence of Private Institutional Problems:** The legacy codebase has no database or logical mechanism for an individual college, department, or professor to author private, confidential exam questions.

### I.3 Cryptographic Standards & Token Handling
- Endpoints requiring authorization validate RS256-signed JWT access tokens issued by Module 1.
- Tokens are extracted via standard Bearer scheme from the `Authorization` header.

### I.4 Security Vulnerabilities & Exposure Audit
1. **Confidential Test Case Secrecy:** Verified secure in API layer. `ProblemsRepository.findBySlug` explicitly restricts Prisma queries to `where: { isSample: true }`, ensuring hidden test cases are never transmitted to student browsers.
2. **Missing Input Sanitization in Test Cases:** Test case inputs and expected outputs are stored as raw text without size limits. An administrator could theoretically ingest multi-gigabyte files leading to database disk exhaustion.
3. **Search Substring DoS Risk:** `POST /search` executes unindexed `ILIKE %query%` queries across `content.problems.title`. Concurrent search traffic with single-character queries could cause database CPU exhaustion.

---

## SECTION J: INTEGRATIONS AND DEPENDENCIES

### J.1 Internal Subsystem Dependencies
- **Module 6 (Code Submission & Judge0):** Reads `timeLimitMs`, `memoryLimitMb`, and `allowedLanguages` from `content.problems` to initialize execution jobs.
- **Module 7 (Evaluation Engine):** Worker queries `content.problem_test_cases` directly by `problemId` to retrieve hidden inputs and outputs for sandbox verification.
- **Module 9 (Assessments):** `assessment.assessments` contains a foreign key column `problem_id` linking exam questions to challenges in `content.problems`.

### J.2 External Infrastructure Integrations
- **Judge0 CE Runner:** Compilers and execution limits match Judge0 specifications. The language whitelist string array (`allowedLanguages`) corresponds to Judge0 execution profiles (`python`, `cpp`, `java`).
- **PostgreSQL Multi-Schema Engine:** Tables reside in dedicated schema `content`, separated from `identity`, `tenancy`, and `assessment`.

---

## SECTION K: EDGE CASES AND FAILURE HANDLING

### K.1 Infrastructure & Network Faults
- **Database Connection Failure:** If PostgreSQL is unreachable during `POST /problems`, NestJS returns HTTP 500. No partial problem record is created.
- **Test Case Loop Network Disconnection:** In `ProblemCreatePage.tsx`, test cases are inserted sequentially in a loop. If network drops on test case 3 of 5, test cases 1 and 2 remain saved in the database, while cases 3–5 fail. The problem remains in `draft` status.

### K.2 Concurrency & Race Conditions
- **Simultaneous Drafts with Identical Titles:** If two managers draft problems with identical titles simultaneously without providing explicit slugs, both generate identical derived slugs. PostgreSQL throws unique constraint violation on `problems_slug_key` for the second transaction, returning HTTP 500 / 409.
- **Concurrent Test Case Ingestion & Publication:** If Manager A adds a test case while Manager B publishes the problem, Manager A's call will succeed or fail depending on whether the transaction commits before or after `status = 'published'`. Once published, subsequent test case insertions are rejected with `problem_published`.

### K.3 Input Anomalies & Malicious Payloads
- **Empty Test Case Set Publication:** Attempting to publish a problem with 0 test cases is explicitly trapped by `ContentService.publishProblem` and rejected with HTTP 400 `no_test_cases`.
- **Excessive Resource Limits:** Attempting to pass `timeLimitMs = 999999` is rejected by class-validator (`@Max(60000)`). Attempting to pass `memoryLimitMb = 2048` is rejected (`@Max(512)`).

---

## SECTION L: STATE MACHINES AND LIFECYCLE

### L.1 State Transition Diagram
```mermaid
stateDiagram-v2
    [*] --> DRAFT: POST /problems (Draft Created)
    DRAFT --> DRAFT: POST /problems/:id/test-cases (Test Cases Added)
    DRAFT --> PUBLISHED: POST /problems/:id/publish (Invariant: testCases.length >= 1)
    PUBLISHED --> [*]: Terminal State (No Unpublish / No Delete)
```

### L.2 State Transition Matrix Table

| Initial State | Event / Trigger | Precondition / Guard | Next State | Side Effects |
|---|---|---|---|---|
| `[None]` | `POST /problems` | `create:content` permission | `draft` | Author ID logged, slug generated |
| `draft` | `POST /problems/:id/test-cases` | `create:content`, `status == 'draft'` | `draft` | Test case record created |
| `draft` | `POST /problems/:id/publish` | `publish:content`, `testCases.count >= 1` | `published` | `published_at` set to current timestamp |
| `published` | `POST /problems/:id/test-cases` | — | ❌ Blocked | Throws HTTP 400 `problem_published` |
| `published` | `POST /problems/:id/publish` | — | ❌ Blocked | Throws HTTP 400 `already_published` |

### L.3 Terminal States & Irreversible Operations
- **`published` is a One-Way Terminal State:** The legacy codebase has **NO unpublish, archive, or draft-rollback endpoints** for problems. Once published, a problem can never return to draft status.
- **Permanent Test Case Lock:** Once a problem transitions to `published`, test case additions are permanently rejected by `ContentService.addTestCase`.

---

## SECTION M: ACCEPTANCE CRITERIA (GHERKIN SCENARIOS)

```gherkin
Scenario: FN-PRB-001-AC-01: Public listing of published problems
  Given the database contains 5 published problems and 2 draft problems
  When an anonymous user sends a GET request to "/problems"
  Then the system responds with HTTP 200 OK
    And the response body contains exactly 5 problem items
    And every returned problem has status "published"
    And draft problems are completely omitted

Scenario: FN-PRB-002-AC-01: Public retrieval of problem statement and masking of hidden test cases
  Given a problem exists with slug "reverse-string" having 2 sample test cases and 5 hidden test cases
  When a student sends a GET request to "/problems/reverse-string"
  Then the system responds with HTTP 200 OK
    And the response body contains the Markdown statement and constraints
    And the "testCases" array contains exactly 2 items
    And every test case in the response has "isSample" equal to true
    And 0 hidden test cases are present in the response body

Scenario: FN-PRB-003-AC-01: Content manager drafts a new problem
  Given an authenticated user holding role "content_manager" and permission "create:content"
  When the user sends a POST request to "/problems" with title "Valid Anagram" and statement "Check anagram..."
  Then the system responds with HTTP 201 Created
    And the problem is inserted into the database with status "draft"
    And the derived slug is "valid-anagram"
    And default execution limits "timeLimitMs = 2000" and "memoryLimitMb = 256" are applied

Scenario: FN-PRB-004-AC-01: Adding a test case to a draft problem
  Given a problem exists in "draft" status with id "UUID-1"
    And an authenticated user holds "create:content"
  When the user sends a POST request to "/problems/UUID-1/test-cases" with input "test" and expectedOutput "pass"
  Then the system responds with HTTP 201 Created
    And a record is created in "content.problem_test_cases" linked to "UUID-1"

Scenario: FN-PRB-004-AC-02: Rejection of test case addition on published problem
  Given a problem exists in "published" status with id "UUID-2"
  When an authenticated user sends a POST request to "/problems/UUID-2/test-cases"
  Then the system responds with HTTP 400 Bad Request
    And the response error code is "problem_published"

Scenario: FN-PRB-005-AC-01: Successful problem publication with verified test cases
  Given a problem exists in "draft" status with id "UUID-3" having 3 test cases
    And an authenticated user holds "publish:content"
  When the user sends a POST request to "/problems/UUID-3/publish"
  Then the system responds with HTTP 200 OK
    And the problem status transitions to "published"
    And "published_at" timestamp is populated with current time

Scenario: FN-PRB-005-AC-02: Rejection of publication when problem has 0 test cases
  Given a problem exists in "draft" status with id "UUID-4" having 0 test cases
  When an authenticated user sends a POST request to "/problems/UUID-4/publish"
  Then the system responds with HTTP 400 Bad Request
    And the response error code is "no_test_cases"
    And the problem status remains "draft"
```

---

## SECTION N: QA AND TEST COVERAGE MATRIX

| Test ID | Category | Scenario Description | Input Data / Setup | Expected Outcome | Verification Method | Status |
|---|---|---|---|---|---|---|
| `TC-PRB-01` | Positive Path | Public catalog listing | `GET /problems?limit=10` | HTTP 200, 10 items, `nextCursor` string | Automated Jest / Supertest | `PASS` |
| `TC-PRB-02` | Positive Path | Fetch problem by slug | `GET /problems/two-sum` | HTTP 200, statement, sample test cases | Automated Jest / Supertest | `PASS` |
| `TC-PRB-03` | Positive Path | Create problem draft | Valid `CreateProblemDto` | HTTP 201, status `draft`, derived slug | Integration Test | `PASS` |
| `TC-PRB-04` | Positive Path | Add sample & hidden test cases | `POST /problems/:id/test-cases` | HTTP 201, records committed | Integration Test | `PASS` |
| `TC-PRB-05` | Positive Path | Publish problem with test cases | `POST /problems/:id/publish` | HTTP 200, status `published` | Integration Test | `PASS` |
| `TC-PRB-06` | Boundary | Minimum title length (3 chars) | Title: `"ABC"` | HTTP 201 Created | Unit Test / DTO | `PASS` |
| `TC-PRB-07` | Boundary | Title below minimum (2 chars) | Title: `"AB"` | HTTP 400 Bad Request | Unit Test / DTO | `PASS` |
| `TC-PRB-08` | Boundary | Minimum CPU limit (100 ms) | `timeLimitMs: 100` | HTTP 201 Created | Unit Test / DTO | `PASS` |
| `TC-PRB-09` | Boundary | Maximum CPU limit (60000 ms) | `timeLimitMs: 60000` | HTTP 201 Created | Unit Test / DTO | `PASS` |
| `TC-PRB-10` | Boundary | Exceeding CPU limit (60001 ms) | `timeLimitMs: 60001` | HTTP 400 Bad Request | Unit Test / DTO | `PASS` |
| `TC-PRB-11` | Security | Test case secrecy verification | `GET /problems/:slug` | Verify 0 test cases with `isSample: false` | E2E Security Audit | `PASS` |
| `TC-PRB-12` | Security | Unauthorized problem creation | Missing Bearer token | HTTP 401 Unauthorized | E2E Security Test | `PASS` |
| `TC-PRB-13` | Security | Insufficient role creation | Role: `student` | HTTP 403 Forbidden | E2E Security Test | `PASS` |
| `TC-PRB-14` | Business Rule | Publish with 0 test cases | Zero test cases in DB | HTTP 400 `no_test_cases` | Integration Test | `PASS` |
| `TC-PRB-15` | Business Rule | Mutate published problem | `POST /:id/test-cases` on published | HTTP 400 `problem_published` | Integration Test | `PASS` |

---

## SECTION O: SOURCE EVIDENCE AND TRACEABILITY

### O.1 Source File Inventory

| Layer | Physical File Path | Primary Class / Identifier | Lines | Description |
|---|---|---|---|---|
| Database Schema | [`prisma/content/schema.prisma`](file:///E:/Dezolver%20project/dezlovers-main/Final%20product/dezlovers-main(cuurent%20update)/prisma/content/schema.prisma) | `model Problem`, `model ProblemTestCase` | L15–L55 | PostgreSQL table definitions and constraints |
| Controller | [`src/content/controllers/problems.controller.ts`](file:///E:/Dezolver%20project/dezlovers-main/Final%20product/dezlovers-main(cuurent%20update)/src/content/controllers/problems.controller.ts) | `ProblemsController` | L1–L99 | Problem REST API endpoints |
| Controller | [`src/content/controllers/search.controller.ts`](file:///E:/Dezolver%20project/dezlovers-main/Final%20product/dezlovers-main(cuurent%20update)/src/content/controllers/search.controller.ts) | `SearchController` | L1–L23 | Full-text cross-content search endpoint |
| Service | [`src/content/services/content.service.ts`](file:///E:/Dezolver%20project/dezlovers-main/Final%20product/dezlovers-main(cuurent%20update)/src/content/services/content.service.ts) | `ContentService` | L40–L105 | Problem authoring, publication, slug derivation |
| Repository | [`src/content/repositories/problems.repository.ts`](file:///E:/Dezolver%20project/dezlovers-main/Final%20product/dezlovers-main(cuurent%20update)/src/content/repositories/problems.repository.ts) | `ProblemsRepository` | L1–L111 | Data access layer for problems and test cases |
| DTOs | [`src/content/dto/content.dto.ts`](file:///E:/Dezolver%20project/dezlovers-main/Final%20product/dezlovers-main(cuurent%20update)/src/content/dto/content.dto.ts) | `CreateProblemDto`, `AddTestCaseDto` | L15–L100 | Class-validator request contracts |
| Interceptor | [`src/shared/interceptors/envelope.interceptor.ts`](file:///E:/Dezolver%20project/dezlovers-main/Final%20product/dezlovers-main(cuurent%20update)/src/shared/interceptors/envelope.interceptor.ts) | `EnvelopeInterceptor` | L1–L37 | Response wrapper interceptor |
| Guard | [`src/shared/auth/permission.guard.ts`](file:///E:/Dezolver%20project/dezlovers-main/Final%20product/dezlovers-main(cuurent%20update)/src/shared/auth/permission.guard.ts) | `PermissionGuard` | L11–L76 | Role and permission check evaluation |
| Consumer | [`src/worker/consumers/csv-import.consumer.ts`](file:///E:/Dezolver%20project/dezlovers-main/Final%20product/dezlovers-main(cuurent%20update)/src/worker/consumers/csv-import.consumer.ts) | `CsvImportConsumer.importProblems` | L160–L186 | Broken background worker importing CSV problems |
| Frontend Page | [`frontend/src/pages/content/problems/ProblemsPage.tsx`](file:///E:/Dezolver%20project/dezlovers-main/Final%20product/frontend/src/pages/content/problems/ProblemsPage.tsx) | `ProblemsPage` | L1–L155 | Catalog list view with filtering |
| Frontend Page | [`frontend/src/pages/content/problems/ProblemCreatePage.tsx`](file:///E:/Dezolver%20project/dezlovers-main/Final%20product/frontend/src/pages/content/problems/ProblemCreatePage.tsx) | `ProblemCreatePage` | L1–L413 | Multi-step problem authoring view |
| Frontend Page | [`frontend/src/pages/content/problems/ProblemDetailPage.tsx`](file:///E:/Dezolver%20project/dezlovers-main/Final%20product/frontend/src/pages/content/problems/ProblemDetailPage.tsx) | `ProblemDetailPage` | L1–L217 | Problem inspection and publication view |
| Frontend Hook | [`frontend/src/hooks/useProblems.ts`](file:///E:/Dezolver%20project/dezlovers-main/Final%20product/frontend/src/hooks/useProblems.ts) | `useProblems`, `useProblem`, etc. | L1–L59 | TanStack Query bindings |
| Frontend UI | [`frontend/src/components/assessment/ProblemPanel.tsx`](file:///E:/Dezolver%20project/dezlovers-main/Final%20product/frontend/src/components/assessment/ProblemPanel.tsx) | `ProblemPanel` | L1–L62 | Student exam problem statement component |

---

## SECTION P: UNRESOLVED ITEMS AND DECISION LOG

### P.1 Missing Legacy Functionality
1. **No Problem Update (`PATCH`/`PUT`) Endpoints:** Once a problem is drafted or published, neither its title, statement, difficulty, nor resource limits can be updated via the API.
2. **No Problem Deletion (`DELETE`) Endpoints:** There is no mechanism to remove or archive obsolete or errant problems.
3. **No Test Case Update or Deletion:** Once a test case is added to a problem, it cannot be modified or deleted.
4. **No Tenant-Private Problem Isolation:** Colleges cannot author private problems; all problems are globally visible.

### P.2 Unverifiable Behaviors
- **Judge0 Compiler Execution Limit Tuning:** The exact behavior when `memoryLimitMb` exceeds physical worker container RAM cannot be verified statically without live Judge0 container infrastructure.

### P.3 Conflicting Implementations
1. **`EnvelopeInterceptor` vs. `ContentService.listProblems`:**
   - `EnvelopeInterceptor` ([`src/shared/interceptors/envelope.interceptor.ts:15-22`](file:///E:/Dezolver project/dezlovers-main/Final product/dezlovers-main(cuurent update)/src/shared/interceptors/envelope.interceptor.ts#L15-L22)) checks `isPaginatedResult` by looking for `'items' in data && 'pagination' in data`.
   - `ContentService.listProblems` returns `{ items: page, nextCursor }` (lacks key `'pagination'`).
   - Consequently, `EnvelopeInterceptor` wraps the output as `{ data: { items: [...], nextCursor: ... } }`.
   - The frontend Axios service (`content.service.ts`) expects `{ data: Problem[], pagination: { nextCursor, hasMore } }`, resulting in malformed client pagination structures.
2. **`platform_moderator` Permission Mismatch:**
   - `ContentService.createProblem` allows roles `['content_manager', 'platform_admin', 'platform_moderator']`.
   - However, `PermissionGuard` checks `@RequirePermission('create:content')`, which `platform_moderator` **does not have** in `ROLE_PERMISSIONS` ([`src/shared/auth/permission.guard.ts:34-36`](file:///E:/Dezolver%20project/dezlovers-main/Final%20product/dezlovers-main(cuurent%20update)/src/shared/auth/permission.guard.ts#L34-L36)), causing immediate HTTP 403 blocks.
3. **`CsvImportConsumer` Schema Crash:**
   - `CsvImportConsumer.importProblems` passes `tenantId` and `description` to `this.prisma.content.problem.create`, and omits required fields `slug` and `statementMd`, causing runtime crashes on CSV imports.

### P.4 Product Decision Log

| Decision ID | Open Question / Gap | Options Considered | Business Impact | Recommended Target Direction | Decision Owner |
|---|---|---|---|---|---|
| `DEC-PRB-01` | Should institutional colleges be allowed to author private problems? | A) Keep problems 100% global.<br>B) Add `tenant_id` (nullable) to allow both global and college-private problems. | Critical for university exam confidentiality. | **Option B:** Add nullable `tenant_id` to `content.problems` with tenant scoping middleware. | Academic Steering Committee |
| `DEC-PRB-02` | How should post-publication problem edits be handled? | A) Allow unrestricted edits.<br>B) Create immutable version snapshots on edit.<br>C) Retain immutable problems. | Changing test cases on active exams causes grading inconsistencies. | **Option B:** Implement problem versioning matching Room versioning pattern. | Principal Architect |
| `DEC-PRB-03` | Should non-atomic test case creation in frontend be replaced? | A) Keep sequential HTTP calls.<br>B) Support atomic bulk test case creation in `POST /problems`. | Network disconnects currently leave orphan problems with partial test cases. | **Option B:** Update `POST /problems` to accept an optional `testCases` array within a Prisma transaction. | Frontend / Backend Leads |

### P.5 Technical Debt & Architectural Recommendations
1. **Implement Complete Problem CRUD:** Add `PATCH /problems/:id` and `DELETE /problems/:id` with integrity checks preventing deletion if referenced by active assessments.
2. **Implement Test Case CRUD:** Add `PATCH /problems/:id/test-cases/:tcId` and `DELETE /problems/:id/test-cases/:tcId` for draft problems.
3. **Fix `EnvelopeInterceptor` Consistency:** Standardize `listProblems` return payload to `{ items, pagination: { nextCursor, hasMore } }`.
4. **Refactor `CsvImportConsumer`:** Align CSV parsing with current `Problem` schema (`statementMd`, `slug`, remove `tenantId`).

---

## SECTION Q: MODULE REVIEW GATE CHECKLIST

### Module Completion Sign-Off Checklist
- [x] 1. All relevant legacy source files were inspected directly (no doc assumptions).
- [x] 2. All relevant functions were identified, numbered, and traced end-to-end (`FN-PRB-001` through `FN-PRB-007`).
- [x] 3. All API endpoints were cataloged with DTOs, parameters, and error codes (`GET /problems`, `GET /problems/:slug`, `POST /problems`, `POST /problems/:id/test-cases`, `POST /problems/:id/publish`, `POST /search`).
- [x] 4. All database entities, columns, constraints, and indexes were documented (`content.problems`, `content.problem_test_cases`).
- [x] 5. All frontend pages, components, states, and user interactions were traced (`ProblemsPage`, `ProblemDetailPage`, `ProblemCreatePage`, `ProblemPanel`, `ProblemExamples`).
- [x] 6. All mathematical formulas and algorithms were verified from code (slugification, base64 cursor pagination).
- [x] 7. All permissions, RBAC claims, and tenant isolation checks were inspected (verified global catalog scope).
- [x] 8. All cross-module dependencies and external integrations were verified (Judge0 limits, Assessment module linkages).
- [x] 9. All edge cases, failure modes, and recovery behaviors were documented (zero-test-case publication block, published mutation locks).
- [x] 10. Every function has testable Given/When/Then Acceptance Criteria.
- [x] 11. An exhaustive QA scenario matrix (Positive, Boundary, Security, Fault) is included (`TC-PRB-01` to `TC-PRB-15`).
- [x] 12. Every assertion includes exact file path, class/method, and line number citations.
- [x] 13. All unverified assumptions have been eliminated or marked UNVERIFIABLE.
- [x] 14. Product decisions, open questions, and technical recommendations are segregated (`DEC-PRB-01` to `DEC-PRB-03`).
- [x] 15. The specification strictly conforms to SECTIONS A through Q of `DEZ-STD-FS-001`.
- [x] 16. Git status verified: Zero modifications made to application source code or schemas.

