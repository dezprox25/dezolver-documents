# Module 11 — Contests (11.1–11.6)

## 1. Module Executive Summary

Module 11 governs competitive programming contests, hackathons, and technical events on the Dezolver platform, encompassing contest scheduling, problem associations, participant registrations, live submission grading via Judge0, real-time Redis leaderboards, ELO platform rating calculations, and certificate eligibility.

A forensic investigation of the legacy codebase (`src/events/`, `prisma/events/`, `src/billing/`, and `frontend/src/pages/events/`) uncovers significant architectural implementations alongside critical legacy defects:

1. **Fully Implemented Event Lifecycle & Automated State Machine:** The platform features a robust, automated state machine managed by `EventStateWatcherService` (30-second cron) transitioning events through `draft` -> `published` -> `registration_open` (at $T-7$ days) -> `registration_closed` (at $T$) -> `live` (at $T$) -> `grading` (at $T_{\text{end}}$) -> `completed` (at $T_{\text{end}}+1\text{h}$).
2. **Missing Problem Association Schema:** In `prisma/events/schema.prisma`, there is **no `EventProblem` model or `config` JSONB column**. While the frontend defines `CompetitionProblem` and `EventConfig`, the backend database possesses no table or column to persist problem IDs for an event.
3. **Indirect Contest Submission Architecture:** The endpoint `POST /events/:id/submissions` is an intentional stub returning a guidance message. Contestants actually submit code to `POST /assessments/:id/submissions` specifying `{ metadata: { eventId } }`.
4. **Live Redis Leaderboard with Inverse Rank Defect:** `SubmissionGradedConsumer` updates Redis sorted set `leaderboard:event:${eventId}` on accepted submissions and broadcasts real-time rank updates via WebSocket. However, it queries rank using `redis.zrank` (ascending order) instead of `redis.zrevrank`, resulting in **inverted rankings where higher scores receive worse ranks**!
5. **Rigorous ELO Rating Engine:** Dezolver implements a pairwise ELO rating algorithm (`K_FACTOR = 32`, `INITIAL_RATING = 1200`, floor 100) inside `PlatformRatingService`, automatically recalculating each participant's `platformRating` on `identity.persons` upon event completion.
6. **Paid Events & Automated Razorpay Refunds:** Full payment integration via `POST /event-registration-orders` (Razorpay orders) and `EventCancelledConsumer`, which automatically issues Razorpay refunds to all registrants if an event is cancelled.
7. **Top-3 Certificate Eligibility:** `EventFinalGradingService` assigns final ranks by descending score and sets `certificateEligible: true` strictly for the top 3 participants ($i < 3$).
8. **Broken Global Leaderboard Query:** `EventsRepository.findGlobalLeaderboard` erroneously queries `(this.prisma.events as any).platformRating` (which does not exist in the events schema), returning empty arrays.

---

## 2. Directory Layout & Sub-Module Coverage

```
docs/functional-spec/module-11-contests/
├── README.md               # Executive summary, findings, and navigational guide (this file)
└── SPECIFICATION.md        # Comprehensive 17-Section (A–Q) legacy functional reconstruction
```

Module 11 exhaustively covers all 6 required contest sub-domains:
- **11.1 Contest Creation & Scheduling:** Kind selection, draft lifecycle, manual publishing, automated state transitions, update, extension, and cancellation.
- **11.2 Contest Problems & Test Cases:** Forensic analysis of problem linking, schema gaps, and submission routing.
- **11.3 Registration & Eligibility:** Free vs. paid registration, Razorpay order integration, capacity constraints, and withdrawal.
- **11.4 Contest Window & Real-time Submissions:** Live window activation, Judge0 code execution, and WebSocket state broadcasts.
- **11.5 Scoring, Penalties & Redis Leaderboards:** Real-time scoring, Redis ZSET mechanics, ZRANK inversion defect, and missing ICPC penalty tracking.
- **11.6 Post-Contest Analytics & Results Archival:** Automated final grading, top-3 certificate eligibility, pairwise ELO rating computation, and global leaderboards.

---

## 3. Subsystem Architecture & Flow

