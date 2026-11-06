# Module 17 — Functional State Machines

## 1. Module Executive Summary

Module 17 synthesizes, formalizes, and audits all operational state machines, entity lifecycles, and automated transition mechanisms across the entire Dezolver platform.

A comprehensive forensic audit of all 11 database schemas, backend NestJS services, and React frontend components reveals **12 distinct functional state machines** governing institutional tenancies, competitions, submissions, learning paths, user accounts, subscriptions, content approvals, certificates, and platform launch phases:

1. **Tenant Lifecycle State Machine (`tenancy.tenants`):** Governed by `TenantLifecycleService` (`src/tenancy/services/tenant-lifecycle.service.ts`) through 7 explicit states (`pending`, `trial`, `active`, `suspended`, `expired`, `cancelled`, `purged`). Transitions are strictly enforced via an in-memory dictionary `ALLOWED_TRANSITIONS`.
2. **Contest / Event Automated State Machine (`events.events`):** Governed by `EventStateWatcherService` (`src/events/services/event-state-watcher.service.ts`) through 8 states (`draft`, `published`, `registration_open`, `registration_closed`, `live`, `grading`, `completed`, `cancelled`). An automated 30-second cron transitions events based on wall-clock thresholds ($T-7\text{d}$, $T$, $T_{\text{end}}$, $T_{\text{end}}+1\text{h}$).
3. **Event Participant Registration State Machine (`events.registrations`):** Covers 5 states (`registered`, `waitlisted`, `pending_payment`, `cancelled`, `refunded`), transitioning on payment capture webhooks, manual unregistration, or automated contest cancellation refunds.
4. **Code Submission & Evaluation State Machine (`assessment.submissions`):** Covers 9 execution verdicts (`pending`, `accepted`, `partial`, `wrong_answer`, `time_limit_exceeded`, `memory_limit_exceeded`, `runtime_error`, `compilation_error`, `system_error`) and a binary proctoring review flag (`isFlagged`). Evaluated asynchronously by `Judge0Consumer`.
5. **Learning Path & Room Progress State Machine (`paths.path_progress` & `paths.room_progress`):** Tracks room progress through 4 states (`not_started`, `in_progress`, `skipped`, `completed`) via raw SQL, driving asynchronous path progress aggregation ($0\% \to 100\%$) and certificate triggers via `ProgressAggregatorConsumer`.
6. **User Lifecycle & Invitation State Machine (`identity.users` & `tenancy.invitations`):** Invitations transition through 4 states (`pending`, `accepted`, `expired`, `revoked` with a 7-day TTL), while user accounts transition through 4 states (`pending`, `active`, `suspended`, `deleted`).
7. **Subscription, Billing & Invoice State Machine (`billing.subscriptions` & `billing.invoices`):** Subscriptions transition through 5 states (`trial`, `active`, `past_due`, `cancelled`, `unpaid`) driven by Razorpay webhooks and the `DunningService` automated retry cron.
8. **Room Authoring & Content Approval State Machine (`content.rooms`):** Authorship pipeline spanning 5 states (`draft`, `in_review`, `approved`, `published`, `archived`) with explicit review, approve, rollback, and archive endpoints.
9. **Assessment & Problem Lifecycle State Machine (`assessment.assessments` & `content.problems`):** Academic assessments transition through 4 states (`draft`, `published`, `active`, `archived`), while problem definitions transition through 3 states (`draft`, `published`, `archived`).
10. **Certificate Credential State Machine (`credentials.certificates`):** Dual-state lifecycle (`issued`, `revoked`) with public hex verification, revocation, and reissuance capabilities.
11. **Platform Launch Phase State Machine (`src/launch/launch.service.ts`):** Irreversible system-wide progression across 3 phases (`A` -> `B` -> `C`) gating direct signups, SSO, course catalogs, and the Judge0 execution engine, protected by date-coded confirmation tokens (`YYYYMMDD`).
12. **Transactional Outbox Event Dispatch State Machine (`platform.outbox_events`):** Reliable messaging pipeline transitioning from `pending` -> `dispatched` or `failed` with exponential retry backoff, routing failed events to `platform.outbox_failed`.

---

## 2. Directory Layout & Document Navigation

```
docs/functional-spec/module-17-state-machines/
├── README.md               # Executive summary, state machine registry, and navigation guide (this file)
└── SPECIFICATION.md        # Comprehensive 17-Section (A–Q) legacy specification & traceability matrix
```

- **Detailed Functional Specification:** [`SPECIFICATION.md`](./SPECIFICATION.md)
- **Authoring Standard:** [`FUNCTIONAL-SPECIFICATION-STANDARD.md`](../../FUNCTIONAL-SPECIFICATION-STANDARD.md) (`DEZ-STD-FS-001`)
- **Master Index & Registry:** [`INDEX.md`](../INDEX.md)

---

## 3. Master State Machine Registry

