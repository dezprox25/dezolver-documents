# Module 11 — Contests (11.1–11.6)
## Comprehensive Functional Specification & Forensic Reconstruction

**Standard:** [`FUNCTIONAL-SPECIFICATION-STANDARD.md`](../../FUNCTIONAL-SPECIFICATION-STANDARD.md) (`DEZ-STD-FS-001`)  
**Domain:** Competitive Programming Contests, Technical Events, Live Leaderboards, and ELO Rating Engine  
**Legacy Package:** `src/events/`, `prisma/events/`, `src/billing/`, `frontend/src/pages/events/`  
**Reconstruction State:** `COMPLETED & FORENSICALLY VERIFIED`  
**Investigation Mode:** Strict Forensic Codebase Extraction (Zero Hallucination, Zero Assumptions)  

---

## SECTION A: Module Overview & Scope Boundaries

### A.1 Functional Definition
Module 11 governs the complete lifecycle of competitive programming contests, hackathons, workshops, and technical webinars on the Dezolver platform. It encompasses event authoring and scheduling, participant registration and fee processing, real-time code submission routing via Judge0, dynamic Redis-backed leaderboards with live WebSocket rank broadcasting, post-contest automated ranking, pairwise ELO rating recalculation, and certificate eligibility determination.

### A.2 Sub-Module Structure
As mandated by the Dezolver Functional Specification Standard, Module 11 is reconstructed across six core functional domains:
- **11.1 Contest Creation & Scheduling:** Kind selection (`competition`, `workshop`, `hackathon`, `webinar`), multi-state lifecycle machine, drafting, automated time-based state transitions, parameter updates, cancellation, and time extensions.
- **11.2 Contest Problems & Test Cases:** Problem selection, point assignments, execution constraints, and forensic schema analysis of problem storage.
- **11.3 Registration & Eligibility:** Free and paid registration workflows, capacity management, Razorpay payment order generation, webhook confirmations, and unregistration.
- **11.4 Contest Window & Real-time Submissions:** Live window opening, submission routing via the assessment proxy pipeline, Judge0 CE execution, and real-time state broadcasts.
- **11.5 Scoring, Penalties & Redis Leaderboards:** Real-time scoring, Redis ZSET mechanics (`leaderboard:event:${id}`), inverted rank defect analysis (`zrank` vs `zrevrank`), and ICPC penalty evaluation.
- **11.6 Post-Contest Analytics & Results Archival:** Automated final grading, top-3 certificate eligibility enforcement, pairwise ELO rating calculation ($K=32$), and global leaderboards.

### A.3 Scope Boundaries

#### In-Scope (Implemented & Traced in Source Code)
- Complete database schema for `events.events`, `events.registrations`, and `events.event_results`.
- Automated cron state watcher running every 30 seconds (`EventStateWatcherService`).
- Registration capacity gating and duplicate registration prevention.
- Paid event registration integration via Razorpay (`EventOrdersController` and `RazorpayClient`).
- Automated payment refunds upon event cancellation (`EventCancelledConsumer`).
- Real-time event result upsert and Redis sorted set leaderboard updates on accepted submissions (`SubmissionGradedConsumer`).
- Real-time WebSocket event state broadcasts (`RealtimeEventsConsumer`).
- Automated post-contest ranking cron running every 5 minutes (`EventFinalGradingService`).
- Pairwise ELO platform rating computation updating `identity.persons.platformRating` (`PlatformRatingService`).
- Automated 24-hour and 1-hour pre-contest email reminders (`RegistrationReminderService`).
- Nine frontend contest pages (`EventCreatePage`, `EventDetailPage`, `EventLivePage`, `EventLeaderboardPage`, `EventResultsPage`, etc.).

#### Out-of-Scope (Proven NOT IMPLEMENTED in Legacy Code)
- **Database Problem Association:** Table `events.events` has **no foreign key, join table, or `config` JSONB column** to store problem IDs. Problem linking is absent from the PostgreSQL database.
- **ICPC Penalty System:** No cumulative time tracking, 20-minute wrong submission penalties, or tie-breaker time logic exists in the backend.
- **Direct Event Submission Endpoint:** `POST /events/:id/submissions` does not evaluate code; it returns a stub message directing the client to use the assessments endpoint.
- **Dynamic Certificate Thresholds:** Certificate eligibility is hardcoded to `rank < 3` (top 3 participants only); configurable score or percentile thresholds do not exist.
- **Audience Scope Enforcement:** Neither `audienceScope` nor cohort eligibility is validated by `EventsService.register`.

---

## SECTION B: Architecture & Dependency Topology

### B.1 Architectural System Topology

