# Module 10 — Groups (Cohorts & Student Grouping)
## Comprehensive Functional Specification & Forensic Reconstruction

**Standard:** [`FUNCTIONAL-SPECIFICATION-STANDARD.md`](../../FUNCTIONAL-SPECIFICATION-STANDARD.md) (`DEZ-STD-FS-001`)  
**Domain:** Multi-Tenant Organization, Learner Grouping, Cohort Scoping, and Curriculum Customisation  
**Legacy Package:** `src/tenancy/`, `src/identity/`, `src/curriculum/`, `frontend/src/pages/tenant/cohorts/`  
**Reconstruction State:** `COMPLETED & FORENSICALLY VERIFIED`  
**Investigation Mode:** Strict Forensic Codebase Extraction (Zero Hallucination, Zero Assumptions)  

---

## SECTION A: Module Overview & Scope Boundaries

### A.1 Functional Definition
Module 10 defines the organizational primitives and scoping mechanisms used to partition, group, and manage students within a college or institution on the Dezolver platform. In standard academic and technical education workflows, grouping mechanisms govern batch management, class scheduling, section-based problem/assessment assignments, cohort-specific curriculum tracking, and peer communication.

### A.2 Implementation Reality & Divergence
A forensic audit of the legacy source code reveals a stark divergence between enterprise academic expectations and the actual implemented codebase:
1. **The "Group" Primitive Does Not Exist:** There is no database entity, table, TypeScript model, or service named `Group`, `Department`, `Section`, or `LabGroup`. Across all 11 PostgreSQL schemas in `prisma/` and all NestJS modules in `src/`, the sole grouping primitive is **`Cohort`** (`tenancy.cohorts`).
2. **Flat Hierarchy (No Nesting):** Cohorts are attached directly to a `Tenant` (`tenancy.tenants`). There are no intermediate layers such as Schools, Departments, Faculties, or Programs.
3. **Broad Batch Semantics:** Cohorts represent high-level graduating classes or admission years (e.g., "CSE 2025", "Class of 2026") rather than flexible, ad-hoc student project groups or classroom sections.
4. **Single-Membership Model (1:N):** Student association to a cohort is implemented as a single, nullable scalar foreign key `cohort_id` directly on the `identity.users` table. Dezolver does **not** support many-to-many group membership. A student can belong to at most one cohort at any given time.
5. **Read-Only / Append-Only Cohort Model:** The backend API exposes endpoints to create and list cohorts, but provides **zero endpoints to update, rename, archive, or delete a cohort**.

### A.3 Scope Boundaries

#### In-Scope (Implemented & Traced in Code)
- Database schema for `tenancy.cohorts` with tenant scoping.
- Creation of cohorts via `POST /tenants/:id/cohorts` and listing via `GET /tenants/:id/cohorts`.
- Assigning `cohort_id` to pending invitations via `POST /tenants/:id/invitations`.
- Background worker ingestion support for assigning `cohortId` during CSV roster import (`CsvImportConsumer`).
- Downstream curriculum syllabus tailoring via `curriculum.syllabus_overlays` bound to `(syllabusId, cohortId, nodeId)`.
- WebSocket room auto-joining (`cohort:<cohortId>`) and authorization rules in `ChannelAuthorizers`.
- Frontend administrative interface for listing cohorts and triggering creation modal (`CohortsPage.tsx`).

#### Out-of-Scope (Proven NOT IMPLEMENTED in Legacy Code)
- **Nested Organizational Units:** No Departments, Branches, or Sections.
- **Project / Team Groups:** No sub-cohort groups, team-based hackathon rosters, or peer programming pairings.
- **Many-to-Many Group Memberships:** No `cohort_members` or `user_groups` join table.
- **Cohort Modification / Deletion:** No `PATCH`, `PUT`, or `DELETE` endpoints for cohorts.
- **Direct Student Group Management:** No API endpoint to add, move, or remove existing students from a cohort.
- **Cohort-Scoped Assessments:** Assessments lack `cohort_id` or `group_id` columns (globally open across tenants to any user with `read:cohort`).
- **Cohort-Scoped Problems:** Problems belong to tenants or platform global pools; problems cannot be restricted to specific cohorts.

---

## SECTION B: Architecture & Dependency Topology

### B.1 Module Placement & Architectural Context
Cohort management is implemented as a sub-domain inside the `TenancyModule` (`src/tenancy/tenancy.module.ts`), interacting across PostgreSQL database schemas with `IdentityModule` (user membership), `CurriculumModule` (syllabus overlays), and `WebsocketModule` (real-time channels).

