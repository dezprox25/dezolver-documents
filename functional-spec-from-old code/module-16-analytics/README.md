# Module 16 — Student Analytics

## 1. Module Executive Summary

Module 16 establishes the functional requirements, operational state, and architectural specifications for the Student Analytics, Learning Progress, and Performance Metrics subsystem of the Dezolver platform.

A thorough forensic audit across all 11 database schemas, backend NestJS services, and React frontend clients reveals that **a dedicated backend analytics service is 100% NOT IMPLEMENTED**. Instead, the platform relies on an asymmetric collection of fragmented learning path progress records, client-side in-memory mock services, and browser-computed real-time statistical reductions:

1. **Total Absence of a Dedicated Analytics Backend Subsystem:** Across all 11 Prisma schemas in `prisma/` and all controllers/services in `src/`, there is **zero occurrences of an analytics schema, controller, service, or repository**. No database tables exist for daily active time, study streaks, concept mastery, coding velocity, or learning retention curves.
2. **Asymmetric Learning Path Progress Persistence:** The sole persistent student progress mechanism in the platform exists within the `paths` domain:
   - `paths.path_progress` (`PathProgress` model in `prisma/paths/schema.prisma:50-67`) persists `stepsCompleted`, `stepsTotal`, `percentComplete`, `lastActivityAt`, and `completedAt`.
   - `paths.room_progress` is an unmapped PostgreSQL table managed via raw SQL (`$executeRaw` in `src/paths/controllers/me-paths.controller.ts:83, 114, 145`) storing room-level states (`in_progress`, `skipped`, `completed`).
   - `ProgressAggregatorConsumer` (`src/paths/consumers/progress-aggregator.consumer.ts`) executes an asynchronous BullMQ worker on `RoomProgressUpdated`, recomputing path completion percentages via raw SQL and upserting `path_progress`.
3. **Purely Mock KPI & Activity Service:** The primary dashboard analytics interface `dashboardService` (`frontend/src/services/api/dashboard.service.ts:127-145`) makes zero HTTP requests to the backend. It returns hardcoded mock in-memory data via `setTimeout(300)`, accompanied by an explicit code comment: *"The mock implementation returns static data; replace queryFn with a real API call once the aggregation endpoint exists."*
4. **Client-Side Real-Time Math Reductions:** Every analytics screen across the React application derives charts and metrics on-the-fly in browser JavaScript, accompanied by explicit warning banners in the UI:
   - `AssessmentAnalyticsPage.tsx`: Fetches raw personal submissions (`GET /me/submissions?assessment=:id`) and computes score averages, verdict histograms, and language breakdowns in React RAM. Displays banner: *"Analytics show your own submissions only... Class-wide analytics require a dedicated backend analytics endpoint which does not exist."*
   - `EventAnalyticsPage.tsx`: Fetches raw registration rows (`GET /events/:id/registrations`) and counts participant statuses client-side. Displays banner: *"Analytics are derived from the registrations API in real-time... No dedicated analytics endpoint exists in the backend."*
   - `PlatformAnalyticsPage.tsx`: Fetches raw tenant records (`GET /tenants?limit=200`) and aggregates status/growth metrics client-side. Displays banner: *"No dedicated analytics API is available in the backend... Event-level usage analytics require a dedicated aggregation endpoint."*
   - `FacultyProgressPage.tsx`: Displays banner: *"Backend Limitation: The paths module is in skeleton state. Cohort-level progress analytics require a service layer that has not yet been implemented."*
5. **Broken Global Competitive Standing Queries:** While `PlatformRatingService` (`src/events/services/platform-rating.service.ts`) updates individual ELO ratings in `identity.persons.platformRating` upon event completion, the public query endpoint `GET /leaderboard/global` (`src/events/controllers/leaderboard.controller.ts:13`) fails completely. `EventsRepository.findGlobalLeaderboard` attempts to query `(this.prisma.events as any).platformRating?.findMany(...)`, which evaluates to `undefined` and returns an empty list `[]`. Similarly, `GET /persons/:id/standing` returns `null` and throws `404 NotFoundException`.
6. **Non-Existent Student Metric Primitives:** Zero tracking of daily study streaks, calendar commit heatmaps, active problem-solving time, or topic mastery (e.g., Arrays, Dynamic Programming, Graphs) exists anywhere in the architecture.

