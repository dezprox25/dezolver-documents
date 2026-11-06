# Module 02: College / Department / Section Management

**Document Identifier:** `DEZ-SPEC-MOD-002`  
**Standard Compliance:** [`FUNCTIONAL-SPECIFICATION-STANDARD.md`](../FUNCTIONAL-SPECIFICATION-STANDARD.md) (`DEZ-STD-FS-001`)  
**Status:** `CONFIRMED LEGACY RECONSTRUCTION`  
**Investigation Date:** November 2026  
**Primary Specification:** [`SPECIFICATION.md`](./SPECIFICATION.md)  

---

## 1. Executive Summary

Module 02 governs collegiate multi-tenancy, institutional branding, cohort grouping, student invitation lifecycles, bulk roster ingestion, and statutory tenant maintenance across the Dezolver platform.

In the legacy implementation, multi-tenancy is partitioned into an isolated PostgreSQL schema (`tenancy`) managed by a dedicated NestJS domain module (`src/tenancy/tenancy.module.ts`) and consumed by both platform super-admin portals (`frontend/src/pages/platform/tenants/`) and collegiate admin dashboards (`frontend/src/pages/tenant/`).

### Key Forensic Findings
1. **Absence of Separate Department and Section Entities:**  
   The legacy codebase possesses **0 database models, 0 API endpoints, and 0 frontend views for "Department" or "Section"**. Academic hierarchy below the College (`Tenant`) is modeled exclusively via the `Cohort` entity (`tenancy.cohorts`), which captures classroom or batch divisions using a descriptive `name` (e.g., "Computer Science 2026 - Section A") and an optional `academicYear` string (e.g., "2025-2026").
2. **Formal 7-State Tenant Lifecycle Machine:**  
   The platform implements a strict state transition engine in `TenantLifecycleService`:
   $$\text{pending} \longrightarrow \text{trial} \longrightarrow \text{active} \rightleftarrows \text{suspended} \longrightarrow \text{cancelled} \longrightarrow \text{purged}$$
   Illegal transitions are rejected with HTTP 400 Bad Request (`invalid_transition`). Every transition emits a transactional `TenantStatusChanged` outbox event.
3. **Automated Trial Subscription Provisioning:**  
   Creating a collegiate tenant via `POST /api/v1/tenants` automatically queries `billing.plans` and creates an active 14-day trial subscription in `billing.subscriptions`, linking tenancy creation directly to subscription enforcement.
4. **Automated Dunning & Inactivity Maintenance:**  
   A daily cron job at 03:00 AM (`TenantMaintenanceService.suspendOverdueTenants`) automatically identifies colleges whose subscription status has been `past_due` for $> 7$ days and transitions the tenant to `suspended`, causing all active user refresh tokens to be revoked.
5. **Critical Broken Integration Identified in CSV Roster Import:**  
   While `CsvImportConsumer` is fully implemented to stream CSV files from AWS S3, hash passwords, and create `User` records with `status: 'invited'`, the calling controller endpoint `TenantsController.importRoster` enqueues a BullMQ job containing only `{ tenantId, uploadedBy }`—omitting `s3Key` and `type`. As a result, invoking CSV import causes the background worker to fail immediately with an S3 error.

---

## 2. Specification Navigation

The comprehensive specification for this module is structured into the seventeen mandatory standard sections:

| Section | Title | Primary Content |
|---|---|---|
| [Section A](./SPECIFICATION.md#section-a-module-overview) | **Module Overview** | Purpose, business objectives, boundaries, actors, glossary. |
| [Section B](./SPECIFICATION.md#section-b-legacy-implementation-architecture) | **Legacy Architecture** | NestJS containers, React state, Prisma multi-schema layout, Mermaid topology. |
| [Section C](./SPECIFICATION.md#section-c-database-specification) | **Database Specification** | Complete schema definitions for `Tenant`, `Cohort`, `Invitation` with constraints & indexes. |
| [Section D](./SPECIFICATION.md#section-d-complete-functional-inventory) | **Functional Inventory** | 17 discrete functions (`FN-TEN-001` to `FN-TEN-017`) mapped to 17-point specs. |
| [Section E](./SPECIFICATION.md#section-e-detailed-workflows) | **Detailed Workflows** | Sequence diagrams and 11-step execution traces for all tenancy operations. |
| [Section F](./SPECIFICATION.md#section-f-api-specification) | **API Specification** | 13 HTTP endpoints cataloged with DTOs, parameters, responses, and error codes. |
| [Section G](./SPECIFICATION.md#section-g-frontend-specification) | **Frontend Specification** | React admin views, TanStack Query hooks, forms, Zod schemas, UI states. |
| [Section H](./SPECIFICATION.md#section-h-business-rules-and-calculations) | **Business Rules & Math** | Subdomain regex, 14-day trial math, 7-day dunning window, cursor pagination. |
| [Section I](./SPECIFICATION.md#section-i-permissions-and-security) | **Permissions & Security** | RBAC matrix, `:id === 'self'` resolution, multi-tenant middleware scoping. |
| [Section J](./SPECIFICATION.md#section-j-integrations-and-dependencies) | **Integrations & Dependencies** | AWS S3, BullMQ `csv-import` queue, Outbox pattern, Billing subscription sync. |
| [Section K](./SPECIFICATION.md#section-k-edge-cases-and-failure-handling) | **Edge Cases & Failure Handling** | Subdomain collision, invalid state transition rejection, S3 missing key failure. |
| [Section L](./SPECIFICATION.md#section-l-state-machines-and-lifecycle) | **State Machines & Lifecycle** | Mermaid lifecycle diagrams for Tenant status and Invitation lifecycle. |
| [Section M](./SPECIFICATION.md#section-m-acceptance-criteria) | **Acceptance Criteria** | Formal Given/When/Then Gherkin scenarios for all 17 functions. |
| [Section N](./SPECIFICATION.md#section-n-qa-and-test-coverage-matrix) | **QA & Test Coverage Matrix** | 12 automated test scenarios covering positive, boundary, security, and fault paths. |
| [Section O](./SPECIFICATION.md#section-o-source-evidence-and-traceability) | **Source Evidence & Traceability** | Traceability matrix citing exact file paths, class names, and line numbers. |
| [Section P](./SPECIFICATION.md#section-p-unresolved-items-and-decision-log) | **Unresolved Items & Decisions** | Proof of missing Department/Section, CSV job payload bug, product decisions. |
| [Section Q](./SPECIFICATION.md#section-q-module-review-gate-checklist) | **Module Review Gate** | 16-point sign-off verification checklist. |