```
                      ┌──────────────────────────────────────┐
                      │        TenantsController             │
                      │  (src/tenancy/controllers)           │
                      └──────────────────┬───────────────────┘
                                         │
                                         ▼
                      ┌──────────────────────────────────────┐
                      │          TenantsService              │
                      │   (src/tenancy/services)             │
                      └──────────┬─────────────────┬─────────┘
                                 │                 │
                                 ▼                 ▼
             ┌─────────────────────────┐     ┌─────────────────────────┐
             │    CohortsRepository    │     │  InvitationsRepository  │
             │(src/tenancy/repos)      │     │  (src/tenancy/repos)    │
             └───────────┬─────────────┘     └─────────────┬───────────┘
                         │                                 │
                         ▼                                 ▼
             ┌─────────────────────────┐     ┌─────────────────────────┐
             │  tenancy.cohorts        │     │   tenancy.invitations   │
             │  (PostgreSQL Table)     │     │   (PostgreSQL Table)    │
             └───────────┬─────────────┘     └─────────────┬───────────┘
                         │                                 │
         ┌───────────────┴───────────────┐                 │
         │                               │                 │
         ▼                               ▼                 ▼
┌─────────────────────────┐   ┌─────────────────────┐   ┌─────────────────────────┐
│ curriculum.             │   │ identity.users      │   │ BullMQ Worker           │
│ syllabus_overlays       │   │ (cohort_id column)  │   │ (CsvImportConsumer)     │
│ (Custom syllabus logic) │   └─────────────────────┘   └─────────────────────────┘
```

### B.2 Dependency Inventory

#### Internal Module Dependencies
- **`src/tenancy/tenancy.module.ts`**: Declares and exports `CohortsRepository` and `TenantsService`.
- **`src/shared/prisma/prisma.service.ts`**: Provides multi-schema Prisma client access (`prisma.tenancy`, `prisma.identity`, `prisma.curriculum`).
- **`src/shared/auth/auth.guard.ts`**: Extracts `cohort_id` from JWT access token payload into `req.principal.cohortId`.
- **`src/shared/auth/permission.guard.ts`**: Evaluates permissions `read:cohort` and `manage:cohort`.
- **`src/shared/context/principal-context.ts`**: Defines the `PrincipalContext` interface including `cohortId?: string`.
- **`src/curriculum/controllers/overlays.controller.ts`**: Manages cohort syllabus customizations.
- **`src/curriculum/controllers/me-syllabus.controller.ts`**: Enforces student cohort membership before resolving effective curriculum.
- **`src/websocket/gateways/submissions.gateway.ts`**: Manages real-time socket connections subscribed to cohort channels.

#### External Runtime Dependencies
- **PostgreSQL 15+**: Host database supporting PostgreSQL schemas (`tenancy`, `identity`, `curriculum`).
- **BullMQ / Redis**: Queueing infrastructure for asynchronous roster imports (`csv-import.consumer.ts`).

---

## SECTION C: Data Models & Persistence Mechanics

### C.1 Database Schema Definition (`prisma/tenancy/schema.prisma`)

```prisma
model Cohort {
  id           String    @id @default(dbgenerated("gen_random_uuid()")) @db.Uuid
  tenantId     String    @map("tenant_id") @db.Uuid
  name         String    @db.VarChar(255)
  description  String?   @db.Text
  academicYear String?   @map("academic_year") @db.VarChar(20)
  createdAt    DateTime  @default(now()) @map("created_at") @db.Timestamptz
  updatedAt    DateTime  @updatedAt @map("updated_at") @db.Timestamptz

  tenant Tenant @relation(fields: [tenantId], references: [id])

  @@index([tenantId])
  @@map("cohorts")
  @@schema("tenancy")
}
```

### C.2 Field-by-Field Technical Specifications

| Column Name | Database Type | Nullable | Default | Validation / Constraints | Purpose & Forensic Context |
|---|---|:---:|---|---|---|
| `id` | `UUID` | No | `gen_random_uuid()` | Primary Key | Unique immutable cohort identifier. |
| `tenant_id` | `UUID` | No | — | Foreign Key -> `tenancy.tenants(id)` | Scopes the cohort strictly to a single college/tenant. |
| `name` | `VARCHAR(255)` | No | — | Class-validator: Min 2, Max 80 chars | Display name of the batch (e.g., "CSE 2025"). |
| `description` | `TEXT` | Yes | `NULL` | Optional string | Administrative notes or description of the cohort. |
| `academic_year`| `VARCHAR(20)` | Yes | `NULL` | Optional string | Target academic session (e.g., "2024-25", "2025-26"). |
| `created_at` | `TIMESTAMPTZ` | No | `now()` | Set once on insert | Audit timestamp for cohort creation. |
| `updated_at` | `TIMESTAMPTZ` | No | Auto `@updatedAt` | Automatic database trigger | Timestamp for last record update. |