```
                                  ┌──────────────────────────────┐
                                  │       EventsController       │
                                  │ (src/events/controllers)     │
                                  └──────────────┬───────────────┘
                                                 │
                                                 ▼
                                  ┌──────────────────────────────┐
                                  │        EventsService         │
                                  │  (src/events/services)       │
                                  └──────┬───────────────┬───────┘
                                         │               │
                                         ▼               ▼
                     ┌───────────────────────┐       ┌───────────────────────┐
                     │    EventsRepository   │       │     PrismaService     │
                     │ (src/events/repos)    │       │ (shared/prisma)       │
                     └───────────┬───────────┘       └───────────┬───────────┘
                                 │                               │
                                 ▼                               ▼
                     ┌───────────────────────┐       ┌───────────────────────┐
                     │     events.events     │       │ platform.outbox_events│
                     │  events.registrations │       └───────────┬───────────┘
                     │  events.event_results │                   │
                     └───────────────────────┘                   │
                                                                 │
      ┌──────────────────────────────────────────────────────────┴─────────────────────────┐
      │                                                                                    │
      ▼                                                                                    ▼
┌──────────────────────────────┐                                            ┌──────────────────────────────┐
│   SubmissionGradedConsumer   │                                            │    PlatformRatingService     │
│  (src/events/consumers)      │                                            │  (src/events/services)       │
│  - Reads submission metadata │                                            │  - Listens to EventCompleted │
│  - Upserts EventResult       │                                            │  - Pairwise ELO calculation  │
│  - ZADD leaderboard:event    │                                            │  - Updates Person.rating     │
│  - Redis PubSub broadcast    │                                            │  - ZADD leaderboard:platform │
└──────────────┬───────────────┘                                            └──────────────────────────────┘
               │
               ▼
┌──────────────────────────────┐
│  Redis Sorted Sets & PubSub  │
│  - leaderboard:event:${id}   │
│  - dezolver#event:${id}:...  │
└──────────────────────────────┘
```

### B.2 Dependency Inventory

#### Internal Module Dependencies
- **`src/events/events.module.ts`**: Declares controllers, repositories, consumers, and cron services.
- **`src/shared/prisma/prisma.service.ts`**: Multi-schema database access (`events`, `identity`, `platform`, `billing`, `assessment`).
- **`src/shared/redis/redis.service.ts`**: Redis sorted sets and pub/sub message dispatch.
- **`src/shared/outbox/consumer-registry.ts`**: Outbox event routing for `SubmissionGraded`, `EventCompleted`, `EventRegistrationPaid`, and `EventCancelled`.
- **`src/billing/controllers/event-orders.controller.ts`**: Generates Razorpay payment orders.
- **`src/billing/consumers/event-cancelled.consumer.ts`**: Issues automatic refunds on event cancellation.
- **`src/websocket/consumers/realtime-events.consumer.ts`**: Emits real-time WebSocket state changes to event rooms.
- **`src/shared/health/health.controller.ts`**: Exposes `GET /time` for synchronized client countdown timers.

#### External Runtime Dependencies
- **Redis 7.0+**: Real-time sorted set leaderboard storage and pub/sub broadcasting.
- **BullMQ**: Queueing infrastructure for email reminders (`email` queue).
- **Razorpay API**: Payment gateway client for event orders and automated refunds.
- **Judge0 CE Runner**: Remote code execution sandbox for competition submissions.

---

## SECTION C: Data Models & Persistence Mechanics

### C.1 Database Schema (`prisma/events/schema.prisma`)

```prisma
datasource db {
  provider = "postgresql"
  url       = env("DATABASE_URL")
  directUrl = env("DIRECT_URL")
  schemas  = ["events"]
}

model Event {
  id                   String    @id @default(dbgenerated("gen_random_uuid()")) @db.Uuid
  tenantId             String    @map("tenant_id") @db.Uuid
  kind                 String    @db.VarChar(30)
  title                String    @db.VarChar(255)
  description          String?   @db.Text
  startsAt             DateTime  @map("starts_at") @db.Timestamptz
  endsAt               DateTime  @map("ends_at") @db.Timestamptz
  maxParticipants      Int?      @map("max_participants")
  visibility           String    @default("tenant_only") @db.VarChar(30)
  paid                 Boolean   @default(false)
  registrationFeePaise Int?      @map("registration_fee_paise")
  status               String    @default("draft") @db.VarChar(20)
  createdByUserId      String    @map("created_by_user_id") @db.Uuid
  createdAt            DateTime  @default(now()) @map("created_at") @db.Timestamptz
  updatedAt            DateTime  @updatedAt @map("updated_at") @db.Timestamptz

  registrations Registration[]
  results       EventResult[]

  @@index([tenantId, kind, status])
  @@map("events")
  @@schema("events")
}

model Registration {
  id           String    @id @default(dbgenerated("gen_random_uuid()")) @db.Uuid
  eventId      String    @map("event_id") @db.Uuid
  userId       String    @map("user_id") @db.Uuid
  tenantId     String    @map("tenant_id") @db.Uuid
  teamName     String?   @map("team_name") @db.VarChar(120)
  status       String    @default("registered") @db.VarChar(30)
  registeredAt DateTime  @default(now()) @map("registered_at") @db.Timestamptz
  cancelledAt  DateTime? @map("cancelled_at") @db.Timestamptz

  event Event @relation(fields: [eventId], references: [id])

  @@unique([eventId, userId])
  @@index([tenantId, userId])
  @@map("registrations")
  @@schema("events")
}

model EventResult {
  id                   String    @id @default(dbgenerated("gen_random_uuid()")) @db.Uuid
  eventId              String    @map("event_id") @db.Uuid
  userId               String    @map("user_id") @db.Uuid
  tenantId             String    @map("tenant_id") @db.Uuid
  rank                 Int?
  score                Float?
  certificateEligible  Boolean   @default(false) @map("certificate_eligible")
  metadata             Json      @default("{}")
  createdAt            DateTime  @default(now()) @map("created_at") @db.Timestamptz

  event Event @relation(fields: [eventId], references: [id])

  @@unique([eventId, userId], map: "eventId_userId")
  @@index([tenantId])
  @@map("event_results")
  @@schema("events")
}
```

