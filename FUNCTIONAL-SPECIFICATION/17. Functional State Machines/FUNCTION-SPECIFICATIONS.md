# Module 17 — Functional State Machines: Function-Level Specifications

**Document ID:** STATE-MACHINES-FUNCTIONS-001 | **Version:** 0.1 | **Status:** Draft; transition policies open  
**Standard:** [Functional Specification Standard](../FUNCTIONAL-SPECIFICATION-STANDARD.md)  
**Parent:** [Overview](./README.md) · [QA cases](./QA-TEST-CASES.md)

> The states/transitions below are legacy-reported by the prior Module 17 forensic report. Treat them as observed legacy behavior, not approved target state contracts. Where transition guards or side effects are unknown, decision IDs apply.

## Shared transition contract
For every state-changing command/event: authenticate actor where applicable → authorize role and scope → load current state → validate transition guard → apply state mutation atomically or via approved outbox/saga → record transition/audit → execute side effects idempotently → return resulting state. Client-supplied state must not bypass server guards. Concurrent transitions require conflict detection. **FSM-D01–D22** govern common behavior.

## FSM-F001 — Tenant lifecycle
**Status:** Legacy-reported; target policy TBD. **Actors:** Super Admin, tenant lifecycle system.
**Reported states:** pending, trial, active, suspended, expired, cancelled, purged.
**Purpose:** Control tenant activation, access, suspension, expiry, cancellation, and purge.
**Inputs:** Tenant ID, requested transition, actor, reason/effective time where required.
**Flow:** Load tenant → authorize global/tenant authority → validate current→target transition against allowlist → check subscription/onboarding/data dependencies → update tenant state → apply access/session/notification/audit side effects → return state.
**Rules:** Prior report says ALLOWED_TRANSITIONS enforced. Purge should not be treated as ordinary cancellation; retention and recoverability must be decided. Suspended/expired access consequences need cross-module mapping.
**Failures:** Illegal transition, concurrent change, missing tenant, dependency failure, partial side effect.
**Acceptance:** Only allowed transitions occur; tenant scope is enforced; downstream access matches approved state; destructive transition has approved confirmation/retention policy.
**QA:** FSM-QA-001–008. Decisions FSM-D02/D03/D15/D16/D18.

## FSM-F002 — Event/contest lifecycle
**Status:** Legacy-reported; timing and override policy TBD. **Actors:** Event admin, scheduled watcher, grading service.
**Reported states:** draft, published, registration_open, registration_closed, live, grading, completed, cancelled.
**Flow:** Create draft → authorized publish/open/close/live transitions or watcher evaluates event timestamps → guard checks configuration/time/registration/result state → state changes → notify/update downstream eligibility → grading/completion follows approved workflow.
**Rules:** Prior report says watcher runs every 30 seconds. Cron interval is not a guaranteed transition deadline. UTC/tenant timezone, manual override, delayed watcher, and cancellation after start are decisions.
**Failures:** Invalid transition, incomplete setup, watcher overlap, clock skew, missing grading, cancellation/refund failure.
**Acceptance:** Scheduled transitions are deterministic and idempotent; only authorized admins override; event state and participant eligibility remain consistent.
**QA:** FSM-QA-009–016. FSM-D04/D15/D16/D19.

## FSM-F003 — Event participant registration
**Status:** Legacy-reported; payment/registration policy TBD. **Actors:** Student/participant, event admin, payment worker.
**Reported states:** registered, waitlisted, pending_payment, cancelled, refunded.
**Flow:** Request registration → validate eligibility/capacity/payment requirement → create/update registration state → payment success/failure or waitlist resolution → cancellation/refund transition when permitted.
**Rules:** Registration lockout defect reported. Define capacity race, waitlist ordering, payment pending timeout, cancellation deadline, and refund eligibility. A refund state must not imply provider refund succeeded without confirmation.
**Failures:** Full event, duplicate registration, payment callback replay, event closed/cancelled, refund failure.
**Acceptance:** Registration is idempotent; capacity is not exceeded; payment and registration state reconcile; no unauthorized registration.
**QA:** FSM-QA-017–024. FSM-D02/D05/D15/D16.