---

## 2. Directory Layout & Document Navigation

```
docs/functional-spec/module-16-analytics/
├── README.md               # Executive summary, forensic findings, and architecture guide (this file)
└── SPECIFICATION.md        # Comprehensive 17-Section (A–Q) legacy specification & traceability matrix
```

- **Detailed Functional Specification:** [`SPECIFICATION.md`](./SPECIFICATION.md)
- **Authoring Standard:** [`FUNCTIONAL-SPECIFICATION-STANDARD.md`](../../FUNCTIONAL-SPECIFICATION-STANDARD.md) (`DEZ-STD-FS-001`)
- **Master Index & Registry:** [`INDEX.md`](../INDEX.md)

---

## 3. Subsystem Architecture & Data Flow

```
    ┌────────────────────────────────────────────────────────┐
    │                STUDENT LEARNING ACTIONS                │
    │  - Start/Skip/Complete Room (/me/rooms/:id/*)          │
    │  - Submit Code to Assessment (/assessments/:id/submit) │
    │  - Compete in Event (/events/:id/submissions)          │
    └───────────┬────────────────────────────────┬───────────┘
                │                                │
                ▼                                ▼
    ┌────────────────────────┐       ┌────────────────────────┐
    │  RAW SQL PROGRESS DB   │       │  ASSESSMENT DB TABLES  │
    │  paths.room_progress   │       │ assessment.submissions │
    └───────────┬────────────┘       └───────────┬────────────┘
                │                                │
                ▼                                │
    ┌────────────────────────┐                   │
    │ Outbox: RoomProgress   │                   │
    └───────────┬────────────┘                   │
                │                                │
                ▼                                │
    ┌────────────────────────┐                   │
    │ ProgressAggregator     │                   │
    │ (BullMQ Consumer)      │                   │
    │ Computes path %        │                   │
    └───────────┬────────────┘                   │
                │                                │
                ▼                                │
    ┌────────────────────────┐                   │
    │   paths.path_progress  │                   │
    └───────────┬────────────┘                   │
                │                                │
                └────────────────┬───────────────┘
                                 │
                                 ▼
    ┌────────────────────────────────────────────────────────┐
    │              FRONTEND CLIENT-SIDE REDUCTIONS           │
    │  - MyProgressPage: queries /me/paths                   │
    │  - AssessmentAnalyticsPage: queries /me/submissions    │
    │    and computes avg/best/histograms in browser RAM     │
    │  - DashboardPage: queries mock dashboardService        │
    │    (hardcoded objects with setTimeout)                 │
    └────────────────────────────────────────────────────────┘
```

---

## 4. Forensic Inventory & Scope

| Component / Layer | Source File Path | Forensic Role & Behavior |
|---|---|---|
| **Path Progress Schema** | `prisma/paths/schema.prisma:50-67` | Model `PathProgress` (`paths.path_progress`) storing aggregate steps and completion percentage. |
| **Room Progress Table** | Direct PostgreSQL DDL (`paths.room_progress`) | Unmapped raw SQL table storing individual room completion records. |
| **Personal Paths Controller** | `src/paths/controllers/me-paths.controller.ts` | Exposes `/me/paths`, `/me/paths/:id/progress`, `/me/paths/:id/next-step`, and `/me/rooms/:roomId/*`. |
| **Progress Aggregator Worker** | `src/paths/consumers/progress-aggregator.consumer.ts` | BullMQ worker that calculates path percentage completion on `RoomProgressUpdated`. |
| **Leaderboard Controller** | `src/events/controllers/leaderboard.controller.ts:13, 25` | Exposes `/leaderboard/global` and `/persons/:id/standing`. |
| **Leaderboard Repository** | `src/events/repositories/events.repository.ts:104, 112` | Broken repository queries targeting non-existent `prisma.events.platformRating`. |
| **Platform ELO Service** | `src/events/services/platform-rating.service.ts` | Pairwise ELO calculation engine writing to `identity.persons.platformRating` and Redis. |
| **Frontend Dashboard Service** | `frontend/src/services/api/dashboard.service.ts:127` | Purely mock client service returning static KPI metrics via `setTimeout(300)`. |
| **Frontend Progress Service** | `frontend/src/services/api/progress.service.ts` | Client API wrapper for personal path progress and room completion endpoints. |
| **Student Progress Page** | `frontend/src/pages/paths/MyProgressPage.tsx` | Student UI rendering enrolled paths, progress bars, and achievement certificates. |
| **Faculty Analytics Page** | `frontend/src/pages/paths/FacultyProgressPage.tsx` | Faculty view displaying backend skeleton limitation banner. |
| **Assessment Analytics Page**| `frontend/src/pages/assessments/AssessmentAnalyticsPage.tsx`| Computes assessment attempt statistics and verdict distributions in browser RAM. |
| **Event Analytics Page** | `frontend/src/pages/events/EventAnalyticsPage.tsx` | Computes participant registration counts in browser RAM. |
| **Platform Analytics Page** | `frontend/src/pages/platform/analytics/PlatformAnalyticsPage.tsx` | Computes tenant lifecycle and growth statistics in browser RAM. |