### C.3 Associated Inter-Schema Relations

#### 1. Identity Schema (`prisma/identity/schema.prisma` - `identity.users`)
```prisma
model User {
  id              String    @id @default(dbgenerated("gen_random_uuid()")) @db.Uuid
  tenantId        String    @map("tenant_id") @db.Uuid
  // ...
  cohortId        String?   @map("cohort_id") @db.Uuid
  // ...
}
```
- **Nature of Relation:** Logical Foreign Key pointing to `tenancy.cohorts(id)`.
- **Relational Integrity:** No Prisma cross-schema `@relation` decorator is declared due to Prisma multi-schema boundary limitations across separate generated clients. Referential integrity is not enforced at the Prisma level.
- **Cardinality:** Exactly 0 or 1 cohort per user.

#### 2. Tenancy Invitations (`prisma/tenancy/schema.prisma` - `tenancy.invitations`)
```prisma
model Invitation {
  id                String    @id @default(dbgenerated("gen_random_uuid()")) @db.Uuid
  tenantId          String    @map("tenant_id") @db.Uuid
  email             String    @db.VarChar(255)
  role              String    @db.VarChar(50)
  cohortId          String?   @map("cohort_id") @db.Uuid
  // ...
}
```
- **Nature of Relation:** Scalar UUID referencing the target cohort that the invitee should join upon registration.

#### 3. Curriculum Syllabus Overlays (`prisma/curriculum/schema.prisma` - `curriculum.syllabus_overlays`)
```prisma
model SyllabusOverlay {
  id         String    @id @default(dbgenerated("gen_random_uuid()")) @db.Uuid
  syllabusId String    @map("syllabus_id") @db.Uuid
  cohortId   String    @map("cohort_id") @db.Uuid
  nodeId     String    @map("node_id") @db.Uuid
  dueDate    DateTime? @map("due_date") @db.Timestamptz
  visible    Boolean   @default(true)
  status     String    @default("draft") @db.VarChar(20)
  createdAt  DateTime  @default(now()) @map("created_at") @db.Timestamptz

  @@unique([syllabusId, cohortId, nodeId], map: "syllabus_overlay_unique")
  @@map("syllabus_overlays")
  @@schema("curriculum")
}
```
- **Nature of Relation:** Composite unique constraint binding syllabus customization operations directly to a `cohortId`.

---

## SECTION D: API & Endpoint Specifications

### D.1 Ingress API Endpoints

#### 1. List Cohorts for a Tenant
- **Method / Path:** `GET /tenants/:id/cohorts`
- **Controller:** `TenantsController.listCohorts` (`src/tenancy/controllers/tenants.controller.ts`, Line 170)
- **RBAC Guard:** `@RequireAuth()`, `@RequirePermission('read:cohort')`
- **Request Parameters:**
  - `id`: Tenant UUID or string literal `'self'`. When `'self'`, resolves to `req.principal!.tenantId`.
- **Response Status:** `200 OK`
- **Response Payload:** Array of Cohort records:
  ```json
  [
    {
      "id": "c1f7b88e-6482-4f3e-8c3d-71bfae8d4a99",
      "tenantId": "e3a8904e-4f11-4475-b6d4-839e564d26f7",
      "name": "CSE Batch of 2025",
      "description": "Final year computer science batch",
      "academicYear": "2024-25",
      "createdAt": "2026-09-01T10:00:00.000Z",
      "updatedAt": "2026-09-01T10:00:00.000Z"
    }
  ]
  ```
- **Error Responses:**
  - `401 Unauthorized`: Missing or invalid Bearer JWT.
  - `403 Forbidden`: Authenticated user lacks `read:cohort` permission.

#### 2. Create a New Cohort
- **Method / Path:** `POST /tenants/:id/cohorts`
- **Controller:** `TenantsController.createCohort` (`src/tenancy/controllers/tenants.controller.ts`, Line 183)
- **RBAC Guard:** `@RequireAuth()`, `@RequirePermission('manage:cohort')`
- **Audit Action:** `@AuditAction('cohort.create')`
- **Request Parameters:**
  - `id`: Tenant UUID or string literal `'self'`.
