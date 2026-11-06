# DEZOLVER — Module 18: Edge Cases & Failure Handling

**Document ID:** EDGE-FAILURE-FS-001  
**Version:** 0.1  
**Status:** Detailed draft — legacy resilience mechanisms captured; recovery, fail-open/fail-closed, and operational policies require approval  
**Governing standard:** [Functional Specification Standard v1.0](../FUNCTIONAL-SPECIFICATION-STANDARD.md)  
**Source reconstruction:** [Earlier Module 18 forensic specification](../../functional-spec-from-old%20code/module-18-edge-cases/SPECIFICATION.md)

> **Evidence caveat:** Current mechanisms and defects below are from the prior Module 18 forensic reconstruction, not fresh source checkout validation. They are legacy observations, not approved target behavior. Recommendations remain separate from requirements.

## 1. Purpose and mission

Module 18 defines platform-wide behavior when requests, data access, queues, caches, external services, and asynchronous workflows fail or encounter unusual conditions. It establishes predictable error contracts, safe retry and idempotency, rate limiting, tenant fairness, dead-letter recovery, health/readiness signals, and cross-module failure handling.

The prior forensic reconstruction reports:
- GlobalErrorFilter standardized HTTP error envelope.
- Redis-backed idempotency for mutation requests.
- Per-tenant plan-based rate limiting using a fixed window.
- Background worker tenant fairness (reported max 3 jobs per tenant).
- Outbox/DLQ processing with periodic archival/recovery.
- Redis fail-open behavior and a reconnection defect.
- Prisma multi-client connection pool constraints.
- Health/readiness endpoints and frontend error handling.
- Critical failure hazards: Redis reconnect, Prisma error masking, anonymous idempotency namespace collisions, fixed-window burst.

## 2. Scope

### In scope
- Standard error envelope and exception mapping.
- Request/trace correlation.
- Idempotency key lifecycle and response replay.
- Tenant rate limits and quota behavior.
- Worker tenant fairness.
- Outbox dispatch, retries, dead-letter, replay, archival.
- Redis outage/recovery and fail-open/fail-closed policy.
- Database connection pool exhaustion and dependency failures.
- External service timeout/retry/circuit breaker behavior (Judge0, email/SES, payment gateway, etc.).
- Health/readiness/degraded state.
- Frontend error, retry, refresh, and recovery contracts.
- Cross-module edge cases and failure catalog.

### Out of scope
- Domain-specific business rules except failure/edge behavior (Modules 1–17).
- Infrastructure deployment architecture beyond functional dependency/recovery behavior.
- Detailed monitoring vendor/dashboard configuration.
- Legal retention policy beyond operational failure artifacts and cross-reference Module 15.

## 3. Sources and evidence

| Source | Evidence |
|---|---|
| [Prior Module 18 forensic reconstruction](../../functional-spec-from-old%20code/module-18-edge-cases/SPECIFICATION.md) | Error filter, idempotency, rate limiting, tenant fairness, DLQ, Redis, Prisma pools, health checks, defects, and cross-module edge catalog |
| [Functional Specification Standard](../FUNCTIONAL-SPECIFICATION-STANDARD.md) | Function-level detail, evidence labels, acceptance, QA and decision gates |
| [RBAC](../../docs/source-text/RBAC.md) | Auth, tenant/scope, ownership, server-side access checks |
| [Business Flow v2](../../docs/source-text/Business%20Flow%20v2.md) | Main workflow dependencies and integrations |
| Modules 1–17 | Domain-specific states, permissions, events, side effects, and decision dependencies |

### Evidence labels
- **Source-confirmed:** explicit in supplied source docs.
- **Legacy-verified (prior forensic report):** behavior reported in earlier forensic reconstruction; not independently revalidated.
- **Derived:** consequence of multiple source facts.
- **Recommendation:** proposed target policy, not approved.
- **TBD / Decision Required:** policy or architecture choice remains open.

## 4. Shared error response contract

Prior report describes the envelope:

```json
{
  "error": {
    "code": "invalid_credentials",
    "message": "Invalid email or password",
    "details": null,
    "requestId": "trace-or-request-id"
  }
}
```

Reported mappings include:
- 400 Bad Request / validation
- 401 Unauthorized
- 402 Feature unavailable in plan
- 403 Forbidden / tenant inactive / impersonation mutation blocked
- 404 Not Found / record not found
- 409 Conflict / idempotency in-flight / unique constraint
- 422 Unprocessable Entity
- 429 Rate limit exceeded
- 500 Internal Server Error

