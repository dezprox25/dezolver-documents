# Module 01: Authentication & User Lifecycle

**Document Identifier:** `DEZ-SPEC-MOD-001`  
**Standard Compliance:** [`FUNCTIONAL-SPECIFICATION-STANDARD.md`](../FUNCTIONAL-SPECIFICATION-STANDARD.md) (`DEZ-STD-FS-001`)  
**Status:** `CONFIRMED LEGACY RECONSTRUCTION`  
**Investigation Date:** November 2026  
**Primary Specification:** [`SPECIFICATION.md`](./SPECIFICATION.md)  

---

## 1. Executive Summary

Module 01 governs identity creation, authentication, session lifecycle, credential verification, multi-factor authentication (MFA/TOTP), multi-tenant user isolation, role-based access control (RBAC), and data privacy compliance (DPDP/GDPR) across the Dezolver platform.

In the legacy implementation, the identity subsystem is partitioned into an isolated PostgreSQL schema (`identity`) managed by a dedicated NestJS domain module (`src/identity/identity.module.ts`) and consumed by client-side React routes (`frontend/src/pages/auth/`).

### Key Forensic Findings
1. **Person vs. User Entity Separation:**  
   The platform implements a two-tier identity model. A physical human is represented once by a `Person` record in the global identity schema (`persons` table), while a `User` record represents a tenant-specific membership persona (`users` table). A single Person can hold multiple active User accounts across different college tenants (`users_person_tenant_unique`).
2. **Asymmetric RS256 JWT & HttpOnly Refresh Cookies:**  
   Access tokens are signed using 2048-bit RSA keys with an RS256 algorithm and a 15-minute expiration (`15m`). Refresh tokens are 256-bit cryptographically random opaque tokens stored as SHA-256 hashes in the database with a 30-day lifetime (`30d`), delivered via `HttpOnly`, `SameSite=Strict`, `Secure` cookies (`dz_refresh`) scoped to `/api/v1/auth`.
3. **Multi-Layer Brute Force Defense:**  
   Redis pipelines enforce dual rate limits: max 5 consecutive failed attempts per tenant/email pair within a 15-minute sliding window (`account_locked_out`), and max 10 failed attempts per client IP within a 1-hour window (`ip_locked_out`).
4. **Hardware MFA (TOTP) with AES-256-GCM Encryption:**  
   MFA enrollment generates a 160-bit SHA-1 TOTP secret encrypted at rest using AES-256-GCM (`enc:<base64>` payload). Eight one-time recovery codes are hashed via bcrypt.
5. **Critical Implementation Divergences Identified:**  
   - **Reset Password DTO Mismatch:** The backend DTO expects `newPassword`, while the frontend client dispatches `password`, causing HTTP 400 validation failures.
   - **Signup Response Mismatch:** The backend returns `{ userId, personId }` (HTTP 201), but the frontend client expects `{ accessToken, user }`, causing silent session failures upon signup.
   - **MFA Enrollment Password Bypass:** The backend requires `currentPassword` in `MfaEnrollDto`, but the frontend service dispatches an empty POST body.
   - **Orphaned Recovery Codes:** Eight recovery codes are generated and hashed into `mfa_recovery_codes`, but no API endpoint or verification path exists to consume them.
   - **Missing Email Verification:** The `emailVerifiedAt` timestamp exists in the database schema, but no verification endpoint, token generator, or delivery mechanism was implemented.

---

## 2. Specification Navigation

The comprehensive specification for this module is structured into the seventeen mandatory standard sections:

| Section | Title | Primary Content |
|---|---|---|
| [Section A](./SPECIFICATION.md#section-a-module-overview) | **Module Overview** | Purpose, business objectives, boundaries, actors, glossary. |
| [Section B](./SPECIFICATION.md#section-b-legacy-implementation-architecture) | **Legacy Architecture** | NestJS containers, React state, Prisma multi-schema layout, Mermaid topology. |
| [Section C](./SPECIFICATION.md#section-c-database-specification) | **Database Specification** | Complete schema definitions for all 9 identity entities with constraints & indexes. |
| [Section D](./SPECIFICATION.md#section-d-complete-functional-inventory) | **Functional Inventory** | 16 discrete functions (`FN-AUTH-001` to `FN-AUTH-016`) mapped to 17-point specs. |
| [Section E](./SPECIFICATION.md#section-e-detailed-workflows) | **Detailed Workflows** | Sequence diagrams and 11-step execution traces for all auth operations. |
| [Section F](./SPECIFICATION.md#section-f-api-specification) | **API Specification** | 18 HTTP endpoints cataloged with DTOs, parameters, responses, and error codes. |
| [Section G](./SPECIFICATION.md#section-g-frontend-specification) | **Frontend Specification** | React pages, forms, Zustand store (`authStore.ts`), Monaco integration, demo personas. |
| [Section H](./SPECIFICATION.md#section-h-business-rules-and-calculations) | **Business Rules & Math** | Exact KaTeX formulas for brute-force lockouts, TOTP window validation, bcrypt rounds. |
| [Section I](./SPECIFICATION.md#section-i-permissions-and-security) | **Permissions & Security** | RBAC matrix (7 roles), RS256 token validation, tenant filter middleware, security audit. |
| [Section J](./SPECIFICATION.md#section-j-integrations-and-dependencies) | **Integrations & Dependencies** | Redis caching, BullMQ email/export queues, Outbox pattern event dispatch. |
| [Section K](./SPECIFICATION.md#section-k-edge-cases-and-failure-handling) | **Edge Cases & Failure Handling** | Network disconnects, Redis outage, token theft, race conditions, schema drift. |
| [Section L](./SPECIFICATION.md#section-l-state-machines-and-lifecycle) | **State Machines & Lifecycle** | Mermaid lifecycle diagrams for User status, PasswordReset, and RefreshToken. |
| [Section M](./SPECIFICATION.md#section-m-acceptance-criteria) | **Acceptance Criteria** | Formal Given/When/Then Gherkin scenarios for all 16 functions. |
| [Section N](./SPECIFICATION.md#section-n-qa-and-test-coverage-matrix) | **QA & Test Coverage Matrix** | 18 automated test cases covering positive, boundary, security, and fault paths. |
| [Section O](./SPECIFICATION.md#section-o-source-evidence-and-traceability) | **Source Evidence & Traceability** | Traceability matrix citing exact file paths, class names, and line numbers. |
| [Section P](./SPECIFICATION.md#section-p-unresolved-items-and-decision-log) | **Unresolved Items & Decisions** | 5 conflicting bugs, 3 missing legacy features, 4 open product decisions. |
| [Section Q](./SPECIFICATION.md#section-q-module-review-gate-checklist) | **Module Review Gate** | 16-point sign-off verification checklist. |