- **Request Body (`CreateCohortDto` - `src/tenancy/dto/tenant.dto.ts`, Line 91):**
  ```json
  {
    "name": "CSE Batch of 2026",
    "description": "Third year undergraduate cohort",
    "academicYear": "2025-26"
  }
  ```
- **Validation Rules:**
  - `name`: String, minimum 2 characters, maximum 80 characters (DTO) / 255 characters (DB).
  - `description`: Optional string.
  - `academicYear`: Optional string.
- **Response Status:** `201 Created`
- **Response Payload:** Newly created `Cohort` record.
- **Error Responses:**
  - `400 Bad Request`: Validation failure (name missing or shorter than 2 chars).
  - `401 Unauthorized`: Missing Bearer JWT.
  - `403 Forbidden`: Authenticated user lacks `manage:cohort` permission (e.g. students and faculty are rejected).

#### 3. Assign Cohort via User Invitation
- **Method / Path:** `POST /tenants/:id/invitations`
- **Controller:** `TenantsController.inviteUser` (`src/tenancy/controllers/tenants.controller.ts`, Line 203)
- **RBAC Guard:** `@RequireAuth()`, `@RequirePermission('manage:cohort')`
- **Audit Action:** `@AuditAction('invitation.create')`
- **Request Body (`InviteUserDto` - `src/tenancy/dto/tenant.dto.ts`, Line 109):**
  ```json
  {
    "email": "student26@college.edu",
    "role": "student",
    "cohortId": "c1f7b88e-6482-4f3e-8c3d-71bfae8d4a99"
  }
  ```
- **Response Status:** `201 Created`

#### 4. Asynchronous Bulk CSV Roster Import
- **Method / Path:** `POST /tenants/:id/roster/import`
- **Controller:** `TenantsController.importRoster` (`src/tenancy/controllers/tenants.controller.ts`, Line 267)
- **RBAC Guard:** `@RequireAuth()`, `@RequirePermission('manage:cohort')`
- **Audit Action:** `@AuditAction('roster.import')`
- **Response Status:** `202 Accepted`

---

## SECTION E: State Machines & Transition Tables

### E.1 Cohort Entity Lifecycle
Unlike `Tenant` (which defines an elaborate 7-state lifecycle machine: `pending` -> `trial` -> `active` -> `suspended` -> `expired` -> `cancelled` -> `purged`), the `Cohort` entity is **stateless**:
- There is **no `status` column** in `tenancy.cohorts`.
- There is no draft, published, archived, active, or inactive flag.
- Once inserted, a cohort remains permanently queryable via `findByTenant()`.

```
                    ┌─────────────────────────┐
                    │       NON-EXISTENT      │
                    └────────────┬────────────┘
                                 │
                                 │ POST /tenants/:id/cohorts
                                 │ (via CohortsRepository.create)
                                 ▼
                    ┌─────────────────────────┐
                    │         CREATED         │
                    │   (Permanently Active)  │
                    └─────────────────────────┘
```

### E.2 Invitation-to-Cohort Binding Lifecycle
The cohort assignment lifecycle through the invitation subsystem is governed as follows:

| Current State | Event / Trigger | Target State | Cohort Binding Effect | Legacy Implementation Status |
|---|---|---|---|:---:|
| Non-existent | Staff submits `POST /tenants/:id/invitations` with `cohortId` | `pending` | `cohort_id` stored in `tenancy.invitations` | `CONFIRMED` |
| `pending` | Invitee navigates to registration token link | `pending` | Validates expiration (`expiresAt < new Date()`) | `CONFIRMED` |
| `pending` | Invitee calls `acceptInvitation(token)` | `accepted` | `accepted_at` timestamp stamped in DB | `CONFIRMED` |
| `accepted` | User record created / updated | `User.cohortId` | `identity.users.cohort_id` updated to invitation cohort | **`DEFECT — NOT EXECUTED`** |

> [!WARNING]
> **Forensic Reality:** In `TenantsService.acceptInvitation` (`src/tenancy/services/tenants.service.ts`, Line 226), the method updates `tenancy.invitations.acceptedAt = new Date()`, but **does not create or update `identity.users.cohortId`**. The cohort reference is abandoned in the invitations table upon acceptance.

---

## SECTION F: Business Logic, Algorithms & Invariants

### F.1 Multi-Tenant Isolation Invariant
- **Rule:** A cohort must strictly belong to exactly one tenant (`tenant_id`).
- **Enforcement:** `CohortsRepository.findByTenant` enforces `where: { tenantId }`. In `TenantsController`, `id === 'self'` automatically maps to `req.principal!.tenantId`.
- **Cross-Tenant Guard:** Staff from Tenant A cannot list or create cohorts in Tenant B unless they possess the `platform_admin` role (`*`).