### C.2 Field-by-Field Technical Specifications

#### Table: `events.events`
| Column Name | Database Type | Nullable | Default | Description & Forensic Details |
|---|---|:---:|---|---|
| `id` | `UUID` | No | `gen_random_uuid()` | Primary Key. Immutable unique event identifier. |
| `tenant_id` | `UUID` | No | — | Foreign Key to tenant owning/hosting the event. |
| `kind` | `VARCHAR(30)` | No | — | Event type: `'competition'`, `'workshop'`, `'hackathon'`, `'webinar'`. |
| `title` | `VARCHAR(255)` | No | — | Public title of the event (min 3 chars). |
| `description` | `TEXT` | Yes | `NULL` | Formatted markdown overview or instructions. |
| `starts_at` | `TIMESTAMPTZ` | No | — | Planned event start timestamp. |
| `ends_at` | `TIMESTAMPTZ` | No | — | Planned event conclusion timestamp. |
| `max_participants` | `INT` | Yes | `NULL` | Maximum allowed registrations. If null, unlimited. |
| `visibility` | `VARCHAR(30)` | No | `'tenant_only'` | Scope: `'tenant_only'`, `'tenant_open'`, `'public'`. |
| `paid` | `BOOLEAN` | No | `false` | Flag indicating whether a registration fee is required. |
| `registration_fee_paise` | `INT` | Yes | `NULL` | Registration fee in Indian Paise (e.g. 50000 = ₹500.00). |
| `status` | `VARCHAR(20)` | No | `'draft'` | Current state in the 7-state lifecycle machine. |
| `created_by_user_id` | `UUID` | No | — | User ID of the staff member who created the event. |
| `created_at` | `TIMESTAMPTZ` | No | `now()` | Audit record creation timestamp. |
| `updated_at` | `TIMESTAMPTZ` | No | Auto `@updatedAt` | Automatic update timestamp. |

#### Table: `events.registrations`
| Column Name | Database Type | Nullable | Default | Description & Constraints |
|---|---|:---:|---|---|
| `id` | `UUID` | No | `gen_random_uuid()` | Primary Key. |
| `event_id` | `UUID` | No | — | Foreign Key referencing `events.events(id)`. |
| `user_id` | `UUID` | No | — | User ID of the registered participant. |
| `tenant_id` | `UUID` | No | — | Tenant ID of the participant. |
| `team_name` | `VARCHAR(120)` | Yes | `NULL` | Optional team designation for team events. |
| `status` | `VARCHAR(30)` | No | `'registered'` | Registration status: `'registered'`, `'cancelled'`, `'pending_payment'`. |
| `registered_at` | `TIMESTAMPTZ` | No | `now()` | Timestamp of registration. |
| `cancelled_at` | `TIMESTAMPTZ` | Yes | `NULL` | Timestamp if the participant withdrew. |

#### Table: `events.event_results`
| Column Name | Database Type | Nullable | Default | Description & Constraints |
|---|---|:---:|---|---|
| `id` | `UUID` | No | `gen_random_uuid()` | Primary Key. |
| `event_id` | `UUID` | No | — | Foreign Key referencing `events.events(id)`. |
| `user_id` | `UUID` | No | — | User ID of the participant. |
| `tenant_id` | `UUID` | No | — | Tenant ID of the participant. |
| `rank` | `INT` | Yes | `NULL` | Official standing/rank (1-based index). |
| `score` | `FLOAT` | Yes | `NULL` | Total points or test case percentage earned. |
| `certificate_eligible`| `BOOLEAN` | No | `false` | Flag indicating certificate award eligibility. |
| `metadata` | `JSON` | No | `'{}'` | JSON metadata (`lastSubmissionId`, run logs). |
| `created_at` | `TIMESTAMPTZ` | No | `now()` | Record creation timestamp. |