## FSM-F004 — Code submission and evaluation verdict
**Status:** Legacy-reported; align Modules 6/7. **Actors:** Student submitter, Judge0 worker, evaluation service.
**Reported states/verdicts:** pending, accepted, partial, wrong_answer, time_limit_exceeded, memory_limit_exceeded, runtime_error, compilation_error, system_error; separate proctoring flag reported.
**Flow:** Submission accepted → pending execution → worker executes/evaluates → verdict and score persisted → terminal result published; system_error/rejudge follows approved retry/reprocessing path.
**Rules:** Verdict lifecycle and terminality must align with Module 6 submission state and Module 7 evaluation. Student must not set verdict. Rejudge must preserve history/version and avoid stale result overwrite.
**Failures:** Judge0 unavailable, timeout, malformed result, worker duplicate, stale rejudge, partial test results.
**Acceptance:** Only trusted evaluation worker can set verdict; submission/result linkage correct; replay is idempotent; rejudge policy traceable.
**QA:** FSM-QA-025–032. FSM-D06/D14/D15/D17.

## FSM-F005 — Learning path and room progress
**Status:** Legacy-reported; align Module 16 and certificate Module 12. **Actors:** Student, progress consumer.
**Reported room states:** not_started, in_progress, skipped, completed; path aggregate percent/completion.
**Flow:** Start/skip/complete room → validate room/path ownership and allowed transition → persist room state → emit progress event → aggregate path progress → emit PathCompleted only under approved rule.
**Rules:** Prior report says skip counted as completion; critical defect. Distinguish bypass from completion. Optional/non-room steps and zero-step path semantics must be decided. Completion cannot independently issue certificate.
**Failures:** Unauthorized room, duplicate event, raw SQL/DB failure, aggregator lag/retry, changed path steps.
**Acceptance:** State transitions follow Module 17; aggregate reconciles; skip does not grant credential eligibility by default; certificate checks Module 12.
**QA:** FSM-QA-033–040. FSM-D07/D15/D18 and Modules 12/16/18.

## FSM-F006 — User and invitation lifecycle
**Status:** Legacy-reported; identity policy TBD. **Actors:** Admin, invitee, auth/invitation service.
**Reported user states:** pending, active, suspended, deleted. Invitation: pending, accepted, expired, revoked.
**Flow:** Admin issues invitation → invite pending → acceptance validates token/expiry and account constraints → invitation accepted and user activation/association follows approved flow; admin can suspend/delete/revoke under permission.
**Rules:** User status and invitation status are separate machines. Define expiry, resend/reinvite, suspension effect on sessions, soft vs hard deletion, and restoring a deleted/suspended account.
**Failures:** Expired/revoked token, duplicate invite, email/account collision, stale acceptance, unauthorized admin.
**Acceptance:** Invitation can be accepted once; expired/revoked invites fail; account status and active sessions follow Module 1.
**QA:** FSM-QA-041–048. FSM-D02/D08/D15/D18.

## FSM-F007 — Subscription, invoice, and payment
**Status:** Legacy-reported; Module 3/payment provider decision dependencies. **Actors:** Finance/admin, customer, payment gateway, dunning worker.
**Reported states:** subscription trial/active/past_due/cancelled/unpaid; invoice draft/open/paid/void/uncollectible; payment created plus provider states.
**Flow:** Billing action/provider event → validate provider signature and object/version → map provider result to permitted billing transition → persist subscription/invoice/payment → trigger entitlement/dunning/receipt/audit effects.
**Rules:** Provider webhook is authoritative for payment confirmation only per approved integration contract. Prior report says subscription webhook sync is missing. Webhook order, retry, cancellation timing, grace period, and refund semantics TBD.
**Failures:** Duplicate/out-of-order webhook, signature invalid, payment success after cancellation, partial DB update, provider unavailable.
**Acceptance:** Verified webhook only; idempotent event processing; subscription entitlement consistent with canonical billing status; no client-side payment status trust.
**QA:** FSM-QA-049–056. FSM-D09/D14/D15/D16/D19.