### F.2 Cohort Member Cardinality Invariant
- **Rule:** A student or user can belong to **at most ONE cohort** at any point in time.
- **Enforcement:** Model `identity.users` has a scalar `cohortId` column. There is no intermediate join table. A user cannot be simultaneously assigned to both "CSE 2025" and "Honor Track 2025".

### F.3 Member Count Computation Omission
- **Rule:** The frontend TypeScript interface `Cohort` (`frontend/src/types/tenancy.types.ts`, Line 63) defines an optional `memberCount?: number`.
- **Enforcement:** `CohortsRepository.findByTenant` executes a simple `findMany` on `prisma.tenancy.cohort` with no `_count` or join against `identity.users`.
- **Result:** `memberCount` is always `undefined` over the network; the frontend card UI renders a fallback without member statistics.

---

## SECTION G: Authentication, Authorization & RBAC

### G.1 Role-Based Access Control Matrix

| Role | `read:cohort` | `manage:cohort` | View Cohorts | Create Cohort | Manage Overlays | WebSocket Cohort Room |
|---|:---:|:---:|:---:|:---:|:---:|:---:|
| `student` | ✅ | ❌ | Allowed | Blocked (403) | Blocked (403) | Allowed if member |
| `faculty` | ✅ | ❌ | Allowed | Blocked (403) | Blocked (403) | Allowed (override) |
| `coordinator` | ✅ | ✅ | Allowed | Allowed | Allowed | Allowed (override) |
| `college_admin`| ✅ | ✅ | Allowed | Allowed | Allowed | Allowed (override) |
| `content_manager`| ✅ | ❌ | Allowed | Blocked (403) | Blocked (403) | Blocked |
| `platform_moderator`| ✅| ❌ | Allowed | Blocked (403) | Blocked (403) | Blocked |
| `platform_admin` | ✅ (`*`) | ✅ (`*`) | Allowed | Allowed | Allowed | Allowed (override) |

### G.2 Critical Defect: JWT Token Serialization Omission

#### 1. The Token Generation Code
In `AuthService` (`src/identity/services/auth.service.ts`):
```typescript
// Line 253 (issueTokensForUser) & Line 316 (buildLoginResponse)
const accessToken = await this.jwtService.signAsync({
  sub: user.id,
  tenant_id: tenantId,
  person_id: user.personId,
  roles,
  email: user.email,
  // CRITICAL DEFECT: cohort_id IS MISSING FROM SIGNED PAYLOAD!
});
```

#### 2. The Auth Guard Extraction Code
In `AuthGuard` (`src/shared/auth/auth.guard.ts`, Line 55):
```typescript
req.principal = {
  userId: payload.sub,
  tenantId: payload.tenant_id,
  personId: payload.person_id,
  roles: payload.roles,
  email: payload.email,
  cohortId: payload.cohort_id, // ALWAYS RESOLVES TO undefined!
  isImpersonating: payload.imp ?? false,
  impersonatorUserId: payload.act?.user_id,
};
```

#### 3. Impact on Downstream Endpoints
Because `req.principal!.cohortId` is `undefined` for every authenticated user in the system:
1. `GET /me/syllabus/:syllabusId/effective` in `MeSyllabusController` (Line 34):
   ```typescript
   const { userId, tenantId, cohortId } = req.principal!;
   if (!cohortId) throw new ForbiddenException({ code: 'no_cohort_assigned' });
   ```
   **Every student request to fetch an effective cohort syllabus fails with HTTP 403 Forbidden!**
2. In `SubmissionsGateway` (`src/websocket/gateways/submissions.gateway.ts`, Line 38):
   ```typescript
   if (principal.cohortId) {
     await socket.join(`cohort:${principal.cohortId}`);
   }
   ```
   The socket **never joins the cohort channel automatically**.

---

## SECTION H: Real-Time & Event-Driven Mechanics

### H.1 WebSocket Cohort Channels
- **Gateway:** `SubmissionsGateway` (`src/websocket/gateways/submissions.gateway.ts`)
- **Channel Format:** `cohort:<cohortId>`
- **Connection Handshake:**
  - Token claims extracted in `SocketAuthService.validateHandshake` (`src/websocket/auth/socket-auth.service.ts`, Line 39):
    ```typescript
    cohortId: claims.cohort_id ?? null
    ```
  - Upon socket connection:
    ```typescript
    if (principal.cohortId) {
      await socket.join(`cohort:${principal.cohortId}`);
    }
    ```

