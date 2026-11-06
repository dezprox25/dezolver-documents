# Module 18 — Edge Cases & Failure Handling: Function-Level Specifications

**Document ID:** EDGE-FAILURE-FUNCTIONS-001 | **Version:** 0.1 | **Status:** Draft; resilience policies open  
**Standard:** [Functional Specification Standard](../FUNCTIONAL-SPECIFICATION-STANDARD.md)  
**Parent:** [Overview](./README.md) · [QA cases](./QA-TEST-CASES.md)

> Reported current behavior is from prior Module 18 forensic analysis. It is not automatically target behavior. Decisions ERR-D01–D24 govern unresolved resilience/security behavior.

## Shared resilience principles
- Do not expose secrets, stack traces, SQL, provider tokens, or internal infrastructure details in client errors.
- Preserve request/trace correlation through HTTP, outbox, queue, and external-service operations where supported.
- Retrying a mutation must not repeat business side effects; use idempotency/outbox/dedupe contracts.
- Fail-open is a risk decision, not a universal default. Rate limits, auth, tenant isolation, idempotency, and critical billing/security operations need explicit dependency outage behavior.
- Availability, data integrity, security, and user-visible status must be balanced by operation class.

## ERR-F001 — Normalize errors into API response envelope
**Status:** Legacy-reported GlobalErrorFilter. **Actors:** API middleware/filter, client.
**Purpose:** Return consistent and safe errors across HTTP/domain/database failures.
**Inputs:** Exception category, request ID, safe message/code, optional safe details.
**Flow:** Catch exception → classify → map status/code → redact unsafe details → attach request correlation → log diagnostic details server-side → return response.
**Reported envelope:** `{ error: { code, message, details, requestId } }`.
**Rules:** Prisma P2002/P2025 reportedly map to conflict/not-found; other codes reportedly fall through. Target mapping needs full catalogue. Never leak stack/SQL/provider secrets. Preserve custom domain error codes.
**Failures:** Error filter itself fails; malformed exception; unknown Prisma/provider error.
**Acceptance:** Stable schema; appropriate status; no sensitive internal details; request ID correlates server logs.
**QA:** ERR-QA-001–008. **Decisions:** ERR-D01–D04/D23.

## ERR-F002 — Correlate requests and expose health/readiness
**Status:** Legacy-reported health and time endpoints. **Actors:** Monitoring/orchestrator/client.
**Purpose:** Make request diagnostics and dependency health observable.
**Inputs:** Request trace headers; dependency probes; app version/build/commit.
**Flow:** Accept/generate trace ID → propagate context → health probe evaluates dependencies → report status/dependency details per approved visibility → readiness signals whether instance can serve.
**Rules:** Prior report mentions public GET /health and GET /time. Health vs readiness vs liveness is not clearly separated. Do not expose sensitive internal info publicly.
**Acceptance:** Trace IDs survive request/worker boundary where supported; readiness accurately reflects serving capability; public health response safe.
**QA:** ERR-QA-009–014. **Decisions:** ERR-D04, D17, D19.

## ERR-F003 — Idempotency for mutation requests
**Status:** Legacy-reported Redis interceptor; anonymous collision defect. **Actors:** API client/interceptor/handler.
**Inputs:** Idempotency-Key, authenticated principal, tenant, method/route, request fingerprint.
**Flow:** Validate key and scope → lookup existing record → processing returns conflict or waits per policy; completed returns replayed response → otherwise claim key → execute handler → store successful result → release/expire key according to outcome.
**Reported behavior:** Redis key by user/key; processing TTL reportedly 30s; completed response TTL 24h; concurrent key yields 409; completed response replay.
**Rules:** Anonymous fallback key namespace is a reported vulnerability. Scope should avoid cross-user/tenant collision. Key must be bound to operation/request body; reuse with different payload needs defined conflict. Do not cache sensitive response fields blindly.
**Failures:** Redis unavailable, handler failure, key lock expiry during long operation, ack lost, concurrent request.
**Acceptance:** Same logical request does not duplicate mutation; another actor cannot replay response; changed payload/key reuse is handled; failure/retry policy explicit.
**QA:** ERR-QA-015–023. **Decisions:** ERR-D05–D09.