| # | Domain / Entity | States Defined | Enforcement Mechanism | Key Automated / Manual Triggers |
|:---:|---|---|---|---|
| **01** | **Tenant Lifecycle** (`Tenant.status`) | `pending`, `trial`, `active`, `suspended`, `expired`, `cancelled`, `purged` | `ALLOWED_TRANSITIONS` map in `TenantLifecycleService` | `POST /tenants/:id/transition`, Dunning expiry |
| **02** | **Contest / Event** (`Event.status`) | `draft`, `published`, `registration_open`, `registration_closed`, `live`, `grading`, `completed`, `cancelled` | `EventStateWatcherService` (30s cron) | Wall-clock time thresholds ($T-7\text{d}$, $T$, $T_{\text{end}}$, $T_{\text{end}}+1\text{h}$) |
| **03** | **Event Registration** (`Registration.status`) | `registered`, `waitlisted`, `pending_payment`, `cancelled`, `refunded` | `EventsService`, `EventOrdersController` | Razorpay payment capture webhook, `EventCancelled` |
| **04** | **Code Submission** (`Submission.verdict`) | `pending`, `accepted`, `partial`, `wrong_answer`, `time_limit_exceeded`, `memory_limit_exceeded`, `runtime_error`, `compilation_error`, `system_error` | `Judge0Consumer` & Test Case Scoring Engine | BullMQ queue `submission-eval`, `POST /submissions/:id/rerun` |
| **05** | **Path & Room Progress** (`room_progress` / `path_progress`) | `not_started`, `in_progress`, `skipped`, `completed` ($0\% \to 100\%$) | `MePathsController`, `ProgressAggregatorConsumer` | `POST /me/rooms/:id/*`, `RoomProgressUpdated` outbox event |
| **06** | **User & Invitation** (`User.status` / `Invitation.status`) | User: `pending`, `active`, `suspended`, `deleted`<br/>Invite: `pending`, `accepted`, `expired`, `revoked` | `TenantsService`, `AuthService`, `UsersService` | `POST /auth/invitations/accept`, 7-day TTL, GDPR deletion |
| **07** | **Subscription & Billing** (`Subscription.status`) | `trial`, `active`, `past_due`, `cancelled`, `unpaid` | `SubscriptionsService`, `DunningService` (daily cron) | Razorpay invoice webhooks, payment retries, manual cancellation |
| **08** | **Room Content Approval** (`Room.status`) | `draft`, `in_review`, `approved`, `published`, `archived` | `RoomsController`, `RoomsService` | Review submit, admin approve, rollback, catalog publish |
| **09** | **Assessments & Problems** (`Assessment.status` / `Problem.status`) | Assessment: `draft`, `published`, `active`, `archived`<br/>Problem: `draft`, `published`, `archived` | `AssessmentsController`, `ProblemsController` | Manual publish, start/end scheduling |
| **10** | **Certificates** (`Certificate.status`) | `issued`, `revoked` | `CredentialsService`, `CredentialConsumers` | Milestone completion (`PathCompleted`, `EventCompleted`), revoke/reissue |
| **11** | **Platform Launch Phase** (`launch_phase_*`) | Phase `A` -> Phase `B` -> Phase `C` | `LaunchService` with confirmation code (`YYYYMMDD`) | `POST /admin/v1/launch/advance-phase` |
| **12** | **Outbox Dispatch** (`OutboxEvent`) | `pending`, `dispatched`, `failed`, `dead_letter` | Outbox Poller Cron (`every 5s`), BullMQ Dispatcher | DB commit, dispatch success, retry limit ($N \ge 5$) |

---

## 4. Critical Defects & State Machine Hazards

| Defect ID | Severity | Entity / Location | Forensic Description & Blast Radius |
|---|---|---|---|
| **DEF-17-01** | `CRITICAL` | `events.events` (`src/events/services/events.service.ts:86`) | **Event Registration State Lockout**: `register` requires `status === 'published'`. However, `EventStateWatcherService` moves events from `published` to `registration_open` at $T-7\text{d}$. Once open, registrations fail with `400 event_not_open`, locking out all participants. |
| **DEF-17-02** | `HIGH` | `paths.room_progress` (`src/paths/consumers/progress-aggregator.consumer.ts:42`) | **Skip-to-Completion State Exploit**: `ProgressAggregatorConsumer` treats `status === 'skipped'` as completed steps when calculating path percentage. A student can skip every room via `POST /me/rooms/:id/skip` and achieve 100% completion, illegally triggering certificate issuance. |
| **DEF-17-03** | `HIGH` | `tenancy.tenants` (`src/tenancy/services/tenant-lifecycle.service.ts:20`) | **Irreversible Cancellation**: Allowed transitions define `cancelled: ['purged']`. If a tenant subscription is accidentally cancelled, there is no transition back to `active` or `trial`. Institutional data is permanently stuck until purged. |
| **DEF-17-04** | `HIGH` | `billing.subscriptions` (`src/billing/services/subscriptions.service.ts:94`) | **Missing Webhook State Synchronization**: Razorpay subscription cancellations are not reconciled against `billing.subscriptions` via incoming webhook handler; only manual API cancellations update `status = 'cancelled'`. |
| **DEF-17-05** | `MEDIUM` | `src/launch/launch.service.ts:24-36` | **Irreversible Launch Phase Progression**: `advancePhase` sets feature flags without providing any rollback mechanism. Advancing from Phase B to C permanently alters platform flag defaults. |

---

## 5. Verification & Traceability Sign-off

- **Negative Verification:** Audited all entity status columns across all 11 schemas and confirmed transition logic in backend services.
- **Traceability Matrix:** Fully documented in [`SPECIFICATION.md`](./SPECIFICATION.md#section-q-master-verification-traceability-matrix).
- **Adherence Standard:** Strictly complies with `DEZ-STD-FS-001`.

