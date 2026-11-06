# Module 10 — Groups (Cohorts & Student Grouping)

## 1. Module Executive Summary

Module 10 governs the organizational structures for grouping students and learners on the Dezolver platform for academic tracking, batch management, invitation routing, curriculum syllabus customisation, and real-time event broadcasting.

A rigorous forensic audit of the legacy codebase reveals a profound divergence between conceptual academic requirements ("Departments", "Sections", "Lab Groups", "Project Teams") and the actual legacy implementation:

1. **Entity Name & Granularity Discrepancy:** The legacy codebase has **zero entities named `Group`**, `Department`, or `Section`. Instead, the sole grouping primitive is **`Cohort`** (`tenancy.cohorts`), representing an institution's broad academic batch or graduation year (e.g., "CSE 2025", "2025-26").
2. **Scalar 1:N Membership (No Join Table):** A user belongs to at most **one cohort** via a single nullable scalar foreign key `cohort_id` on the `identity.users` table (`cohortId String? @map("cohort_id") @db.Uuid`). There is no many-to-many join table (`user_cohorts` does not exist).
3. **Immutable Lifecycle (Create & Read Only):** Cohorts cannot be edited, renamed, or deleted via the API. `TenantsController` provides `GET /tenants/:id/cohorts` and `POST /tenants/:id/cohorts`; no `PATCH`, `PUT`, or `DELETE` endpoints exist.
4. **Critical JWT Serialization Defect (Universal Cohort Context Loss):** `AuthService.buildLoginResponse` and `issueTokensForUser` omit `cohort_id` from the JWT claims payload during token signing. Consequently, `AuthGuard` extracts `req.principal!.cohortId` as `undefined` for **100% of authenticated users**, causing all downstream endpoints requiring cohort identity (such as `GET /me/syllabus/:syllabusId/effective`) to throw `403 Forbidden ({ code: 'no_cohort_assigned' })`.
5. **Disconnected Ingestion Channels:**
   - **Invitations:** While an invitation can record a `cohort_id`, the `TenantsService.acceptInvitation` method only marks the invitation record as accepted; it fails to link the accepting user to the specified cohort.
   - **Bulk CSV Import:** `TenantsController.importRoster` enqueues an asynchronous BullMQ job without passing `cohortId` or `s3Key`, severing the batch ingestion pipeline.
6. **Curriculum Overlays as the Primary Consumer:** The primary functional application of cohorts is cohort-specific curriculum tailoring via `curriculum.syllabus_overlays`, allowing institutions to hide, reorder, rename, and add syllabus nodes per cohort.

---

## 2. Directory Layout & Navigational Links

```
docs/functional-spec/module-10-groups/
├── README.md               # Executive summary, findings, and navigational guide (this file)
└── SPECIFICATION.md        # Comprehensive 17-Section (A–Q) legacy specification & traceability matrix
```

- **Detailed Functional Specification:** [`SPECIFICATION.md`](./SPECIFICATION.md)
- **Authoring Standard:** [`FUNCTIONAL-SPECIFICATION-STANDARD.md`](../../FUNCTIONAL-SPECIFICATION-STANDARD.md)
- **Master Index & Registry:** [`INDEX.md`](../INDEX.md)

---

## 3. Key Forensic Discoveries & Architecture Summary

| Architectural Area | Conceptual Platform Expectation | Actual Legacy Reality | Forensic Status |
|---|---|---|:---:|
| **Group Primitive** | Flexible nested groups (Colleges > Depts > Sections > Groups) | Single flat entity `Cohort` attached directly to `Tenant`. | `CONFIRMED LEGACY IMPLEMENTATION` |
| **Membership Topology** | M:N many-to-many join table with join dates and roles | 1:N scalar `cohort_id` on `identity.users`. Max 1 cohort per user. | `CONFIRMED LEGACY IMPLEMENTATION` |
| **CRUD Operations** | Full Create, Read, Update, Archive, Delete lifecycle | Create and Read only; no Update or Delete endpoints. | `PARTIALLY IMPLEMENTED` |
| **Session Hydration** | User JWT contains `cohort_id` for authorization | JWT signing completely omits `cohort_id`; principal context lacks it. | `DEFECT / BROKEN IMPLEMENTATION` |
| **Downstream Syllabus** | Students receive cohort-tailored curriculum overlays | Endpoint fails with `403 no_cohort_assigned` due to JWT defect. | `DEFECT / BROKEN IMPLEMENTATION` |
| **Real-time Channels** | WebSocket joins `cohort:<cohortId>` room | Auto-join fails on connect due to missing JWT cohort claim. | `DEFECT / BROKEN IMPLEMENTATION` |
| **Sub-cohort Groups** | Project teams, lab groups, coding pairs | 0 models, 0 tables, 0 endpoints. | `NOT IMPLEMENTED` |
| **Member Management** | Staff manually add/remove students from cohorts | 0 endpoints exist to reassign or remove a student from a cohort. | `NOT IMPLEMENTED` |

---

## 4. Source Code Traceability Index

| Subsystem / Layer | Source File Path | Primary Responsibility |
|---|---|---|
| **Database Schema** | `prisma/tenancy/schema.prisma` | Defines `Cohort` and `Invitation` models (`tenancy` schema) |
| **Identity Schema** | `prisma/identity/schema.prisma` | Defines `User.cohortId` scalar relation (`identity` schema) |
| **Curriculum Schema**| `prisma/curriculum/schema.prisma` | Defines `SyllabusOverlay` unique constraint per cohort |
| **Repository** | `src/tenancy/repositories/cohorts.repository.ts` | Data access methods: `findByTenant`, `findById`, `create` |
| **Tenancy Service** | `src/tenancy/services/tenants.service.ts` | Cohort listing, creation, and invitation creation |
| **Tenants Controller**| `src/tenancy/controllers/tenants.controller.ts` | REST endpoints: `GET /:id/cohorts`, `POST /:id/cohorts` |
| **Auth Guard & Context**| `src/shared/auth/auth.guard.ts` | Extracts `cohort_id` from JWT into `req.principal` |
| **Auth Service Defect**| `src/identity/services/auth.service.ts` | Signs JWT without `cohort_id` (Lines 253, 316) |
| **Curriculum Overlay** | `src/curriculum/controllers/overlays.controller.ts` | Cohort-specific syllabus mutation operations |
| **Student Syllabus** | `src/curriculum/controllers/me-syllabus.controller.ts`| `GET /me/syllabus/:syllabusId/effective` resolving overlay |
| **WebSocket Gateway**| `src/websocket/gateways/submissions.gateway.ts` | Auto-joins `cohort:<cohortId>` channel on connection |
| **Frontend Page** | `frontend/src/pages/tenant/cohorts/CohortsPage.tsx` | Cohort card list and `CreateCohortDialog` |
| **Frontend Hook** | `frontend/src/hooks/useCohorts.ts` | React Query hooks `useCohorts` and `useCreateCohort` |