## ERR-F004 — Enforce tenant rate limits
**Status:** Legacy-reported fixed-window Redis guard; boundary burst defect. **Actors:** API gateway/guard, tenant/user/client.
**Inputs:** Tenant identity, plan quota, route/operation class, request timestamp.
**Flow:** Derive trusted tenant/identity → load quota → increment limiter atomically → compare quota → permit or return 429 with approved retry metadata.
**Reported quotas:** starter 1,000/min, professional 10,000/min, enterprise 50,000/min, unlimited plan unmetered. Confirm current plan values before implementation.
**Rules:** Fixed window reportedly allows near double quota across boundary. Algorithm (sliding window/token bucket), burst allowance, route exemptions, and per-IP/user/tenant layering are TBD.
**Failures:** Redis unavailable, quota lookup failure, plan change mid-window, race.
**Acceptance:** Limits enforce approved quotas; concurrent requests do not bypass count; response has correct retry info; fail-open/closed policy is applied per risk class.
**QA:** ERR-QA-024–032. **Decisions:** ERR-D08/D10/D11.

## ERR-F005 — Apply tenant fairness to background jobs
**Status:** Legacy-reported middleware limiting concurrency to 3 jobs/tenant. **Actors:** BullMQ scheduler/worker.
**Purpose:** Prevent one tenant’s batch work from starving others.
**Inputs:** Tenant ID from trusted job payload/context, job priority/type, concurrency configuration.
**Flow:** Validate tenant context → schedule/claim job within fairness cap → execute → release slot on success/failure/timeout → make eligible queued jobs available.
**Rules:** Max 3 per tenant is legacy-reported, not target-approved. Priority, global cap, tenant weights, interactive vs batch fairness, and missing tenant context behavior TBD.
**Failures:** Worker crash while holding slot, stale lock, poison job, tenant floods queue.
**Acceptance:** Fairness cap applied; slot is released/recovered; one tenant cannot indefinitely starve others; scope is not client-forged.
**QA:** ERR-QA-033–038. **Decisions:** ERR-D12/D19.

## ERR-F006 — Dispatch outbox events and recover failures/DLQ
**Status:** Legacy-reported outbox/DLQ. **Actors:** Outbox poller, BullMQ worker, DLQ service, operator.
**Inputs:** Event ID, tenant, aggregate, event type/version, payload, retry count.
**Flow:** Claim pending event → publish to queue → acknowledge/mark dispatched → worker executes → failed event records failure/retry → DLQ threshold reached → archive failure metadata/payload → authorized operator may inspect/replay.
**Reported behavior:** DLQ cron every 5 minutes moves failed undelivered events to outbox_failed; daily 5 AM IST purge reportedly deletes dispatched outbox rows older than 30 days.
**Rules:** Delivery is at-least-once unless proven otherwise. Consumer idempotency required. Retry policy, dead-letter threshold, retention, payload masking, replay authorization, and event ordering TBD.
**Failures:** publish ack lost, duplicate dispatch, worker crash, poison payload, DLQ transfer race, replay failure.
**Acceptance:** No silent event loss; duplicate effects controlled; dead-letter is recoverable and audited; retention does not remove records under hold.
**QA:** ERR-QA-039–049. **Decisions:** ERR-D13/D14/D20/D21; Module 15/17.

## ERR-F007 — Redis outage, degraded mode, and reconnection
**Status:** Prior report describes fail-open Redis and retryStrategy null defect. **Actors:** Redis client, API/worker, operations.
**Purpose:** Maintain defined service behavior during Redis loss and restore controls after recovery.
**Inputs:** Redis connection status, dependent feature (rate limit/idempotency/cache/lock), retry policy.
**Flow:** Detect connection failure → classify impacted controls → enter approved degraded mode → emit telemetry/alert → retry with bounded backoff → reconnect → reconcile transient state/cache → return normal.
**Rules:** Fail-open is not safe for all functions. Security/rate-limit/idempotency behavior must be operation-specific. Prior report says no retry after transient drop; revalidate.
**Acceptance:** Reconnection works; degraded mode is explicit; critical controls do not silently disappear; readiness/alerts reflect dependency state.
**QA:** ERR-QA-050–057. **Decisions:** ERR-D08/D09/D17/D19.

## ERR-F008 — Handle database pool saturation and DB errors
**Status:** Legacy-reported Prisma multi-client pool configuration. **Actors:** API/worker, Prisma, DB.
**Inputs:** Query/transaction, pool availability, DB error code, timeout.
**Flow:** Request obtains DB connection → execute operation → classify errors/timeouts → return mapped safe error or retry only if operation is safe → emit diagnostics and metrics.
**Reported configuration:** 11 Prisma clients × connection_limit=2 per node (22 potential connections), pool_timeout=30s; verify actual current configuration.
**Rules:** P2002/P2025 mapping reported; P2003 foreign-key failure and P2024 pool timeout reportedly generic 500. Do not automatically retry non-idempotent mutations.
**Failures:** Pool exhausted, DB down, deadlock, constraint failure, transaction timeout.
**Acceptance:** Pool timeout is visible and correctly mapped; DB error does not leak SQL; safe retry semantics are explicit; no unbounded request queue.
**QA:** ERR-QA-058–065. **Decisions:** ERR-D02/D16/D19/D20.