## FSM-F008 — Room content approval
**Status:** Legacy-reported; approval permissions TBD. **Actors:** Content author, reviewer, publisher.
**Reported states:** draft, in_review, approved, published, archived.
**Flow:** Author creates draft → submits for review → reviewer approves/rejects (rejection path/state TBD) → authorized publisher publishes → archive action follows policy.
**Rules:** Author/reviewer separation, revision after approval, published content edit freeze, and archive impact on active learning paths are TBD.
**Failures:** Unauthorized approval, stale revision, publish before approval, content referenced by active course/path.
**Acceptance:** Only permitted reviewer/publisher transitions occur; published content is approved revision; archive does not silently break active learners.
**QA:** FSM-QA-057–064. FSM-D02/D10/D16/D18.

## FSM-F009 — Assessment and problem lifecycle
**Status:** Legacy-reported; align Modules 5/9. **Actors:** Author, academic admin, assessment scheduler.
**Reported assessment states:** draft, published, active, archived. Problem: draft, published, archived.
**Flow:** Draft creation/edit → validate configuration → publish → activate according to schedule/manual action → close/archive according to policy.
**Rules:** Define assessment schedule, active attempt behavior, published problem edits, withdrawal, and archive semantics. Problem publication and assessment publication are separate machines.
**Failures:** Invalid config, concurrent edit, attempt in progress, publish/activate conflict.
**Acceptance:** Only approved transitions; active learner attempts remain consistent; published problem changes follow versioning policy.
**QA:** FSM-QA-065–072. FSM-D02/D11/D16/D18/D19.

## FSM-F010 — Certificate credential
**Status:** Legacy-reported; Module 12 owns certificate rules. **Actors:** Credential service, authorized admin.
**Reported states:** issued, revoked.
**Flow:** Eligibility event → Module 12 validates eligibility → certificate issued → verification endpoint reflects active status; authorized revocation records reason and changes verification result.
**Rules:** Issuance eligibility, idempotency, revocation reason, reissue/replacement, immutable certificate ID, and verification behavior TBD. Analytics/path completion cannot bypass issuer eligibility.
**Failures:** Duplicate eligibility event, invalid recipient, revoke nonexistent/already revoked, stale verification cache.
**Acceptance:** Only eligible recipients receive credentials; repeated issuance event does not duplicate; revoked certificate no longer verifies as valid; history is retained per policy.
**QA:** FSM-QA-073–080. FSM-D12/D14/D17/D20.

## FSM-F011 — Platform launch phase
**Status:** Legacy-reported; operational safety decision required. **Actors:** Authorized platform operator/system.
**Reported phases:** A → B → C; date-based confirmation code (YYYYMMDD) and irreversible advancement reported.
**Flow:** Operator requests phase change → verify authority and confirmation challenge → validate current phase and target → apply phase/configuration change → record operator/time/audit → report active phase.
**Rules:** Prior report describes irreversible progression. Do not assume irreversibility is desirable. Define rollback/emergency downgrade, environment, confirmation validity, and separation of duties.
**Failures:** Wrong confirmation, unauthorized operator, concurrent phase update, partial rollout, rollback request.
**Acceptance:** Only authorized and confirmed transition; phase is consistent; rollback/irreversibility policy explicit; sensitive action audited.
**QA:** FSM-QA-081–088. FSM-D02/D13/D15/D16/D17.

## FSM-F012 — Transactional outbox dispatch
**Status:** Legacy-reported; align Module 18. **Actors:** Producer, dispatcher, queue consumer, operator.
**Reported states:** pending, dispatched, failed, dead_letter.
**Flow:** Business transaction creates outbox row → dispatcher claims pending item → publish event → mark dispatched on acknowledgement → failure records retry/error → after approved threshold dead-letter → operator replay/recovery if allowed.
**Rules:** Exactly-once delivery must not be assumed; use idempotent consumer semantics. Define retry count/backoff, lease/claim, event schema/version, ordering, dead-letter retention, and manual replay permissions.
**Failures:** Publish succeeded but acknowledgement lost, worker crash, duplicate dispatch, poison event, queue outage, dead-letter replay.
**Acceptance:** No silent loss; duplicate deliveries handled; state transitions observable; replay is authorized and does not duplicate business side effects.
**QA:** FSM-QA-089–096. FSM-D14/D15/D16/D17/D20 and Module 18.

## Function coverage
FSM-F001–FSM-F012 drafted. Transition matrices, ownership, reversal, side effects, and failure semantics require review under FSM-D01–D22.
