# DEZOLVER — Module 17: Functional State Machines

**Document ID:** STATE-MACHINES-FS-001  
**Version:** 0.1  
**Status:** Detailed draft — legacy lifecycle reconstruction; transition policies and cross-module rules require approval  
**Governing standard:** [Functional Specification Standard v1.0](../FUNCTIONAL-SPECIFICATION-STANDARD.md)  
**Source reconstruction:** [Earlier Module 17 forensic specification](../../functional-spec-from-old%20code/module-17-state-machines/SPECIFICATION.md)

> **Evidence caveat:** State sets and implementation behavior below are taken from the previous forensic reconstruction and supplied schema/RBAC/business-flow documents. They are legacy evidence, not automatically approved target rules. Revalidate source paths and current deployed schema before implementation sign-off.

## 1. Purpose and mission

Module 17 defines the lifecycle/state-transition contract for entities that progress through discrete states across DEZOLVER. It is the cross-module reference for valid transitions, transition authorization, side effects, state-derived UI behavior, asynchronous transition triggers, and terminal/reversal behavior.

Prior forensic reconstruction reports **12 state machines**:
1. Tenant lifecycle
2. Event/contest lifecycle
3. Event participant registration
4. Code submission/evaluation verdict
5. Learning path and room progress
6. User and invitation lifecycle
7. Subscription, invoice, and payment lifecycle
8. Room content approval
9. Assessment and problem lifecycle
10. Certificate lifecycle
11. Platform launch phase
12. Transactional outbox dispatch

## 2. Scope

### In scope
- Canonical state/transition inventory for the 12 reported machines.
- Transition actor/permission/precondition/side-effect contract.
- Synchronous, scheduled, and asynchronous transitions.
- Transition guards, invalidation, cascade, and concurrency considerations.
- Cross-module lifecycle coordination and event-driven side effects.
- Defect identification and target decision register.

### Out of scope
- Full function specs for individual module business features (Modules 1–16).
- Billing/payment provider behavior beyond subscription/payment/invoice state transitions (Module 3).
- Contest scoring/rank algorithm (Module 11).
- Code execution/evaluation details (Modules 6–7).
- Certificate design/verification mechanics (Module 12).
- General retries/error handling not specific to a state transition (Module 18).

## 3. Evidence sources

| Source | Evidence |
|---|---|
| [Prior Module 17 forensic specification](../../functional-spec-from-old%20code/module-17-state-machines/SPECIFICATION.md) | Reported 12 state machines, transitions, defects, services, workers, and traces |
| [Schema reconstruction](../../docs/source-text/scheema.md) | Conceptual entity/status fields and enum names |
| [Business Flow v2](../../docs/source-text/Business%20Flow%20v2.md) | High-level user and coding/contest lifecycle flow |
| [RBAC](../../docs/source-text/RBAC.md) | Roles, scope, ownership, server-side authorization |
| [Functional Specification Standard](../FUNCTIONAL-SPECIFICATION-STANDARD.md) | Evidence and function-level authoring requirements |
| Modules 1–16 functional specs | Existing module-level business rules and decisions; coordinate rather than override |

### Evidence labels
- **Source-confirmed:** explicit in supplied source documentation.
- **Legacy-verified (prior forensic record):** previous forensic report’s observed implementation behavior; current source not rechecked in this draft.
- **Derived:** conclusion from multiple sources.
- **Recommendation:** target suggestion, not approved.
- **TBD / Decision Required:** missing or conflicting policy.

## 4. Lifecycle/state machine inventory

| # | Entity / machine | Legacy-reported states | Transition control | Evidence/status |
|---:|---|---|---|---|
| 1 | Tenant | pending, trial, active, suspended, expired, cancelled, purged | Synchronous lifecycle service; strict allowlist reported | Prior forensic record |
| 2 | Event/contest | draft, published, registration_open, registration_closed, live, grading, completed, cancelled | Admin action + scheduled watcher | Prior forensic record |
| 3 | Registration | registered, waitlisted, pending_payment, cancelled, refunded | Registration/payment/cancellation flows | Prior forensic record |
| 4 | Submission verdict | pending, accepted, partial, wrong_answer, time_limit_exceeded, memory_limit_exceeded, runtime_error, compilation_error, system_error | Judge0/evaluation worker | Prior forensic record; align Modules 6/7 |
| 5 | Path/room progress | not_started, in_progress, skipped, completed; path aggregate | Student action + progress consumer | Prior forensic record; align Module 16 |
| 6 | User/invitation | user: pending, active, suspended, deleted; invitation: pending, accepted, expired, revoked | Admin/auth/invitation workflows | Prior forensic record |
| 7 | Subscription/invoice/payment | subscription: trial, active, past_due, cancelled, unpaid; invoice: draft, open, paid, void, uncollectible; payment: created and provider statuses | Billing actions, provider webhooks, dunning cron | Prior forensic record; align Module 3 |
| 8 | Room content approval | draft, in_review, approved, published, archived | Author/reviewer actions | Prior forensic record |
| 9 | Assessment/problem | assessment: draft, published, active, archived; problem: draft, published, archived | Academic/admin actions | Prior forensic record |
| 10 | Certificate | issued, revoked | Credential service/admin | Prior forensic record; align Module 12 |
| 11 | Platform launch phase | A, B, C | Launch service; irreversible phases reported | Prior forensic record |
| 12 | Outbox event | pending, dispatched, failed, dead_letter | Dispatcher/worker retry | Prior forensic record; align Module 18 |

**Warning:** Status names are not sufficient to define a state machine. Every transition must have source state, target state, actor, guard, side effects, failure behavior, and reversibility.

## 5. Control topologies