### H.2 Channel Authorization Rules (`ChannelAuthorizers`)
In `src/websocket/auth/channel-authorizers.ts` (Lines 21–27):
```typescript
case 'cohort':
  return (
    id === principal.cohortId ||
    principal.roles.includes('faculty') ||
    principal.roles.includes('college_admin') ||
    principal.roles.includes('platform_admin')
  );
```
- A user can join a cohort WebSocket channel if:
  1. The target channel UUID matches `principal.cohortId`, OR
  2. The user holds staff privileges: `faculty`, `college_admin`, or `platform_admin`.

---

## SECTION I: Frontend Architecture & User Interface

### I.1 Page & Component Hierarchy
- **Route:** `/tenant/cohorts` (`frontend/src/routes/index.tsx`, Line 724)
- **Primary Page Component:** `CohortsPage` (`frontend/src/pages/tenant/cohorts/CohortsPage.tsx`)
- **Key Child Components:**
  - `PageHeader`: Title, description, and action button.
  - `CreateCohortDialog`: Modal dialog utilizing React Hook Form and Zod validation.
  - `Card`: Grid cards rendering cohort name, academic year, and created date.
  - `Skeleton`: Loading placeholders.

### I.2 State Management & Network Hooks
- **Hook:** `useCohorts(tenantId: string | undefined)` (`frontend/src/hooks/useCohorts.ts`, Line 6)
  - React Query query key: `[...QUERY_KEYS.TENANTS, tenantId, 'cohorts']`
  - Calls: `tenantsService.listCohorts(tenantId)`
  - Cache Stale Time: 2 minutes (`staleTime: 2 * 60 * 1000`).
- **Mutation:** `useCreateCohort(tenantId: string)` (`frontend/src/hooks/useCohorts.ts`, Line 15)
  - Calls: `tenantsService.createCohort(tenantId, dto)`
  - On Success: Automatically invalidates `['tenants', tenantId, 'cohorts']` query cache.

### I.3 Form Validation Schema (`tenant.schemas.ts`)
```typescript
export const createCohortSchema = z.object({
  name: z.string().min(2, 'At least 2 characters').max(80, 'At most 80 characters'),
  academicYear: z.string().max(20).optional(),
})
```

### I.4 UI Limitations
- **No Edit Action:** Cohort cards display no menu, edit modal, or rename trigger.
- **No Delete Action:** There is no delete button or archive confirmation modal.
- **No Student Drill-Down:** Clicking a cohort card does nothing; no `/tenant/cohorts/:id` route exists to inspect enrolled students.

---

## SECTION J: Cross-Module Interactions & External Integrations

### J.1 Curriculum Module (`SyllabusOverlay`)
The most significant consumer of cohorts is the Curriculum module.
1. **Overlay Upsert:** `POST /overlays` (`OverlaysController.upsert` in `src/curriculum/controllers/overlays.controller.ts`, Line 76) binds an overlay entry to a `cohortId`.
2. **Overlay Operations:** Staff can append typed operations to modify the base syllabus for that specific cohort:
   - `hide_node`: Hides a syllabus unit or topic from this cohort.
   - `reorder_children`: Reorders chapters or problems for this cohort.
   - `rename_node`: Replaces a topic title with custom institutional phrasing.
   - `remap_content`: Swaps the problem or reading associated with a node.
   - `add_child_node`: Adds a cohort-specific supplemental problem or module.
3. **Activation & Versioning:** `POST /overlays/:id/activate` marks the overlay `active` for that `(syllabusId, cohortId)` pair while setting existing active overlays to `archived`.
4. **Resolution Engine:** `EffectiveSyllabusResolver.resolve(syllabusId, cohortId)` merges base syllabus nodes with the active overlay operations, caching the final resolved tree in Redis for 30 minutes (`ttl: 1800`).

### J.2 Identity Module (`CsvImportConsumer`)
In `src/worker/consumers/csv-import.consumer.ts`:
- Consumer processes background jobs from queue `csv-import`.
- `importUsers(stream, tenantId, cohortId, importedByUserId)` reads CSV rows and creates users with:
  ```typescript
  cohortId: cohortId ?? null
  ```
- **Controller Gap:** `TenantsController.importRoster` does not accept a `cohortId` body parameter and enqueues `{ tenantId, uploadedBy: req.principal!.userId }` without forwarding a cohort.

---

## SECTION K: Edge Cases, Failure Modes & Resilience

