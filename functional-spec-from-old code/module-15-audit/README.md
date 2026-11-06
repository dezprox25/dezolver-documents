# Module 15 — Audit Logging

## 1. Module Executive Summary

Module 15 establishes the compliance, security auditing, and forensic activity tracking subsystem of the Dezolver platform. It provides persistent, immutable tracking of administrative actions, authentication milestones, data modifications, and lifecycle transitions across all tenants.

A comprehensive forensic audit of the legacy codebase reveals an operational audit framework characterized by extensive controller instrumentation, but undermined by severe data loss at the interceptor level and critical serialization/contract incompatibilities between the NestJS backend and React frontend:

1. **Dedicated Database Schema & Storage Model:** A dedicated PostgreSQL schema `audit` containing table `audit.audit_entries` (Prisma model `AuditEntry` in `prisma/audit/schema.prisma`) stores audit records with fields `id`, `tenantId`, `actorUserId`, `action`, `resourceType`, `resourceId`, `ipAddress`, `userAgent`, `metadata`, and `occurredAt`.
2. **Extensive Controller Instrumentation:** Exactly 89 controller endpoints across all functional domains (identity, tenancy, courses, problems, curriculum, paths, assessments, submissions, events, credentials, billing, and platform settings) are decorated with `@AuditAction(...)` using the custom method decorator defined in `src/shared/auth/decorators.ts`.
3. **Severe Interceptor Data Loss (Null Resources & Empty Metadata):** The global `AuditInterceptor` (`src/shared/interceptors/audit.interceptor.ts`) instantiates audit entries with `resourceType: null`, `resourceId: null`, and `metadata: {}`. Despite the rich context available on incoming requests (route params, request bodies, target entity IDs, operational reasons), none of this context is extracted or saved. The audit log records *what* action occurred and *who* executed it, but completely omits *which entity* was affected.
4. **Fatal Envelope Interceptor / Frontend Contract Breakage:** `AuditController.list` returns `{ items: page, nextCursor }`. Because it lacks the `pagination` property expected by `EnvelopeInterceptor.isPaginatedResult`, the response is wrapped as `{ data: { items: [...], nextCursor: "..." } }`. The frontend service `auditService.list` (`frontend/src/services/api/audit.service.ts`) expects `res.data.data` to be an array and `res.data.pagination` to hold cursor state. Consequently, the frontend assigns an object to `items`, causing `AuditLogsPage.tsx` and `PlatformDashboardPage.tsx` to crash immediately with `TypeError: items.map is not a function`.
5. **Entity Field Name Inconsistencies:** The frontend TypeScript contract (`AuditEntry` in `frontend/src/types/platform.types.ts`) defines `actorId`, `targetId`, `targetType`, and `createdAt`. The database and backend return `actorUserId`, `resourceId`, `resourceType`, and `occurredAt`. Furthermore, the frontend expects `actorEmail` and `actorRole`, but the backend query executes a raw `findMany` without joining `identity.users` or `identity.persons`. When rendered against live data, date parsing fails (`Invalid Date`), and accessing `entry.actorId.slice(0, 8)` crashes the UI.
6. **"Security Events" Tab Dead-End:** The frontend `AuditLogsPage.tsx` filters security events with `action: 'auth.'`. However, all authentication and session actions in the backend use the `user.` namespace (`user.logout`, `user.login.success`, `user.mfa.enroll`, `user.impersonate.start`). Zero backend events match `'auth.'`, rendering the Security Events tab permanently blank.
7. **Failure Events Are Never Audited:** `AuditInterceptor` attaches via RxJS `tap(() => { ... })`, which triggers exclusively upon successful HTTP completion. If an attacker attempts unauthorized operations, triggers 403 Forbidden, submits malicious payloads, or causes a 500 Internal Error, the event is completely omitted from the audit trail.
8. **Double Logging on Impersonation:** Starting an impersonation session triggers `@AuditAction('user.impersonate.start')` on `UsersController.impersonate` and an explicit `prisma.audit.auditEntry.create` call inside `AuthService.issueImpersonationTokens`, generating two audit log entries for every session.
9. **Dead PII Redaction Decorator:** The `@LogRedact` decorator (`src/shared/auth/log-redact.decorator.ts`) was authored to redact sensitive fields, but `AuditInterceptor` never inspects `LOG_REDACT_KEY`, nor does it capture request bodies anyway.
10. **Role Gate Lockout for College Admins:** While the backend RBAC matrix (`src/shared/auth/permission.guard.ts`) grants `read:audit` to `college_admin` (with automatic tenant scoping), the frontend routing table (`frontend/src/routes/index.tsx`) restricts `/platform/audit` to `platform_admin` and `platform_moderator`, locking institutional administrators out of their own audit logs.

---

## 2. Directory Layout & Document Navigation