| Topology | Example from prior report | Required functional controls |
|---|---|---|
| Synchronous request-driven | Tenant transitions, room approval, launch phase | Auth, permission, guard, atomic state mutation, response |
| Scheduled/wall-clock | Event watcher, dunning, final grading | Idempotency, clock/time-zone semantics, concurrency lock, missed-run behavior |
| Asynchronous/event-driven | Judge0, path aggregation, cancellation/refund, certificates | Event contract, retry, dedupe, ordering, terminal failure, observability |

## 6. Cross-cutting transition requirements

- State changes must be authorized server-side, including tenant/department/section/group and ownership checks.
- Client-supplied status values must not bypass transition guards.
- State updates must be atomic with required side effects or use a documented durable outbox/saga contract.
- Duplicate requests/events must not produce unintended repeated side effects.
- Concurrent transitions must not overwrite a newer state without detection.
- Terminal state, reversal, correction, cancellation, expiry, and deletion/purge semantics must be explicit.
- State-derived UI must distinguish invalid transition, permission failure, stale state, and service failure.
- Audit events for sensitive transitions must follow Module 15.
- Cross-module transitions must not bypass the owning module’s authorization or business rules.

## 7. Critical legacy defects reported

| ID | Finding | Impact | Current status |
|---|---|---|---|
| FSM-DEF-01 | Event registration lockout defect | Registration state/guard may prevent legitimate registration | Revalidate legacy code |
| FSM-DEF-02 | Skip-to-completion fraud | Skipped learning rooms may be treated as complete/certificate eligible | Critical; align Module 16/12 |
| FSM-DEF-03 | Tenant cancellation lockout | Cancellation reported irreversible with no recovery | Product/ops decision |
| FSM-DEF-04 | Missing payment-provider subscription webhook sync | Subscription state can diverge from provider | Billing integration gap |
| FSM-DEF-05 | Launch phase progression reported irreversible | Mistyped/incorrect phase cannot be reverted | Deployment/platform risk |
| FSM-DEF-06 | Outbox terminal/retry handling | Failed/dead-letter events can block downstream state | Align Module 18 |

## 8. Decision log

| ID | Decision | Considerations | Status |
|---|---|---|---|
| FSM-D01 | Canonical state catalog and naming | Legacy vs target states, enum ownership, migration | Open |
| FSM-D02 | Transition authority and role matrix | Actor, permission, tenant/ownership scope | Open |
| FSM-D03 | Tenant suspend/expire/cancel/purge semantics | Reversibility, data retention, downstream access | Open |
| FSM-D04 | Event schedule transitions | Time zone, registration windows, cron delays, manual overrides | Open |
| FSM-D05 | Event registration lockout and payment transitions | Eligibility, waitlist, cancellation/refund, retry | Open |
| FSM-D06 | Submission verdict terminality and rejudge | Pending → verdict, system_error retry, rejudge policy | Open |
| FSM-D07 | Room/path skip and completion semantics | Required/optional rooms, certificate gate, path completion | Open |
| FSM-D08 | User and invitation transition rules | Invite expiry, suspension, deletion, reinvite | Open |
| FSM-D09 | Subscription/invoice/payment transition contract | Provider authority, webhook order, dunning, refund | Open |
| FSM-D10 | Content approval transition matrix | Reviewer permissions, revision, publish/archive rules | Open |
| FSM-D11 | Assessment/problem publish and archive rules | Active attempts, edit freeze, withdrawal | Open |
| FSM-D12 | Certificate issue/revoke/reissue policy | Eligibility, revocation reasons, reissue, verification | Open |
| FSM-D13 | Platform launch phase reversal/rollback | Date-code control, operator access, emergency rollback | Open |
| FSM-D14 | Outbox event state/retry/dead-letter contract | Retry limit, replay, idempotency, operator recovery | Open |
| FSM-D15 | Transition transaction and side-effect atomicity | DB transaction vs outbox/saga | Open |
| FSM-D16 | Concurrency and optimistic locking | Concurrent admin actions, watcher vs manual transition | Open |
| FSM-D17 | Transition audit/event catalogue | Audit, notification, emitted events per state change | Open |
| FSM-D18 | State invalidation/cascade rules | Parent changes, dependent records, caches | Open |
| FSM-D19 | Time source and scheduled transition semantics | UTC, tenant timezone, daylight saving, late cron | Open |
| FSM-D20 | State history and correction policy | Current-state-only vs transition history, backfill | Open |
| FSM-D21 | Frontend state reflection and stale-state behavior | Refresh, optimistic UI, conflict response | Open |
| FSM-D22 | Cross-module canonical transition ownership | Modules 1–16 and Module 18 responsibility boundaries | Open |

## 9. Review status and readiness

- [x] Prior Module 17 forensic report reviewed.
- [x] Schema, Business Flow v2, RBAC, and Functional Specification Standard reviewed.
- [x] Twelve reported state machines catalogued.
- [x] Legacy defects and decision dependencies identified.
- [ ] Current implementation independently revalidated.
- [ ] State/transition matrix approved by module owners.
- [ ] Transition authority, concurrency, retry, reversal, and event side effects approved.
- [ ] Product, security, architecture, and QA sign-off complete.

**Implementation readiness:** Not ready until FSM-D01–D22 and linked module decisions are resolved.

## 10. Review workflow
1. Each owning module reviews its state and transitions.
2. Architecture defines shared transition/event/outbox invariants.
3. Product/security approves role, scope, reversibility, and sensitive transitions.
4. Module 18 resolves retries, ordering, concurrency, and partial-failure semantics.
5. QA updates decision-dependent tests.
6. Approved decisions become target requirements; legacy behavior remains separately labeled.
