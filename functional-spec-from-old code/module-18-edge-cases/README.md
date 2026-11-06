# Module 18 — Edge Cases & Failure Handling

## 1. Module Executive Summary

Module 18 establishes the comprehensive architectural, operational, and forensic specifications for Error Handling, Edge Cases, Fault Tolerance, Resiliency, and Disaster Recovery across the entire Dezolver platform.

A forensic audit of all backend filters, guards, interceptors, queues, outbox mechanisms, and frontend error boundaries reveals a mature resilience subsystem designed to isolate multi-tenant blast radiuses, prevent cascading infrastructure failures, and handle network anomalies:

1. **Centralized Error Filter & Standardized Error Envelope:** All uncaught exceptions pass through `GlobalErrorFilter` (`src/shared/filters/error.filter.ts`), mapping NestJS `HttpException` instances and PostgreSQL Prisma error codes (`P2002` -> `409 Conflict`, `P2025` -> `404 Not Found`) into a consistent JSON envelope `{ error: { code, message, details, requestId } }` with distributed tracing UUIDs.
2. **Distributed Idempotency Engine:** Handled by `IdempotencyInterceptor` (`src/shared/interceptors/idempotency.interceptor.ts`) for all `POST`, `PUT`, and `PATCH` requests carrying an `Idempotency-Key` header. In-flight duplicates are rejected with `409 Conflict ({ code: 'idempotency_request_in_flight' })`, while completed responses are replayed from Redis with header `Idempotency-Replay: true` (24-hour TTL).
3. **Multi-Tenant Rate Limiting & Tiered Throttling:** Handled by `PerTenantRateLimitGuard` (`src/shared/rate-limit/per-tenant-rate-limit.guard.ts`) enforcing subscription plan quotas (`starter`: 1k/min, `professional`: 10k/min, `enterprise`: 50k/min, `unlimited`: unmetered) via Redis pipelines. Over-quota requests receive `429 Too Many Requests` with a `Retry-After: 60` header.
4. **Queue Fairness & Tenant Starvation Prevention:** BullMQ background worker middleware `createTenantFairnessMiddleware` (`src/shared/queue/tenant-fairness.guard.ts`) throttles background concurrency to a maximum of 3 concurrent jobs per tenant (`MAX_CONCURRENT_PER_TENANT = 3`), delaying excess jobs for automatic exponential retry.
5. **Dead-Letter Queue (DLQ) & Message Recovery:** Handled by `DlqService` (`src/shared/outbox/dlq.service.ts`). A 5-minute cron moves permanently failed outbox events to `platform.outbox_failed`, breaking infinite retry loops while preserving payloads for administrative manual replay (`DlqService.retryEvent`). A daily 5:00 AM IST cron hard-purges dispatched outbox records older than 30 days.
6. **Inactive Tenant & Account Isolation:** Handled by `RequireActiveTenantGuard` (`src/shared/guards/require-active-tenant.guard.ts`), intercepting requests from tenants in `['suspended', 'expired', 'cancelled', 'purged']` states with `403 Forbidden ({ code: 'tenant_inactive' })` (cached in Redis for 30s).
7. **Impersonation Safety Gate (Mutation Lockout):** Handled by `BlockImpersonationGuard` (`src/shared/guards/block-impersonation.guard.ts`), denying all mutating verbs (`POST`, `PUT`, `PATCH`, `DELETE`) with `403 Forbidden` during operator impersonation sessions unless explicitly bypassed via `@AllowImpersonation()`.
8. **Client-Side Fault Tolerance & Self-Healing:** The React frontend client (`frontend/src/services/api/client.ts`) implements automatic 401 token refresh with a subscriber queue to serialize concurrent retries, automatic `Idempotency-Key` generation on all mutations, and React `ErrorBoundary` components to isolate render crashes to individual widgets.
9. **Fail-Open Cache & Database Resilience:** Redis operates under a fail-open architecture (`retryStrategy: () => null`); if Redis is down, rate limiting, caching, and idempotency locks pass through to prevent system-wide outages. PostgreSQL database pool defaults to 2 connections per schema client with a 30-second timeout, with `Promise.allSettled` preventing single-schema startup crashes.

---

## 2. Directory Layout & Document Navigation

```
docs/functional-spec/module-18-edge-cases/
├── README.md               # Executive summary, resilience inventory, and architecture guide (this file)
└── SPECIFICATION.md        # Comprehensive 17-Section (A–Q) legacy specification & traceability matrix
```

- **Detailed Functional Specification:** [`SPECIFICATION.md`](./SPECIFICATION.md)
- **Authoring Standard:** [`FUNCTIONAL-SPECIFICATION-STANDARD.md`](../../FUNCTIONAL-SPECIFICATION-STANDARD.md) (`DEZ-STD-FS-001`)
- **Master Index & Registry:** [`INDEX.md`](../INDEX.md)

---

## 3. Resilience Architecture & Defense-in-Depth Pipeline