## ERR-F009 — Handle external service failure, timeout, retry, and circuit breaker
**Status:** Some Judge0/email/payment retry behavior legacy-reported; target policies TBD. **Actors:** External service client, queue worker, operator.
**Services:** Judge0, email/SES, payment provider, other external integrations.
**Flow:** Call with timeout → classify failure (transient/permanent/validation) → retry only if safe/idempotent → after threshold open circuit → fail fast/queue fallback → probe half-open → restore on success.
**Rules:** Circuit-breaker threshold and reset duration are recommendations in prior report, not approved. Payment/side-effect APIs require provider idempotency. Judge0 result retry must not duplicate submission verdict.
**Acceptance:** Timeout bounded; transient retry uses approved backoff; permanent errors not endlessly retried; circuit state observable; provider-specific behavior documented.
**QA:** ERR-QA-066–074. **Decisions:** ERR-D15/D13/D19/D20.

## ERR-F010 — Frontend API error, refresh, and retry handling
**Status:** Legacy-reported Axios 401 refresh queue and ErrorBoundary. **Actors:** Frontend client/user.
**Purpose:** Recover from session expiry and transient failures without duplicating mutations or misleading users.
**Flow:** API request → parse normalized error → on eligible 401 queue refresh once → replay safe requests with same idempotency key → show domain/error feedback → allow user retry/reset when appropriate.
**Rules:** Do not retry 403/validation errors as auth refresh. Retry mutations only with idempotency/approved retry policy. Avoid infinite refresh loops. ErrorBoundary isolates render crash but must not hide persistent API errors.
**Acceptance:** Concurrent 401 requests share refresh; failed refresh logs user out safely; request replay does not duplicate mutations; error code maps to appropriate UI state.
**QA:** ERR-QA-075–082. **Decisions:** ERR-D01/D06/D07/D18/D23.

## ERR-F011 — Apply cross-module edge-case and partial-failure handling
**Status:** Legacy report catalogs edge cases across Modules 1–18. **Actors:** Module services, shared infrastructure.
**Purpose:** Ensure domain-specific exceptional conditions are not silently dropped or inconsistently represented.
**Flow:** Module operation encounters edge case → owning module validates domain rule → shared error/retry/outbox policy applied → compensation/reconciliation if needed → user/system receives explicit result → event/audit/telemetry recorded as applicable.
**Examples:** auth lockout/session revocation; tenant uniqueness/suspension; payment dunning; unauthorized path room; invalid test cases; Judge0 timeout; late assessment; missing cohort; contest cancellation/refund; certificate render missing recipient; notification persistence loss; unsupported ticket endpoint; audit write loss; analytics stale/broken state; illegal transition.
**Rules:** Module owner remains source of business rule; Module 18 defines shared resilience mechanics only. Never turn a missing route into synthetic success.
**Acceptance:** Every module edge case has an owner, expected status/state, retry/compensation policy, and QA trace.
**QA:** ERR-QA-083–090. **Decisions:** ERR-D20/D22/D24 and all relevant module decisions.

## ERR-F012 — Archive/purge resilience records safely
**Status:** Prior report says DLQ/outbox archival purge exists; retention and recovery policy TBD. **Actors:** Scheduled job, authorized operator.
**Purpose:** Remove old operational records without destroying recoverability, audit evidence, or unresolved failures.
**Inputs:** Record state, age, retention threshold, legal hold, resolved status.
**Flow:** Scheduled job identifies eligible records → check resolved/retention/hold → archive or purge according to policy → record counts/results → alert on partial failure.
**Rules:** Prior report mentions dispatched outbox older than 30 days purged daily 5 AM IST. Treat as legacy, not approved retention. Failed/dead-letter and legal-held records must not be purged blindly.
**Acceptance:** Only eligible records removed; failures observable; retention/hold respected; recovery data preserved per policy.
**QA:** ERR-QA-091–096. **Decisions:** ERR-D14/D21.

## Function coverage
ERR-F001–ERR-F012 drafted. Shared failure, security, recovery, and operations policies remain open under ERR-D01–D24.