---

## SECTION D: API & Endpoint Specifications

### D.1 Comprehensive Route Inventory

| Method | Route Path | RBAC Permission | Controller & Line | Purpose & Description |
|---|---|---|---|---|
| `GET` | `/events` | `read:cohort` | `EventsController:32` | Paginated listing of events for tenant. |
| `GET` | `/events/:id` | `read:cohort` | `EventsController:53` | Retrieves detailed event metadata. |
| `POST` | `/events` | `create:event` | `EventsController:66` | Creates an event in `draft` status. |
| `POST` | `/events/:id/publish` | `create:event` | `EventsController:79` | Manually transitions event from `draft` to `published`. |
| `PATCH`| `/events/:id` | `manage:event` | `EventsController:111` | Updates title, times, or capacity (draft only). |
| `POST` | `/events/:id/cancel` | `manage:event` | `EventsController:125` | Cancels event and triggers automatic refunds. |
| `POST` | `/events/:id/extend` | `manage:event` | `EventsController:139` | Extends event end time during live state. |
| `POST` | `/events/:id/registrations` | `read:cohort` | `EventsController:94` | Registers current user for an event. |
| `DELETE`| `/events/:id/registrations/me`| `read:cohort` | `EventsController:163` | Unregisters current user from an event. |
| `GET` | `/events/:id/registrations` | `manage:event` | `EventsController:150` | Organizer roster view of all registrations. |
| `POST` | `/events/:id/submissions` | `read:cohort` | `EventsController:176` | Stub submission endpoint pointing to assessments API. |
| `GET` | `/events/:id/leaderboard` | `read:cohort` | `EventsController:186` | Retrieves results ordered by rank ascending. |
| `GET` | `/events/:id/me/standing` | `read:cohort` | `EventsController:196` | Retrieves caller's specific result and rank. |
| `POST` | `/events/:id/results` | `manage:event` | `EventsController:210` | Manual result upsert by contest administrator. |
| `POST` | `/event-registration-orders` | `read:cohort` | `EventOrdersController:34` | Generates Razorpay order for paid registration. |
| `GET` | `/leaderboard/global` | `read:cohort` | `LeaderboardController:18` | Top-N platform rating global leaderboard. |
| `GET` | `/persons/:id/standing` | `read:cohort` | `LeaderboardController:31` | Global rank and PlatformRating for a person. |
| `GET` | `/time` | Public | `HealthController:74` | Server timestamp for synchronized countdown clocks. |

---

## SECTION E: State Machines & Transition Tables

### E.1 Event Lifecycle State Machine

The event lifecycle is governed by an automated, cron-driven state machine evaluated every 30 seconds by `EventStateWatcherService` (`src/events/services/event-state-watcher.service.ts`).

```
                ┌────────────────────────────────────────────────────────┐
                │                         DRAFT                          │
                └───────────────────────────┬────────────────────────────┘
                                            │ POST /events/:id/publish
                                            ▼
                ┌────────────────────────────────────────────────────────┐
                │                       PUBLISHED                        │
                └───────────────────────────┬────────────────────────────┘
                                            │ Cron: now >= startsAt - 7d
                                            ▼
                ┌────────────────────────────────────────────────────────┐
                │                   REGISTRATION_OPEN                    │
                └───────────────────────────┬────────────────────────────┘
                                            │ Cron: now >= startsAt
                                            ▼
                ┌────────────────────────────────────────────────────────┐
                │                  REGISTRATION_CLOSED                   │
                └───────────────────────────┬────────────────────────────┘
                                            │ Cron: now >= startsAt
                                            ▼
                ┌────────────────────────────────────────────────────────┐
                │                          LIVE                          │
                └───────────────────────────┬────────────────────────────┘
                                            │ Cron: now >= endsAt
                                            ▼
                ┌────────────────────────────────────────────────────────┐
                │                        GRADING                         │
                └───────────────────────────┬────────────────────────────┘
                                            │ Cron: EventFinalGradingService
                                            │ (now >= endsAt + 30m)
                                            ▼
                ┌────────────────────────────────────────────────────────┐
                │                       COMPLETED                        │
                └────────────────────────────────────────────────────────┘
```

### E.2 State Transition Matrix & Conditions