```
docs/functional-spec/module-15-audit/
├── README.md               # Executive summary, forensic findings, and architecture guide (this file)
└── SPECIFICATION.md        # Comprehensive 17-Section (A–Q) legacy specification & traceability matrix
```

- **Detailed Functional Specification:** [`SPECIFICATION.md`](./SPECIFICATION.md)
- **Authoring Standard:** [`FUNCTIONAL-SPECIFICATION-STANDARD.md`](../../FUNCTIONAL-SPECIFICATION-STANDARD.md) (`DEZ-STD-FS-001`)
- **Master Index & Registry:** [`INDEX.md`](../INDEX.md)

---

## 3. Subsystem Architecture & Audit Flow

```
                                      INCOMING HTTP REQUEST
                                                │
                                                ▼
                                    ┌───────────────────────┐
                                    │    JwtAuthGuard       │
                                    │  (establishes req.    │
                                    │      principal)       │
                                    └───────────┬───────────┘
                                                │
                                                ▼
                                    ┌───────────────────────┐
                                    │    PermissionGuard    │
                                    │   (checks RBAC perms) │
                                    └───────────┬───────────┘
                                                │
                                                ▼
                                    ┌───────────────────────┐
                                    │   AuditInterceptor    │
                                    │ (reads @AuditAction)  │
                                    └───────────┬───────────┘
                                                │
                                                ▼
                                    ┌───────────────────────┐
                                    │ Controller Handler    │
                                    │ (executes business    │
                                    │        logic)         │
                                    └───────────┬───────────┘
                                                │
                         ┌──────────────────────┴──────────────────────┐
                         │                                             │
             [SUCCESS: Observable Next]                    [ERROR: Thrown Exception]
                         │                                             │
                         ▼                                             ▼
             ┌───────────────────────┐                     ┌───────────────────────┐
             │   RxJS tap Handler    │                     │  GlobalErrorFilter    │
             │ Creates audit entry   │                     │  (HTTP 4xx / 5xx)     │
             │ in audit.audit_entries│                     │                       │
             │ [metadata: {},        │                     │  *** AUDIT SKIPPED ***│
             │  resourceId: null]    │                     │  (Zero failure logs)  │
             └───────────┬───────────┘                     └───────────────────────┘
                         │
                         ▼
             ┌───────────────────────┐
             │ EnvelopeInterceptor   │
             │ (wraps response JSON) │
             └───────────┬───────────┘
                         │
                         ▼
             ┌───────────────────────┐
             │ Express HTTP Response │
             └───────────────────────┘
```

---

## 4. Forensic Inventory & Scope

| Component / Layer | Source File Path | Forensic Role & Behavior |
|---|---|---|
| **Database Schema** | `prisma/audit/schema.prisma` | Defines `model AuditEntry` (`audit.audit_entries`) with indexes on tenant, actor, and timestamp. |
| **Prisma Client Client** | `src/shared/prisma/prisma.service.ts:58, 70` | Instantiates `this.audit` Prisma client targeting `node_modules/.prisma/audit`. |
| **Audit Decorator** | `src/shared/auth/decorators.ts:6, 23` | Defines `AUDIT_ACTION_KEY = 'auditAction'` and `@AuditAction(action: string)`. |
| **PII Redact Decorator** | `src/shared/auth/log-redact.decorator.ts:3, 13` | Defines `LOG_REDACT_KEY = 'log_redact'` and helper `redactFields` (dormant/orphaned). |
| **Global Interceptor** | `src/shared/interceptors/audit.interceptor.ts` | Global interceptor triggered by `@AuditAction`; writes fire-and-forget entries to DB. |
| **Registration in main** | `src/main.ts:58` | Injects `new AuditInterceptor(reflector, prisma)` into global interceptor pipeline. |
| **Query Controller** | `src/audit/controllers/audit.controller.ts` | Exposes `GET /audit/entries` for compliance queries with cursor pagination. |
| **Module Definition** | `src/audit/audit.module.ts` | Registers `AuditController`; imported into root `AppModule` (`src/app.module.ts:63`). |
| **Manual AuthService Logs**| `src/identity/services/auth.service.ts:269, 332` | Explicitly creates audit entries for `user.impersonate.start` and `user.login.success`. |
| **Security Controller** | `src/platform/controllers/admin-security.controller.ts:36, 48` | Audits token revocation (`admin.session.revoke`, `admin.sessions.revoke_all_for_tenant`). |
| **Frontend API Service** | `frontend/src/services/api/audit.service.ts` | HTTP client wrapper for `GET /audit/entries` with pagination metadata mapping. |
| **Frontend Query Hook** | `frontend/src/hooks/useAudit.ts` | TanStack Query `useInfiniteQuery` hook wrapping `auditService.list`. |
| **Frontend Page** | `frontend/src/pages/platform/audit/AuditLogsPage.tsx` | Main compliance dashboard rendering audit tables, timelines, and filter tabs. |
| **Frontend Dashboard** | `frontend/src/pages/platform/PlatformDashboardPage.tsx:110` | Platform overview displaying recent 5 audit events and link to `/platform/audit`. |
| **Frontend Routing** | `frontend/src/routes/index.tsx:550` | Role-gated route for `/platform/audit` restricted to `platform_admin` & `platform_moderator`. |
| **Frontend Types** | `frontend/src/types/platform.types.ts:3-27` | Defines frontend interfaces `AuditEntry` and `ListAuditParams`. |
| **Mock Adapter** | `frontend/src/mock/mockAdapter.ts:1152` | Intercepts `GET /audit/entries` during mock development, masking real backend wire defects. |