```
                                    INCOMING HTTP REQUEST
                                              │
                                              ▼
                                  ┌───────────────────────┐
                                  │   GlobalErrorFilter   │
                                  │   (Catches All Errs)  │
                                  └───────────┬───────────┘
                                              │
                                              ▼
                                  ┌───────────────────────┐
                                  │ EnvelopeInterceptor   │
                                  │   (Response Wrapper)  │
                                  └───────────┬───────────┘
                                              │
                                              ▼
                                  ┌───────────────────────┐
                                  │   AuditInterceptor    │
                                  │  (@AuditAction Logs)  │
                                  └───────────┬───────────┘
                                              │
                                              ▼
                                  ┌───────────────────────┐
                                  │ IdempotencyInterceptor│
                                  │ (Redis Replay / Lock) │
                                  └───────────┬───────────┘
                                              │
        ┌─────────────────────────────────────┴─────────────────────────────────────┐
        │                                                                           │
        ▼                                                                           ▼
┌───────────────────────┐                                               ┌───────────────────────┐
│       APP_GUARDS      │                                               │       APP_GUARDS      │
│  - AuthGuard          │                                               │  - RequireActiveTenant│
│  - PermissionGuard    │                                               │  - BlockImpersonation │
│  - PolicyGuard        │                                               │  - PerTenantRateLimit │
│  - RequireFeatureGuard│                                               │                       │
└───────────┬───────────┘                                               └───────────┬───────────┘
            │                                                                       │
            └─────────────────────────────────┬─────────────────────────────────────┘
                                              │
                                              ▼
                                  ┌───────────────────────┐
                                  │ Controller Handler    │
                                  │ (Business Logic)      │
                                  └───────────┬───────────┘
                                              │
                                              ▼
                                  ┌───────────────────────┐
                                  │ Database Transaction  │
                                  │ (Prisma Multi-Schema) │
                                  └───────────────────────┘
```

---

## 4. Forensic Inventory & Scope

| Component / Layer | Source File Path | Forensic Role & Behavior |
|---|---|---|
| **Global Error Filter** | `src/shared/filters/error.filter.ts` | Catches all exceptions; standardizes JSON error format and maps Prisma P2002/P2025. |
| **Idempotency Interceptor** | `src/shared/interceptors/idempotency.interceptor.ts` | Prevents duplicate mutations; caches and replays responses via Redis. |
| **Rate Limit Guard** | `src/shared/rate-limit/per-tenant-rate-limit.guard.ts` | Enforces plan-based rate limits (1k to 50k req/min) via Redis fixed-window counters. |
| **Queue Fairness Middleware**| `src/shared/queue/tenant-fairness.guard.ts` | Limits background worker concurrency to 3 jobs per tenant in BullMQ. |
| **Dead-Letter Queue Service**| `src/shared/outbox/dlq.service.ts` | 5-minute cron moves failed outbox events to DLQ table; 30-day archival cron. |
| **Active Tenant Guard** | `src/shared/guards/require-active-tenant.guard.ts` | Rejects requests from suspended, expired, or cancelled tenants (403). |
| **Impersonation Safety Guard**| `src/shared/guards/block-impersonation.guard.ts` | Denies mutating HTTP verbs during impersonation sessions (403). |
| **Feature Entitlement Guard** | `src/shared/guards/require-feature.guard.ts` | Verifies plan feature flags; returns `402 Payment Required` if feature disabled. |
| **Resource Policy Guard** | `src/shared/auth/policy.guard.ts` | ABAC policy guard; throws `404 Not Found` on authorization failure to hide resource existence. |
| **Health Controller** | `src/shared/health/health.controller.ts` | Exposes `/health`, `/health/metrics`, `/version`, and `/time`. |
| **Database Pool Configuration**| `src/shared/prisma/prisma.service.ts:17-27` | Enforces `connection_limit=2` and `pool_timeout=30s` across 11 Prisma clients. |
| **Redis Fail-Open Client** | `src/shared/redis/redis.service.ts:17` | Configures `retryStrategy: () => null` to allow graceful degradation without crashes. |
| **Frontend Axios Interceptor** | `frontend/src/services/api/client.ts:38-78` | Handles automatic 401 token refresh queue and injects idempotency headers. |
| **Frontend Error Boundary** | `frontend/src/components/shared/ErrorBoundary.tsx` | React class boundary preventing unhandled UI render crashes. |

---

## 5. Critical Defects & Edge Case Hazards Summary

| Defect ID | Severity | Location | Forensic Description & Blast Radius |
|---|---|---|---|
| **DEF-18-01** | `HIGH` | `src/shared/redis/redis.service.ts:17` | **Permanent Redis Disconnect Post-Failure**: `retryStrategy: () => null` prevents automatic reconnection after transient Redis network drops. The system remains permanently in fail-open mode until the container is restarted. |
| **DEF-18-02** | `HIGH` | `src/shared/filters/error.filter.ts:45-56` | **Incomplete Prisma Error Mapping**: Only `P2002` (unique violation) and `P2025` (not found) are mapped. Other common Prisma errors (e.g. `P2003` foreign key failure, `P2024` connection pool timeout) fall into the generic 500 handler, masking database pool exhaustion. |
| **DEF-18-03** | `MEDIUM` | `src/shared/interceptors/idempotency.interceptor.ts:38` | **Anonymous User Cache Collision**: For unauthenticated routes decorated with mutating methods, `userId` defaults to `'anonymous'`, causing multiple anonymous users submitting identical idempotency keys to conflict. |
| **DEF-18-04** | `MEDIUM` | `src/shared/rate-limit/per-tenant-rate-limit.guard.ts:39` | **Fixed-Window Burst Vulnerability**: 60-second fixed window allows users to execute $2\times$ their plan quota across window boundaries (e.g. 10k requests at $T=59\text{s}$ and 10k requests at $T=61\text{s}$). |
| **DEF-18-05** | `LOW` | `src/shared/outbox/dlq.service.ts:38` | **Premature Dispatch Timestamp on DLQ Move**: Moving failed events to `outbox_failed` updates the original event `dispatched_at = NOW()`. While this prevents re-polling, it corrupts audit logs by indicating the event was successfully dispatched. |

---

## 6. Verification & Traceability Sign-off

- **Negative Verification:** Verified error filters, guard pipelines, retry strategies, and DLQ implementations across all layers.
- **Traceability Matrix:** Fully documented in [`SPECIFICATION.md`](./SPECIFICATION.md#section-q-master-verification-traceability-matrix).
- **Adherence Standard:** Strictly complies with `DEZ-STD-FS-001`.