| Current State | Target State | Trigger Mechanism | Condition / Evaluation Logic | Outbox Event Emitted |
|---|---|---|---|---|
| `draft` | `published` | `POST /events/:id/publish` | Manual organizer call; requires `status === 'draft'`. | None |
| `published` | `registration_open` | Automated Cron (30s) | `now >= new Date(startsAt.getTime() - 7 * 86400000)` | None |
| `registration_open` | `registration_closed`| Automated Cron (30s) | `now >= startsAt` | None |
| `registration_closed`| `live` | Automated Cron (30s) | `now >= startsAt` | `EventLive` (`{ eventId }`) |
| `live` | `grading` | Automated Cron (30s) | `now >= endsAt` | None |
| `grading` | `completed` | Automated Cron (5m) | `endsAt < now - 30m` (`EventFinalGradingService`) | `EventCompleted` (`{ eventId, finalRanksAssigned }`) |
| Any active state | `cancelled` | `POST /events/:id/cancel` | `status in ['draft', 'published', 'registration_open', 'registration_closed', 'live']` | `EventCancelled` (`{ eventId, reason }`) |
| `live` / `reg_open` | Same (Extended) | `POST /events/:id/extend` | `newEndsAt > endsAt` | `EventExtended` (`{ eventId, newEndsAt }`) |

---

## SECTION F: Sub-Module Forensic Investigations (11.1–11.6)

### 11.1 Contest Creation & Scheduling
- **Creation Endpoint:** `POST /events` requires permission `create:event`.
- **RBAC Discrepancy:** In `permission.guard.ts`, `coordinator`, `college_admin`, and `platform_admin` possess `create:event`. Faculty users hold `create:assessment`, but **lack `create:event`**, preventing faculty from creating contests via API unless granted administrative roles.
- **Duration Constraints:** `CreateEventDto` validates `startsAt` and `endsAt` as ISO date strings. It does not validate that `endsAt > startsAt` in class-validator (handled at runtime).
- **Modification Guard:** `PATCH /events/:id` rejects updates with `400 BadRequestException({ code: 'not_draft' })` if the event has advanced past `draft`.

### 11.2 Contest Problems & Test Cases
- **Database Reality:** There is **NO `EventProblem` or `ContestProblem` join table** across any of the 11 Prisma schemas.
- **Missing `config` Column:** While frontend types (`EventConfig`, `CompetitionProblem`) define problem mappings (`problemId`, `points`, `order`), table `events.events` in `prisma/events/schema.prisma` possesses **no `config` JSONB column**. Problem configurations cannot be persisted in the events database.
- **Submission Indirection:** `POST /events/:id/submissions` does not accept code. It returns:
  ```json
  {
    "submitted": true,
    "eventId": "...",
    "message": "Use assessments/:id/submissions with metadata.eventId"
  }
  ```
  Contest solutions are executed by proxying through `AssessmentService.submitCode` while tagging the submission with `{ metadata: { eventId } }`.

### 11.3 Registration & Eligibility
- **Free Events:** `EventsService.register` checks `event.status === 'published'`.
  > [!WARNING]
  > **Defect:** `EventsService.register` strictly checks `if ((event as any).status !== 'published') throw new BadRequestException({ code: 'event_not_open' })`. When `EventStateWatcherService` advances the event to `registration_open`, registration requests **fail with `400 event_not_open`**!
- **Capacity Limits:** If `maxParticipants` is configured, `regs.length >= maxParts` triggers `400 BadRequestException({ code: 'event_full' })`.
- **Paid Events Integration:**
  - Client calls `POST /event-registration-orders` passing `eventId` and `amountPaise`.
  - `RazorpayClient.createOrder` creates an order with receipt `evt_${eventId}_${Date.now()}` and notes `{ eventId, userId }`.
  - Razorpay webhook emits `EventRegistrationPaid`.
  - `EventRegistrationConsumer` marks `events.registrations.status = 'registered'` with idempotency guarded by `platform.consumed_events`.
- **Automated Cancellation Refunds:**
  - When an event is cancelled, `EventsService.cancel` emits `EventCancelled`.
  - `EventCancelledConsumer` catches this event, queries captured payments in `billing.payments`, calls `razorpay.refundPayment()`, and marks payments as `refunded`.

### 11.4 Contest Window & Real-time Submissions
- **Opening the Window:** When `startsAt` is reached, `EventStateWatcherService` sets status to `live` and emits `EventLive`.
- **Real-time State Fanout:** `RealtimeEventsConsumer` catches `EventLive` and calls `RealtimeBroadcaster.toEventChannel(eventId, 'event:state', { eventId, status: 'live' })`.
- **Clock Synchronization:** Clients synchronize countdown clocks against `GET /time`, which returns `{ serverNow: new Date().toISOString() }` with millisecond precision.

### 11.5 Scoring, Penalties & Redis Leaderboards
- **Ingestion Pipeline:**
  1. Student submits code to `POST /assessments/:id/submissions` with `{ metadata: { eventId } }`.
  2. BullMQ executes code in Judge0 CE sandbox runner.
  3. `CodeExecutionConsumer` emits `SubmissionGraded`.
  4. `SubmissionGradedConsumer` receives event, validates `verdict in ['accepted', 'partial']`, and confirms `event.status === 'live' && event.kind === 'competition'`.
- **Redis Leaderboard Updates:**
  - Key: `leaderboard:event:${eventId}`
  - Command: `await this.redis.zadd(leaderKey, numScore, userId);`
  - Upserts `events.event_results` with `{ score: numScore, metadata: { lastSubmissionId } }`.