**Target decisions:** exact envelope for every error class, safe details exposure, retryability flag, request ID propagation, and compatibility/versioning. Do not leak SQL, stack traces, credentials, provider payloads, or internal topology.

## 5. Functional inventory

| ID | Function | Evidence / status |
|---|---|---|
| ERR-F001 | Normalize errors into API envelope | Legacy-reported |
| ERR-F002 | Correlate requests and expose health/readiness | Legacy-reported |
| ERR-F003 | Idempotency for mutation requests | Legacy-reported; anonymous collision defect reported |
| ERR-F004 | Enforce plan-based tenant rate limits | Legacy-reported; fixed-window burst defect |
| ERR-F005 | Apply tenant fairness to background jobs | Legacy-reported |
| ERR-F006 | Dispatch outbox events and recover failures/DLQ | Legacy-reported |
| ERR-F007 | Redis dependency outage/degraded mode and reconnection | Legacy-reported; permanent reconnect defect |
| ERR-F008 | Handle database pool saturation and DB errors | Legacy-reported; Prisma error mapping gap |
| ERR-F009 | Handle external service timeouts/retries/circuit breakers | Some legacy behavior reported; target policy TBD |
| ERR-F010 | Frontend API error, token-refresh, and retry handling | Legacy-reported |
| ERR-F011 | Cross-module edge-case and partial-failure handling | Legacy report catalogues 18 modules |
| ERR-F012 | Purge/archive operational records safely | Legacy-reported DLQ archival; retention policy TBD |

## 6. Legacy mechanisms and defects

| ID | Reported behavior/defect | Consequence | Status |
|---|---|---|---|
| ERR-DEF-01 | Redis retry strategy reportedly returns null | Transient Redis outage may permanently disable reconnection until restart | Revalidate source |
| ERR-DEF-02 | Global error filter reportedly maps only P2002/P2025 explicitly | P2003/P2024 fall through to generic 500, hiding DB/pool issues | Revalidate source |
| ERR-DEF-03 | Idempotency key namespace reportedly uses 'anonymous' when principal absent | Unauthenticated requests can collide and potentially replay another response | Security-critical |
| ERR-DEF-04 | Fixed 60-second rate limit window | Burst can straddle window boundary and exceed effective short-period quota | Revalidate source |
| ERR-DEF-05 | Outbox failed-event movement and retention are cron-based | Delay, race, or recovery gaps if poller/cron overlap | Revalidate source |
| ERR-DEF-06 | Redis fail-open mode can disable cache/rate limit/idempotency controls | Availability vs security/integrity tradeoff is unresolved |
| ERR-DEF-07 | DB connection pool per client is reportedly constrained to 2 connections, pool timeout 30s | Pool exhaustion can produce timeouts and cascading failures |
| ERR-DEF-08 | Frontend refresh/retry queue can interact with mutations | Retry must preserve idempotency and avoid repeated side effects |

## 7. Cross-module edge-case principles

- Authentication/session: expired/revoked refresh token, brute-force lockout, concurrent refresh, role revocation (Module 1).
- Tenant/org: duplicate subdomain, suspended tenant, scope mismatch, cross-tenant references (Module 2).
- Billing: payment/webhook replay, past-due/dunning, provider outage, entitlement mismatch (Module 3).
- Courses/enrollment: invalid enrollment/path, content removed while active (Module 4).
- Problems/test cases: empty/invalid cases, hidden test leakage, concurrent edits (Module 5).
- Judge0/submission: timeout, network failure, malformed response, retry, duplicate result (Module 6).
- Evaluation: partial grading, rejudge, score update conflict (Module 7).
- Labs: routes reported absent; avoid frontend pretending unsupported actions exist (Module 8).
- Assessment: late submission, deadline boundary, duplicate attempt (Module 9).
- Groups: missing cohort, membership changes, scope mismatch (Module 10).
- Contests: registration window/lockout, cancellation/refund, tie/late result (Module 11).
- Certificates: missing recipient fields, duplicate issue, revocation/verification (Module 12).
- Notifications: lost/read state, channel failure, duplicate delivery (Module 13).
- Tickets: legacy route absence reported; UI/API must communicate capability accurately (Module 14).
- Audit: failed write, metadata redaction, read envelope mismatch (Module 15).
- Analytics: broken leaderboard, stale aggregation, skip fraud (Module 16).
- State machines: illegal transition, duplicate/out-of-order events, cancellation/reversal (Module 17).
- General edge behavior: all relevant transitions and side effects must follow Module 18 recovery policy.