---

## 5. Critical Defects & Discrepancies Summary

| Defect ID | Severity | Location | Forensic Description & Blast Radius |
|---|---|---|---|
| **DEF-15-01** | `CRITICAL` | `src/shared/interceptors/envelope.interceptor.ts:30` vs `src/audit/controllers/audit.controller.ts:65` | **Pagination Envelope Mismatch**: Controller returns `{ items, nextCursor }`. `EnvelopeInterceptor` checks `'items' in data && 'pagination' in data`, failing and nesting payload as `{ data: { items, nextCursor } }`. Frontend `auditService.list` receives an object where an array was expected, crashing the UI with `items.map is not a function`. |
| **DEF-15-02** | `CRITICAL` | `src/shared/interceptors/audit.interceptor.ts:33-43` | **Total Resource & Metadata Erasure**: Interceptor sets `resourceType: null`, `resourceId: null`, and `metadata: {}` on all 89 audited endpoints. Audit logs fail to record which entity was modified or why. |
| **DEF-15-03** | `HIGH` | `src/types/platform.types.ts:3-16` vs `prisma/audit/schema.prisma:15-26` | **Schema Property Mismatch**: Frontend expects `actorId`, `targetId`, `targetType`, `createdAt`, `actorEmail`, and `actorRole`. Database provides `actorUserId`, `resourceId`, `resourceType`, and `occurredAt`. Causes `Invalid Date` and `TypeError: Cannot read properties of undefined (reading 'slice')`. |
| **DEF-15-04** | `HIGH` | `frontend/src/pages/platform/audit/AuditLogsPage.tsx:285` | **Security Tab Namespace Mismatch**: "Security Events" tab queries `action: 'auth.'`. All backend auth actions use `user.*`. The tab returns 0 results in perpetuity. |
| **DEF-15-05** | `HIGH` | `src/shared/interceptors/audit.interceptor.ts:31` | **Zero Failure Auditing**: Interceptor uses RxJS `tap()`, which only triggers on HTTP success. Failed logins, authorization rejections (403), validation errors (400), and exploit attempts are completely invisible to the audit log. |
| **DEF-15-06** | `MEDIUM` | `src/identity/controllers/users.controller.ts:144` vs `src/identity/services/auth.service.ts:269` | **Duplicate Impersonation Entries**: Impersonation endpoint has `@AuditAction` on the controller and an explicit `prisma.audit.auditEntry.create` call in the service, inserting two duplicate entries for every session. |
| **DEF-15-07** | `MEDIUM` | `src/shared/auth/log-redact.decorator.ts` | **Dead PII Redaction Code**: `@LogRedact` decorator is applied in `auth.controller.ts` but never inspected or processed anywhere in the backend pipeline. |
| **DEF-15-08** | `MEDIUM` | `src/audit/controllers/audit.controller.ts:56-58` | **Unstable Cursor Pagination**: Controller paginates using `cursor: { id: cursorId }` with `orderBy: { occurredAt: 'desc' }` without `id` as secondary sort, causing records with identical millisecond timestamps to be skipped or duplicated. |
| **DEF-15-09** | `LOW` | `frontend/src/routes/index.tsx:553` vs `src/shared/auth/permission.guard.ts:28` | **College Admin Route Lockout**: Backend allows `college_admin` to view tenant audit logs, but frontend route restricts access to platform roles, denying access to college admins. |
| **DEF-15-10** | `LOW` | `src/shared/interceptors/audit.interceptor.ts:45-47` | **Silent Audit Drop on Failure**: DB insert errors are caught with `.catch(() => {})` with no logging, metrics, or alerting, allowing silent audit loss under heavy DB contention. |

---

## 6. Verification & Traceability Sign-off

- **Negative Verification:** Audited all 11 database schemas, all backend controllers, and frontend pages.
- **Traceability Matrix:** Fully documented in [`SPECIFICATION.md`](./SPECIFICATION.md#section-q-master-verification-traceability-matrix).
- **Adherence Standard:** Strictly complies with `DEZ-STD-FS-001`.