- **CRITICAL FORENSIC DEFECT — Inverted Leaderboard Ranks:**
  In `SubmissionGradedConsumer` (Line 42):
  ```typescript
  const rank = await this.redis.zrank(leaderKey, userId as string);
  ```
  `ZRANK` ranks members in **ascending order** (lowest score = index 0). In a competition where higher score is better, `ZREVRANK` must be used. Using `zrank` causes the participant with the lowest score (e.g. 0 points) to receive rank 1, while the top performer receives the worst rank!
- **Absence of ICPC Penalty Engine:**
  Frontend types define `ScoringType = 'icpc' | 'weighted' | 'simple'` and `wrongAttemptPenaltyMinutes`. In the backend source code, **0 penalty calculations, wrong attempt tracking, or cumulative submission time trackers exist**. Scoring is purely based on raw Judge0 test-case percentage.

### 11.6 Post-Contest Analytics & Results Archival
- **Automated Final Grading:** `EventFinalGradingService` runs on cron `*/5 * * * *`.
  - Scans events where `status === 'grading'` and `endsAt < now - 30 minutes`.
  - Fetches all `events.event_results` ordered by `score: 'desc'`.
  - Sequentially assigns 1-based ranks: `rank: i + 1`.
  - Determines certificate eligibility:
    ```typescript
    certificateEligible: i < 3
    ```
    **Hardcoded Rule:** Strictly the top 3 participants ($i \in \{0, 1, 2\}$) are marked eligible for certificates.
  - Updates `event.status = 'completed'` and emits `EventCompleted`.
- **Pairwise ELO Platform Rating Engine:**
  `PlatformRatingService` (`src/events/services/platform-rating.service.ts`) executes when `EventCompleted` is emitted:
  - Requires $N \ge 2$ ranked participants.
  - Constants: $K = 32$, $R_{\text{initial}} = 1200$, Floor = 100.
  - Expected Score Formula:
    $$E(A, B) = \frac{1}{1 + 10^{(R_B - R_A)/400}}$$
  - Pairwise Delta Accumulation:
    $$\Delta R_i = \sum_{j \ne i} K \cdot (S_{ij} - E(R_i, R_j))$$
    Where $S_{ij} = 1$ if $i < j$ (ranked higher), and $S_{ij} = 0$ if $i > j$.
  - Normalized Rating:
    $$R_{\text{new}} = \max\left(100, \text{round}\left(R_i + \frac{\Delta R_i}{N - 1}\right)\right)$$
  - Persists new rating to `identity.persons.platformRating` and updates Redis sorted set `leaderboard:platform:global`.
- **Broken Global Leaderboard Database Query:**
  In `EventsRepository.findGlobalLeaderboard`:
  ```typescript
  return (this.prisma.events as any).platformRating?.findMany(...) ?? [];
  ```
  Model `platformRating` does not exist in `prisma.events`. The method catches the error and returns `[]`.

---

## SECTION G: Authentication, Authorization & RBAC

### G.1 Role-Based Permissions Matrix

| Permission String | Description | Permitted Roles in `permission.guard.ts` |
|---|---|---|
| `read:cohort` | Basic authenticated reading of events & leaderboards | `student`, `faculty`, `coordinator`, `college_admin`, `content_manager`, `platform_moderator`, `platform_admin` |
| `create:event` | Authoring events and publishing drafts | `coordinator`, `college_admin`, `platform_admin` |
| `manage:event` | Updating, extending, cancelling, and viewing rosters | `coordinator`, `college_admin`, `platform_moderator`, `platform_admin` |
| `register:event` | Event registration permission | `student` |

---

## SECTION H: Real-Time & Event-Driven Mechanics

### H.1 Redis Channel & Key Conventions

| Purpose | Channel / Key Pattern | Type | Payload / Structure |
|---|---|---|---|
| **Live Event Leaderboard** | `leaderboard:event:${eventId}` | Redis ZSET | Member: `userId`, Score: `numScore` (Float) |
| **Global Rating Leaderboard** | `leaderboard:platform:global` | Redis ZSET | Member: `userId`, Score: `platformRating` (Int) |
| **Leaderboard Live PubSub** | `dezolver#event:${eventId}:leaderboard` | PubSub | `['leaderboard:update', { userId, score, rank }]` |
| **Event State WebSocket** | Room `event:${eventId}` | Socket.io | Event: `event:state`, Payload: `{ eventId, status }` |

### H.2 Defect in Socket Room Authorization (`channel-authorizers.ts`)
```typescript
const isOrganizer = await this.prisma.events.event.findFirst({
  where: { id, organizerTenantId: principal.tenantId },
}).catch(() => null);
```
**Defect:** Table `events.events` has column `tenantId`, NOT `organizerTenantId`. Prisma throws a schema validation error which is caught, returning `null`. Organizers cannot join the event WebSocket room via tenant matching.