### K.1 Cross-Tenant Cohort Poisoning
- **Scenario:** A malicious tenant admin in Tenant A sends `POST /overlays` specifying a `cohortId` belonging to Tenant B.
- **Legacy Behavior:** `assertOverlayAccess` in `OverlaysController` validates `syllabus.tenantId === req.principal!.tenantId`, but **does not validate that `cohortId` belongs to `req.principal!.tenantId`**! An overlay can theoretically bind a syllabus to a foreign tenant's cohort ID.

### K.2 Orphaned User Cohort IDs
- **Scenario:** A cohort record is deleted directly from the database by a DBA.
- **Legacy Behavior:** Table `identity.users` has no foreign key constraint to `tenancy.cohorts`. Users retain a dangling `cohort_id` pointing to a nonexistent record.

### K.3 Empty Academic Year Handling
- **Scenario:** Cohort is created without `academicYear`.
- **Legacy Behavior:** Backend stores `NULL`. Frontend card safely checks `cohort.academicYear && (...)` and skips the line.

---

## SECTION L: Performance, Caching & Scaling

### L.1 Database Indexing
- Table `tenancy.cohorts` contains index `@@index([tenantId])`.
- Listing cohorts for an institution (`WHERE tenant_id = $1`) uses an indexed B-tree scan.
- Sorting is performed by `orderBy: { createdAt: 'desc' }`. Because `(tenant_id, created_at)` is not a composite index, PostgreSQL performs an in-memory sort on the tenant's cohort subset.

### L.2 Redis Caching in Downstream Syllabus
- Cohort syllabus resolution via `EffectiveSyllabusResolver` caches the computed tree under Redis key:
  `curriculum:effective:${syllabusId}:${cohortId}`
- Cache TTL: 1,800 seconds (30 minutes).
- Cache invalidation occurs on overlay activation (`POST /overlays/:id/activate`).

---

## SECTION M: Security, Threat Model & Compliance

### M.1 RBAC Enforcement Defect in Assessments
- In Module 09, we proved that `assessment.assessments` lacks `cohort_id`.
- Assessments cannot be scoped to a cohort. Any student in the tenant (or across tenants possessing `read:cohort`) can access any assessment by ID.

### M.2 Privilege Escalation Risks
- Staff assigning users via `POST /tenants/:id/invitations` can assign any arbitrary role (`student`, `faculty`, `coordinator`, `college_admin`) along with a `cohortId`.
- No check verifies whether the inviter's role is strictly higher than the role being granted (e.g. a coordinator can invite a college_admin).

---

## SECTION N: Observability, Metrics & Audit Logging

### N.1 Audit Trail Events
Cohort-related controller endpoints are decorated with `@AuditAction`:

| Endpoint | Audit Action Name | Recorded Metadata |
|---|---|---|
| `POST /tenants/:id/cohorts` | `cohort.create` | `tenantId`, `actorId`, `dto` |
| `POST /tenants/:id/invitations` | `invitation.create` | `tenantId`, `actorId`, `email`, `role`, `cohortId` |
| `POST /tenants/:id/roster/import` | `roster.import` | `tenantId`, `actorId` |
| `POST /overlays` | `overlay.upsert` | `syllabusId`, `cohortId`, `nodeId` |
| `POST /overlays/:id/activate` | `overlay.activate` | `overlayId`, `tenantId` |

Audit records are persisted to table `platform.audit_logs`.

---

## SECTION O: Negative Specification (What is NOT Implemented)

To maintain absolute documentation integrity, the following capabilities are explicitly classified as **100% NOT IMPLEMENTED** in the legacy codebase:

```
┌────────────────────────────────────────────────────────────────────────┐
│                   DEZOLVER MODULE 10 NEGATIVE MATRIX                   │
├──────────────────────────────────┬─────────────────────────────────────┤
│ Conceptual Enterprise Feature    │ Legacy Implementation Status        │
├──────────────────────────────────┼─────────────────────────────────────┤
│ 1. Sub-cohort Groups             │ 0 models, 0 tables, 0 endpoints     │
│ 2. College Departments           │ 0 models, 0 tables, 0 endpoints     │
│ 3. Class Sections (e.g. Sec A)   │ 0 models, 0 tables, 0 endpoints     │
│ 4. Many-to-Many Group Membership │ 0 join tables (User.cohortId scalar)│
│ 5. Cohort Update / Rename        │ 0 endpoints (PATCH/PUT not implemented)│
│ 6. Cohort Deletion / Archival    │ 0 endpoints (DELETE not implemented)│
│ 7. Cohort Member List API        │ 0 endpoints (GET /cohorts/:id/members)│
│ 8. Direct Member Reassignment    │ 0 endpoints to move student cohorts │
│ 9. Group / Cohort Assessments    │ 0 columns on assessment.assessments │
│ 10. Cohort Member Count Backend  │ Not computed in CohortsRepository   │
└──────────────────────────────────┴─────────────────────────────────────┘
```