```
   [Organizer]
        │ POST /events (kind: 'competition', startsAt, endsAt)
        ▼
   ┌─────────────┐  POST /events/:id/publish   ┌─────────────────┐
   │ Event:Draft ├────────────────────────────►│ Event:Published │
   └─────────────┘                             └────────┬────────┘
                                                        │
                      ┌─────────────────────────────────┴─────────────────────────────────┐
                      │ Cron: EventStateWatcherService (every 30s)                        │
                      ▼                                                                   ▼
       ┌──────────────────────────────┐                                   ┌──────────────────────────────┐
       │   Event:registration_open    │                                   │          Event:live          │
       │     (at startsAt - 7d)       │                                   │        (at startsAt)         │
       └──────────────┬───────────────┘                                   └──────────────┬───────────────┘
                      │ POST /registrations                                              │
                      ▼                                                                  │ POST /assessments/:id/submissions
       ┌──────────────────────────────┐                                                  │ { metadata: { eventId } }
       │   Registration:registered    │                                                  ▼
       └──────────────────────────────┘                                   ┌──────────────────────────────┐
                                                                          │  CodeExecutionConsumer       │
                                                                          │     (Judge0 CE Runner)       │
                                                                          └──────────────┬───────────────┘
                                                                                         │ emit SubmissionGraded
                                                                                         ▼
                                                                          ┌──────────────────────────────┐
                                                                          │   SubmissionGradedConsumer   │
                                                                          │  - Upsert EventResult        │
                                                                          │  - ZADD leaderboard:event    │
                                                                          │  - WS Broadcast live rank    │
                                                                          └──────────────┬───────────────┘
                                                                                         │ Cron: endsAt
                                                                                         ▼
                                                                          ┌──────────────────────────────┐
                                                                          │        Event:grading         │
                                                                          └──────────────┬───────────────┘
                                                                                         │ Cron: EventFinalGradingService
                                                                                         │ (endsAt + 30 min)
                                                                                         ▼
                                                                          ┌──────────────────────────────┐
                                                                          │       Event:completed        │
                                                                          │  - Assign final ranks (1..N) │
                                                                          │  - Top 3 certificateEligible │
                                                                          │  - Pairwise ELO calculation  │
                                                                          └──────────────────────────────┘
```

---

## 4. Key Forensic Discoveries Table

| Subsystem / Feature | Conceptual Requirement | Legacy Reality in Source Code | Forensic Status |
|---|---|---|:---:|
| **Automated State Watcher** | Cron-based status changes | Transitions events every 30s via `EventStateWatcherService`. | `CONFIRMED LEGACY IMPLEMENTATION` |
| **Problem Storage** | Join table linking problems | 0 tables in schema; `config` JSONB missing from `events.events`. | `NOT FOUND IN SCHEMA` |
| **Submission Ingress** | Direct event submission API | `POST /events/:id/submissions` returns stub; uses assessments API. | `PARTIALLY IMPLEMENTED` |
| **Live Leaderboard** | Real-time score ranking | Redis ZSET `leaderboard:event:${id}` updated via `SubmissionGraded`. | `CONFIRMED LEGACY IMPLEMENTATION` |
| **Leaderboard Rank Logic** | High score gets 1st place | Uses `redis.zrank` (ascending) instead of `zrevrank` (inverted ranks!). | `DEFECT / ALGORITHMIC ERROR` |
| **ICPC Penalty System** | 20m wrong penalty, time rank | 0 penalty tracking code exists in backend. | `NOT IMPLEMENTED` |
| **Pairwise ELO Rating** | Post-contest rating updates | Full pairwise ELO algorithm ($K=32$) in `PlatformRatingService`. | `CONFIRMED LEGACY IMPLEMENTATION` |
| **Certificate Eligibility** | Configurable threshold / top-N | Hardcoded `rank < 3` (strictly top 3 participants). | `CONFIRMED LEGACY IMPLEMENTATION` |
| **Paid Events & Refunds** | Razorpay gateway integration | `EventOrdersController` creates orders; auto-refunds on cancel. | `CONFIRMED LEGACY IMPLEMENTATION` |
| **Global Leaderboard** | Top rated platform users | Broken query on nonexistent `platformRating` model in events DB. | `DEFECT / SCHEMA MISMATCH` |

---

## 5. Source File Traceability Index

| Subsystem | File Path | Primary Functionality |
|---|---|---|
| **Database Schema** | `prisma/events/schema.prisma` | Defines `Event`, `Registration`, and `EventResult` |
| **Controller** | `src/events/controllers/events.controller.ts` | Endpoints for CRUD, publish, register, cancel, extend, standing |
| **Leaderboard Controller** | `src/events/controllers/leaderboard.controller.ts` | Global leaderboard and person standing endpoints |
| **Service** | `src/events/services/events.service.ts` | Business logic for registration, status changes, and roster |
| **State Watcher Cron** | `src/events/services/event-state-watcher.service.ts` | Automated status transitions every 30s |
| **Final Grading Cron** | `src/events/services/event-final-grading.service.ts` | Final rank assignment and top-3 certificate eligibility |
| **ELO Rating Service** | `src/events/services/platform-rating.service.ts` | Pairwise ELO calculation and `platformRating` update |
| **Reminder Cron** | `src/events/services/registration-reminder.service.ts`| 24h and 1h email reminders via BullMQ |
| **Grading Consumer** | `src/events/consumers/submission-graded.consumer.ts` | Redis leaderboard updates and WebSocket broadcasts |
| **Registration Consumer** | `src/events/consumers/event-registration.consumer.ts` | Confirms paid registrations upon Razorpay webhook |
| **Billing Orders** | `src/billing/controllers/event-orders.controller.ts` | Razorpay order generation for paid registrations |
| **Billing Refund** | `src/billing/consumers/event-cancelled.consumer.ts` | Razorpay refund processing on `EventCancelled` |
| **WebSocket Broadcaster**| `src/websocket/consumers/realtime-events.consumer.ts` | Broadcasts `EventLive`, `EventCompleted`, `EventCancelled` |