---

## SECTION I: Frontend Architecture & User Interface

### I.1 Frontend Page Catalog (`frontend/src/pages/events/`)

1. **`EventsPage.tsx`**: Main catalog displaying tabs (`All`, `Competitions`, `Workshops`), search, filters, and event cards.
2. **`EventDetailPage.tsx`**: Detailed overview showing description, timeline countdown, capacity progress bar, fee details, and registration button.
3. **`EventCreatePage.tsx`**: Multi-field creation form validating kind, title, description, times, audience scope, capacity, and pricing.
4. **`EventEditPage.tsx`**: Edit form restricted to draft events.
5. **`EventLivePage.tsx`**: Live competition workspace containing problem statement navigator, countdown clock, code editor, and submit button.
6. **`EventLeaderboardPage.tsx`**: Real-time leaderboard table displaying rank, participant name, score, and last submission time.
7. **`EventResultsPage.tsx`**: Concluded event standings displaying final ranks, winners podium, and certificate download badges.
8. **`EventManagePage.tsx`**: Organizer dashboard showing registration roster, cancellation modal, and extension modal.
9. **`EventAnalyticsPage.tsx`**: Score distribution charts and submission volume telemetry.

---

## SECTION J: Cross-Module Interactions & External Integrations

```
  ┌────────────────────────────────────────────────────────────────────────┐
  │                           MODULE 11 CONTESTS                           │
  └───────┬───────────────────────────┬───────────────────────────┬────────┘
          │                           │                           │
          ▼                           ▼                           ▼
┌──────────────────┐        ┌──────────────────┐        ┌──────────────────┐
│ MODULE 6/7       │        │ MODULE 3         │        │ MODULE 12        │
│ CODE EXECUTION   │        │ BILLING          │        │ CERTIFICATES     │
│ Submits via      │        │ Paid orders via  │        │ Top 3 eligible   │
│ assessment proxy │        │ Razorpay API;    │        │ for automated    │
│ to Judge0        │        │ auto-refunds on  │        │ certificate      │
│ runner.          │        │ cancellation.    │        │ issuance.        │
└──────────────────┘        └──────────────────┘        └──────────────────┘
```

---

## SECTION K: Edge Cases, Failure Modes & Resilience

1. **Simultaneous Final Grading Execution:** Both `EventStateWatcherService` (at $+60$m) and `EventFinalGradingService` (at $+30$m) attempt to advance `grading` -> `completed`. `EventFinalGradingService` runs first and emits `EventCompleted` with ranks assigned; `EventStateWatcherService` skips if already `completed`.
2. **Tied Scores in Final Grading:** `EventFinalGradingService` sorts strictly by `score: 'desc'`. When two participants have identical scores, ranks are assigned arbitrarily by database sort order (`1, 2, 3...`) without applying secondary tie-breaker criteria (such as earliest submission time).
3. **Zero Registrants at Event Start:** If an event reaches `startsAt` with 0 registrations, it advances to `live` and `grading` normally, concluding with `finalRanksAssigned: 0`.

---

## SECTION L: Performance, Caching & Scaling

- **Redis Leaderboard Ingestion:** By using Redis sorted sets (`ZADD`), leaderboard score updates execute in $O(\log N)$ time, allowing Dezolver to handle thousands of concurrent competition submissions.
- **Pairwise ELO Complexity:** In `PlatformRatingService`, pairwise comparison requires $O(N^2)$ iterations. For large contests ($N > 1,000$), this loop executes over 1,000,000 iterations inside an asynchronous BullMQ worker.

---

## SECTION M: Security, Threat Model & Compliance

- **Submission Deadline Integrity:** While assessments lack deadline enforcement, competitions enforce state checks: `submitEventSolution` verifies `event.status === 'live'` and rejects submissions with `400 BadRequestException({ code: 'event_not_live' })`.
- **Payment Tampering Protection:** `EventOrdersController` generates cryptographically signed orders via Razorpay SDK; payment capture verification is validated via server-side webhook signatures before registration activation.

---

## SECTION N: Observability, Metrics & Audit Logging

Audited operations recorded to `platform.audit_logs`:
- `event.create`: Event creation.
- `event.publish`: Moving draft to published.
- `event.update`: Draft parameter modifications.
- `event.cancel`: Cancellation reason and actor.
- `event.extend`: New end time and justification.
- `event.register`: User registration.
- `event.unregister`: User withdrawal.
- `event.record_result`: Staff score override.

---

## SECTION O: Negative Specification (What is NOT Implemented)