---

## SECTION P: Forensic Code Verification & Traceability Matrix

### P.1 Traceability Verification Table

| Requirement / Component | File Path | Line Reference | Forensic Proof & Behavior |
|---|---|---|---|
| **Cohort Entity Definition** | `prisma/tenancy/schema.prisma` | Lines 35–49 | Model `Cohort` with `id`, `tenantId`, `name`, `description`, `academicYear`. |
| **User Cohort Association** | `prisma/identity/schema.prisma` | Line 41 | `cohortId String? @map("cohort_id") @db.Uuid` on model `User`. |
| **Syllabus Overlay Binding** | `prisma/curriculum/schema.prisma` | Lines 52–68 | `model SyllabusOverlay` with `@@unique([syllabusId, cohortId, nodeId])`. |
| **Cohort List Endpoint** | `src/tenancy/controllers/tenants.controller.ts` | Lines 170–181 | `GET :id/cohorts` guarded by `@RequirePermission('read:cohort')`. |
| **Cohort Create Endpoint** | `src/tenancy/controllers/tenants.controller.ts` | Lines 183–201 | `POST :id/cohorts` guarded by `@RequirePermission('manage:cohort')`. |
| **Cohort Repository Methods** | `src/tenancy/repositories/cohorts.repository.ts` | Lines 8–26 | Methods `findByTenant`, `findById`, `create`. No update/delete. |
| **JWT Cohort Claim Defect** | `src/identity/services/auth.service.ts` | Lines 253, 316 | `signAsync` omits `cohort_id`. |
| **AuthGuard Cohort Extraction** | `src/shared/auth/auth.guard.ts` | Line 55 | `cohortId: payload.cohort_id` (extracts undefined). |
| **Effective Syllabus Check** | `src/curriculum/controllers/me-syllabus.controller.ts` | Line 34 | Throws `403 no_cohort_assigned` if `!cohortId`. |
| **WebSocket Channel Check** | `src/websocket/auth/channel-authorizers.ts` | Lines 21–27 | Authorizes `cohort:<cohortId>` if `id === principal.cohortId` or staff. |
| **Frontend Cohorts Page** | `frontend/src/pages/tenant/cohorts/CohortsPage.tsx` | Lines 1–174 | Renders cohort grid and create dialog. |

---

## SECTION Q: Target State Recommendations & Remediations

To transform Module 10 from a partial batch registry into a production-grade academic group management system, the following remediations must be executed in future refactoring phases:

### Q.1 Immediate Critical Hotfixes
1. **Fix JWT Serialization Defect:** In `AuthService.buildLoginResponse` and `issueTokensForUser`, include `cohort_id: user.cohortId` in the `signAsync` payload so students can access their effective syllabus.
2. **Fix Invitation Acceptance Wiring:** In `TenantsService.acceptInvitation`, when an invitation with `cohortId` is accepted, update `identity.users.cohort_id = invitation.cohortId`.
3. **Fix Bulk Roster Import Ingestion:** In `TenantsController.importRoster`, accept `cohortId` as an optional form parameter and pass it into the `csvQueue.add` payload.

### Q.2 Architectural Enhancements
1. **Implement Cohort CRUD Endpoints:** Add `PATCH /tenants/:id/cohorts/:cohortId` and `DELETE /tenants/:id/cohorts/:cohortId` to allow administrators to edit and retire cohorts.
2. **Implement Member Management Endpoints:**
   - `GET /tenants/:id/cohorts/:cohortId/members`: List students enrolled in the cohort with pagination.
   - `POST /tenants/:id/cohorts/:cohortId/members`: Add existing students to a cohort.
   - `DELETE /tenants/:id/cohorts/:cohortId/members/:userId`: Remove or reassign students.
3. **Transition to Many-to-Many Group Model:** Create table `identity.user_cohorts` with composite key `(user_id, cohort_id)` to allow learners to belong to multiple specialized tracks, labs, or elective cohorts.
4. **Scope Assessments to Cohorts:** Add `cohort_id` or join table `assessment_cohorts` to table `assessment.assessments` to prevent exam leakage across unintended student batches.