---

## 5. Critical Defects & Discrepancies Summary

| Defect ID | Severity | Location | Forensic Description & Blast Radius |
|---|---|---|---|
| **DEF-16-01** | `CRITICAL` | Backend Architecture (`src/`, `prisma/`) | **Complete Absence of Analytics Backend Subsystem**: Zero dedicated analytics schemas, tables, services, or controllers exist in the backend. 100% of dashboard KPIs, assessment statistics, and platform growth metrics are either hardcoded mocks or computed client-side in browser RAM. |
| **DEF-16-02** | `CRITICAL` | `src/events/repositories/events.repository.ts:104-110` | **Broken Global Leaderboard Database Query**: `findGlobalLeaderboard` executes `(this.prisma.events as any).platformRating?.findMany(...)`. Because `platformRating` does not exist in `prisma.events`, it evaluates to `undefined` and returns `[]`. Global leaderboard queries always return empty. |
| **DEF-16-03** | `HIGH` | `src/events/repositories/events.repository.ts:112-114` | **Broken Person Standing Query**: `findPersonStanding` queries nonexistent `platformRating` on `prisma.events`, returning `null`. Consequently, `GET /persons/:id/standing` throws a `404 NotFoundException` for all users. |
| **DEF-16-04** | `HIGH` | `src/paths/controllers/me-paths.controller.ts:83, 114, 145` | **Unmapped Raw SQL Room Progress Table**: `room_progress` table is managed via raw `$executeRaw` queries without Prisma ORM schema definition, validation, migrations, or relational integrity constraints. |
| **DEF-16-05** | `MEDIUM` | `frontend/src/services/api/dashboard.service.ts:132-144` | **Hardcoded In-Memory Dashboard Mock**: Dashboard KPIs and activity feeds do not query the backend; they return static arrays after a 300ms artificial delay, misleading users with static synthetic statistics. |
| **DEF-16-06** | `MEDIUM` | `frontend/src/pages/assessments/AssessmentAnalyticsPage.tsx:143-147` | **Absence of Class-Wide Assessment Metrics**: Assessment analytics page displays an explicit banner stating class-wide analytics are impossible because no aggregation endpoint exists; students and faculty only see the logged-in user's personal attempts. |
| **DEF-16-07** | `LOW` | Codebase-wide (`src/`, `frontend/src/`) | **Zero Study Streak / Velocity Tracking**: Daily login streaks, activity heatmaps, problem-solving speed, and concept mastery metrics are completely missing from the platform. |

---

## 6. Verification & Traceability Sign-off

- **Negative Verification:** Verified absence of analytics models across all 11 database schemas, confirmed zero analytics controllers in `src/`, and identified explicit client limitation banners across all analytics pages.
- **Traceability Matrix:** Fully documented in [`SPECIFICATION.md`](./SPECIFICATION.md#section-q-master-verification-traceability-matrix).
- **Adherence Standard:** Strictly complies with `DEZ-STD-FS-001`.