```
┌────────────────────────────────────────────────────────────────────────┐
│                   DEZOLVER MODULE 11 NEGATIVE MATRIX                   │
├──────────────────────────────────┬─────────────────────────────────────┤
│ Feature / Capability             │ Legacy Implementation Reality       │
├──────────────────────────────────┼─────────────────────────────────────┤
│ 1. Database EventProblem Table   │ 0 tables, 0 models across schemas   │
│ 2. Event Config JSONB Column     │ Missing from prisma/events schema   │
│ 3. ICPC Penalty Calculations     │ 0 code occurrences in backend       │
│ 4. Direct Event Code Execution   │ POST /submissions returns a stub    │
│ 5. Correct ZREVRANK Leaderboard  │ Defective: uses ascending ZRANK     │
│ 6. Global Leaderboard DB Model   │ Broken: queries nonexistent model   │
│ 7. Configurable Certificate Gate │ Hardcoded strictly to rank < 3      │
│ 8. Audience / Cohort Restriction │ Not evaluated during registration   │
│ 9. Secondary Score Tie-Breakers  │ Arbitrary database sort order       │
└──────────────────────────────────┴─────────────────────────────────────┘
```

---

## SECTION P: Forensic Code Verification & Traceability Matrix

| Component | Source File Path | Line Reference | Implementation Details |
|---|---|---|---|
| **Event Database Schema** | `prisma/events/schema.prisma` | Lines 15–38 | `model Event` definition. |
| **Registration Schema** | `prisma/events/schema.prisma` | Lines 40–56 | `model Registration` with `@@unique([eventId, userId])`. |
| **Event Results Schema** | `prisma/events/schema.prisma` | Lines 58–75 | `model EventResult` with `@@unique([eventId, userId])`. |
| **Lifecycle State Watcher** | `src/events/services/event-state-watcher.service.ts` | Lines 5–59 | Automated 30s cron transitions. |
| **Final Grading Service** | `src/events/services/event-final-grading.service.ts` | Lines 15–53 | Final ranking and `certificateEligible: i < 3`. |
| **Pairwise ELO Engine** | `src/events/services/platform-rating.service.ts` | Lines 6–80 | Formula implementation ($K=32$, floor 100). |
| **Leaderboard Consumer** | `src/events/consumers/submission-graded.consumer.ts` | Lines 13–50 | Upserts result, calls `zadd`, pub/sub broadcast. |
| **Inverse Rank Defect** | `src/events/consumers/submission-graded.consumer.ts` | Line 42 | Uses `redis.zrank` (ascending) instead of `zrevrank`. |
| **Submission Stub** | `src/events/services/events.service.ts` | Lines 215–220 | Returns stub pointing to assessments API. |
| **Registration Guard Defect** | `src/events/services/events.service.ts` | Line 93 | Checks `status !== 'published'` (fails on `registration_open`). |
| **Organizer WS Room Defect** | `src/websocket/auth/channel-authorizers.ts` | Line 34 | Queries nonexistent `organizerTenantId`. |
| **Global Leaderboard Defect** | `src/events/repositories/events.repository.ts` | Line 105 | Queries nonexistent `platformRating` model on events client. |
| **Razorpay Payment Orders** | `src/billing/controllers/event-orders.controller.ts` | Lines 26–42 | `POST /event-registration-orders`. |
| **Automated Refund Consumer** | `src/billing/consumers/event-cancelled.consumer.ts` | Lines 15–55 | Automatic Razorpay refunds on `EventCancelled`. |

---

## SECTION Q: Target State Recommendations & Remediations

### Q.1 Immediate Critical Remediations
1. **Fix Inverted Leaderboard Ranking:** In `SubmissionGradedConsumer.ts` (Line 42), replace `this.redis.zrank` with `this.redis.zrevrank` so higher scores receive rank 1.
2. **Fix Registration State Guard:** In `EventsService.register` (Line 93), change condition from `status !== 'published'` to `!['published', 'registration_open'].includes(event.status)`.
3. **Fix Organizer WebSocket Authorization:** In `channel-authorizers.ts` (Line 34), change `organizerTenantId` to `tenantId`.
4. **Fix Global Leaderboard Query:** In `EventsRepository.findGlobalLeaderboard`, query `this.prisma.identity.person.findMany({ orderBy: { platformRating: 'desc' } })` instead of the nonexistent `platformRating` on `prisma.events`.

### Q.2 Architectural Enhancements
1. **Add `EventProblem` Relational Table:** Create table `events.event_problems` with `(event_id, problem_id, points, order, penalty_minutes)` to natively persist multi-problem contest configurations.
2. **Implement Direct Contest Submission Engine:** Enhance `POST /events/:id/submissions` to ingest solution code directly, execute via BullMQ, and attribute scores natively to the contest problem set.
3. **Implement Full ICPC Scoring Engine:** Track wrong submission count per problem, calculate total solve time plus 20-minute penalties, and order leaderboards by problems solved descending, penalty time ascending.
4. **Configurable Certificate Thresholds:** Allow organizers to configure certificate rules (e.g., Top 10%, score $\ge 80\%$, or Top $K$ participants) rather than the hardcoded top 3 rule.