## 8. Decision log

| ID | Decision | Considerations | Status |
|---|---|---|---|
| ERR-D01 | Canonical error envelope/version | Fields, code catalogue, requestId, details | Open |
| ERR-D02 | Error-to-HTTP/status mapping | Prisma, validation, provider, timeout, conflict | Open |
| ERR-D03 | Error detail/stack exposure | User-facing safe message vs internal diagnostic | Open |
| ERR-D04 | Request/trace correlation propagation | HTTP, queues, outbox, external services | Open |
| ERR-D05 | Idempotency key requirements and scope | Required routes, user/tenant/client namespace, anonymous behavior | Open |
| ERR-D06 | Idempotency retention and response replay | TTL, cached response fields, key reuse, status codes | Open |
| ERR-D07 | Idempotency handling for failures | Delete/retain key on 4xx/5xx, retryable errors | Open |
| ERR-D08 | Redis fail-open/fail-closed matrix | Rate limit, idempotency, cache, locks, auth controls | Open |
| ERR-D09 | Redis reconnect/backoff and recovery | Backoff, max attempts, readiness, reconnection alerts | Open |
| ERR-D10 | Rate limiting algorithm and quota policy | Fixed/sliding/token bucket, burst allowance, per route/IP/user/tenant | Open |
| ERR-D11 | Plan quota and rate-limit response contract | Plan changes, 429 headers, retry-after, exempt routes | Open |
| ERR-D12 | Tenant fairness scheduling | Job concurrency, priority, starvation, tenant weights | Open |
| ERR-D13 | Outbox retry/backoff/max attempts | Error classes, poison messages, retry ownership | Open |
| ERR-D14 | DLQ transfer/replay/retention | Operator permission, payload redaction, replay/dedupe | Open |
| ERR-D15 | External service timeout/circuit-breaker policy | Judge0, email, payments; thresholds, half-open state | Open |
| ERR-D16 | DB pool sizing and timeout behavior | Per-client limits, total connection budget, overload response | Open |
| ERR-D17 | Health vs readiness vs degraded response | Dependency checks, fail-open dependency status, orchestrator semantics | Open |
| ERR-D18 | Frontend retry/token-refresh behavior | 401 queue, safe retries, idempotency preservation | Open |
| ERR-D19 | Error observability and alerting | Logs, metrics, traces, PII redaction, SLOs | Open |
| ERR-D20 | Cross-module partial transaction/compensation policy | Outbox, saga, rollback, compensating action | Open |
| ERR-D21 | Archival/purge of failed/outbox records | Retention, audit, legal hold, recoverability | Open |
| ERR-D22 | Failure behavior for unsupported/unimplemented routes | 404 vs disabled capability response, UX accuracy | Open |
| ERR-D23 | Error contract compatibility | API clients, frontend, SDK, versioning | Open |
| ERR-D24 | Maintenance/degraded-mode operator controls | Kill switches, manual recovery, admin permissions | Open |

## 9. Review status and readiness

- [x] Prior Module 18 forensic reconstruction reviewed.
- [x] Standard, RBAC, business flow, and cross-module specs referenced.
- [x] Error/idempotency/rate limit/DLQ/Redis/DB/external service function inventory drafted.
- [x] Cross-module failure catalog and defects captured.
- [ ] Current source behavior independently revalidated.
- [ ] Error catalog and fail-open/fail-closed matrix approved.
- [ ] Idempotency, quota, retry, DLQ, and circuit-breaker policies approved.
- [ ] Operational SLO/alert/recovery policy approved.
- [ ] Security, architecture, product, and QA sign-off completed.

**Implementation readiness:** Not ready until ERR-D01–D24 and module-specific failure decisions are resolved.

## 10. Review workflow
1. Architecture/security approve failure-mode and fail-open/closed boundaries.
2. Backend/platform team approves error mapping, Redis, DB pool, rate-limit, and queue behavior.
3. Module owners confirm business-specific retries and compensating actions.
4. Frontend owners align error/retry and refresh contracts.
5. Operations approve health/readiness, alerting, DLQ replay, and retention.
6. QA updates decision-dependent expected results and resilience tests.
