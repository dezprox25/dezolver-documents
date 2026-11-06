# Module 14 — Support Tickets
## Comprehensive Functional Specification & Forensic Negative Reconstruction

**Standard:** [`FUNCTIONAL-SPECIFICATION-STANDARD.md`](../../FUNCTIONAL-SPECIFICATION-STANDARD.md) (`DEZ-STD-FS-001`)  
**Domain:** Helpdesk, Issue Ticketing, User Inquiry Resolution, and Customer Support  
**Legacy Package:** `src/` (Non-existent), `prisma/` (Non-existent), `frontend/src/` (Non-existent)  
**Reconstruction State:** `COMPLETED & FORENSICALLY VERIFIED (100% NOT IMPLEMENTED)`  
**Investigation Mode:** Strict Forensic Codebase Extraction (Zero Hallucination, Zero Assumptions)  

---

## SECTION A: Module Overview & Scope Boundaries

### A.1 Functional Definition
Module 14 governs user support ticketing, student inquiry tracking, technical bug reporting, billing dispute management, service level agreements (SLAs), conversation threads between users and helpdesk staff, and ticket escalation workflows within the Dezolver platform.

### A.2 Implementation Reality: 100% NOT IMPLEMENTED
A rigorous and systematic forensic examination of the entire legacy codebase establishes an indisputable fact:

> [!IMPORTANT]
> **Authoritative Negative Finding:**  
> The Support Tickets module is **100% NOT IMPLEMENTED** in the legacy codebase. There are **zero database models, zero backend controllers or services, zero API endpoints, zero RBAC permissions, and zero frontend pages or components** dedicated to user support, helpdesk management, or ticketing.

The sole occurrence of the word "ticket" in the entire codebase is an informational remark inside an impersonation justification warning dialog in `frontend/src/pages/platform/persons/PersonDetailPage.tsx` (Line 99):
```tsx
<p className="text-xs text-amber-700 dark:text-amber-400">
  This action is audited. A justification is required and a case ID is recommended for support tickets.
</p>
```
No underlying ticketing software, API client, or database table exists to back this recommendation.

### A.3 Scope Boundaries

#### In-Scope (Forensic Negative Verification)
- Verification of zero models across all 11 PostgreSQL Prisma schemas.
- Verification of zero files, routes, or services across all NestJS backend modules in `src/`.
- Verification of zero permissions across RBAC matrices.
- Verification of zero routes, views, or stores across the React frontend.
- Authoritative architectural specification for the target state implementation.

#### Out-of-Scope (Proven Entirely Non-Existent in Legacy Code)
- Ticket creation, assignment, prioritization, and resolution.
- Multi-tiered support (L1/L2/L3 support agents).
- Discussion threads, private staff notes, and file attachments.
- Service Level Agreement (SLA) timers and automated breach notifications.
- Canned responses, knowledge base integration, and customer satisfaction (CSAT) surveys.

---

## SECTION B: Architecture & Dependency Topology

### B.1 Legacy Reality: Complete Absence of Ticketing Module
In `src/`, there is no `tickets` or `support` module:
```
src/
├── assessment/     (Code submissions, Judge0, grading)
├── billing/        (Subscriptions, payments, invoices, orders)
├── content/        (Media, problems, search)
├── credentials/    (Certificates, templates)
├── curriculum/     (Syllabus, nodes, rooms, overlays)
├── events/         (Contests, registrations, leaderboards)
├── identity/       (Users, auth, MFA, tokens, GDPR)
├── paths/          (Curated learning paths)
├── platform/       (Audit logs, feature flags, health)
├── shared/         (Prisma, Redis, BullMQ, Outbox, Auth)
├── tenancy/        (Tenants, cohorts, invitations)
├── websocket/      (Socket.io gateways and broadcasters)
└── worker/         (BullMQ queue processors)
```
No `tickets.module.ts` or `support.module.ts` exists.

### B.2 Target Architecture Topology (Proposed Blueprint)

```
┌────────────────────────────────────────────────────────┐
│                    STUDENT / USER                      │
│  - Creates ticket (bug, billing, academic, general)    │
│  - Attaches screenshot / log                           │
│  - Replies to staff responses                          │
└───────────┬────────────────────────────────┬───────────┘
            │                                │
            ▼                                ▼
┌────────────────────────┐       ┌────────────────────────┐
│   TicketsController    │       │     TicketsGateway     │
│  (src/support/ctrls)   │       │  (WebSocket Live Chat) │
└───────────┬────────────┘       └───────────┬────────────┘
            │                                │
            ▼                                ▼
┌────────────────────────┐       ┌────────────────────────┐
│     TicketsService     │◄──────┤   RealtimeBroadcaster  │
└───────────┬────────────┘       └────────────────────────┘
            │
            ├────────────────────────────────┬───────────────────────────────┐
            ▼                                ▼                               ▼
┌───────────────────────┐       ┌─────────────────────────┐     ┌────────────────────────┐
│  support.tickets      │       │  platform.outbox_events │     │   BullMQ Queue 'email' │
│  support.messages     │       └────────────┬────────────┘     │   (Staff notification  │
│  support.attachments  │                    │                  │    & user updates)     │
└───────────────────────┘                    ▼                  └────────────────────────┘
                                ┌─────────────────────────┐
                                │     TicketSlaWatcher    │
                                │   (Escalation Cron)     │
                                └─────────────────────────┘
```

---

## SECTION C: Data Models & Persistence Mechanics

### C.1 Negative Verification Across All 11 Prisma Schemas

| Schema Path | Total Models | Ticketing Entities Found | Forensic Status |
|---|:---:|:---:|:---:|
| `prisma/analytics/schema.prisma` | 0 (Draft) | None | `NOT IMPLEMENTED` |
| `prisma/assessment/schema.prisma`| 3 | None | `NOT IMPLEMENTED` |
| `prisma/billing/schema.prisma` | 7 | None | `NOT IMPLEMENTED` |
| `prisma/credentials/schema.prisma`| 2 | None | `NOT IMPLEMENTED` |
| `prisma/curriculum/schema.prisma`| 4 | None | `NOT IMPLEMENTED` |
| `prisma/events/schema.prisma` | 3 | None | `NOT IMPLEMENTED` |
| `prisma/identity/schema.prisma` | 7 | None | `NOT IMPLEMENTED` |
| `prisma/paths/schema.prisma` | 4 | None | `NOT IMPLEMENTED` |
| `prisma/platform/schema.prisma` | 5 | None | `NOT IMPLEMENTED` |
| `prisma/problems/schema.prisma` | 4 | None | `NOT IMPLEMENTED` |
| `prisma/tenancy/schema.prisma` | 3 | None | `NOT IMPLEMENTED` |
| **Total Platform Schemas** | **42 Models** | **0 Models** | **100% NOT IMPLEMENTED** |

### C.2 Target Data Model Specification (`prisma/support/schema.prisma`)

When implementing Module 14, the following database schema must be introduced:

```prisma
datasource db {
  provider = "postgresql"
  url       = env("DATABASE_URL")
  schemas  = ["support"]
}

model Ticket {
  id              String    @id @default(dbgenerated("gen_random_uuid()")) @db.Uuid
  ticketNumber    Int       @default(autoincrement()) @map("ticket_number")
  tenantId        String    @map("tenant_id") @db.Uuid
  creatorUserId   String    @map("creator_user_id") @db.Uuid
  assignedUserId  String?   @map("assigned_user_id") @db.Uuid
  category        String    @db.VarChar(50)
  priority        String    @default("medium") @db.VarChar(20)
  status          String    @default("open") @db.VarChar(30)
  subject         String    @db.VarChar(255)
  slaDueAt        DateTime? @map("sla_due_at") @db.Timestamptz
  firstResponseAt DateTime? @map("first_response_at") @db.Timestamptz
  resolvedAt      DateTime? @map("resolved_at") @db.Timestamptz
  closedAt        DateTime? @map("closed_at") @db.Timestamptz
  createdAt       DateTime  @default(now()) @map("created_at") @db.Timestamptz
  updatedAt       DateTime  @updatedAt @map("updated_at") @db.Timestamptz

  messages    TicketMessage[]
  attachments TicketAttachment[]

  @@index([tenantId, status])
  @@index([creatorUserId])
  @@index([assignedUserId])
  @@map("tickets")
  @@schema("support")
}

model TicketMessage {
  id        String   @id @default(dbgenerated("gen_random_uuid()")) @db.Uuid
  ticketId  String   @map("ticket_id") @db.Uuid
  authorId  String   @map("author_id") @db.Uuid
  isStaff   Boolean  @default(false) @map("is_staff")
  isPrivate Boolean  @default(false) @map("is_private")
  body      String   @db.Text
  createdAt DateTime @default(now()) @map("created_at") @db.Timestamptz

  ticket Ticket @relation(fields: [ticketId], references: [id], onDelete: Cascade)

  @@index([ticketId, createdAt])
  @@map("ticket_messages")
  @@schema("support")
}

model TicketAttachment {
  id        String   @id @default(dbgenerated("gen_random_uuid()")) @db.Uuid
  ticketId  String   @map("ticket_id") @db.Uuid
  messageId String?  @map("message_id") @db.Uuid
  fileName  String   @map("file_name") @db.VarChar(255)
  fileSize  Int      @map("file_size")
  mimeType  String   @map("mime_type") @db.VarChar(100)
  s3Key     String   @map("s3_key") @db.VarChar(512)
  createdAt DateTime @default(now()) @map("created_at") @db.Timestamptz

  ticket Ticket @relation(fields: [ticketId], references: [id], onDelete: Cascade)

  @@map("ticket_attachments")
  @@schema("support")
}
```

---

## SECTION D: API & Endpoint Specifications

### D.1 Negative Verification: Zero API Routes
The following standard support endpoints were searched for and verified as **completely absent**:
- `GET /tickets` $\to$ **404 Not Found**
- `POST /tickets` $\to$ **404 Not Found**
- `GET /tickets/:id` $\to$ **404 Not Found**
- `POST /tickets/:id/messages` $\to$ **404 Not Found**
- `PATCH /tickets/:id` $\to$ **404 Not Found**
- `POST /tickets/:id/assign` $\to$ **404 Not Found**
- `POST /tickets/:id/resolve` $\to$ **404 Not Found**
- `POST /tickets/:id/close` $\to$ **404 Not Found**

### D.2 Target REST API Specification (Blueprint)

| Method | Proposed Route Path | Permitted Roles | Description |
|---|---|---|---|
| `GET` | `/me/tickets` | `student`, `faculty` | Lists tickets opened by the calling user. |
| `POST` | `/me/tickets` | `student`, `faculty` | Opens a new support ticket with subject, category, body. |
| `GET` | `/me/tickets/:id` | `student`, `faculty` | Retrieves ticket thread and messages for caller's ticket. |
| `POST` | `/me/tickets/:id/messages`| `student`, `faculty` | Appends a reply message to the ticket. |
| `GET` | `/admin/tickets` | `support_agent`, `platform_admin` | Helpdesk agent queue with filters (status, priority, SLA). |
| `GET` | `/admin/tickets/:id` | `support_agent`, `platform_admin` | Complete ticket history including private staff notes. |
| `POST` | `/admin/tickets/:id/messages`| `support_agent`, `platform_admin` | Staff response or private internal note (`isPrivate: true`). |
| `PATCH`| `/admin/tickets/:id/assign` | `support_agent`, `platform_admin` | Assigns ticket to a specific support agent. |
| `PATCH`| `/admin/tickets/:id/status` | `support_agent`, `platform_admin` | Transitions status (`in_progress`, `resolved`, `closed`). |

---

## SECTION E: State Machines & Transition Tables

### E.1 Target Ticket Lifecycle State Machine (Blueprint)

```
                       ┌────────────────────────┐
                       │          OPEN          │
                       └───────────┬────────────┘
                                   │ Staff Views / Takes Ownership
                                   ▼
                       ┌────────────────────────┐
                       │      IN_PROGRESS       │
                       └─────┬────────────┬─────┘
                             │            │
            Staff Requests   │            │ Staff Resolves
            User Info        │            │ Issue
                             ▼            ▼
        ┌────────────────────────┐    ┌────────────────────────┐
        │    WAITING_FOR_USER    │    │        RESOLVED        │
        └────────────┬───────────┘    └───────────┬────────────┘
                     │                            │
                     │ User Replies               │ User / Auto-close (7d)
                     ▼                            ▼
        ┌────────────────────────┐    ┌────────────────────────┐
        │      IN_PROGRESS       │    │         CLOSED         │
        └────────────────────────┘    └───────────┬────────────┘
                                                  │
                                                  │ User Reopens (< 14d)
                                                  ▼
                                      ┌────────────────────────┐
                                      │        REOPENED        │
                                      └────────────────────────┘
```

### E.2 State Transition Rules
- `open` $\to$ `in_progress`: Triggered when an agent claims the ticket or posts a response.
- `in_progress` $\to$ `waiting_for_user`: Triggered when an agent requests logs, code snippets, or clarification.
- `waiting_for_user` $\to$ `in_progress`: Triggered automatically when the user posts a reply.
- `in_progress` $\to$ `resolved`: Triggered when an agent provides the solution.
- `resolved` $\to$ `closed`: Triggered by user confirmation, or automatically after 7 calendar days of inactivity.
- `closed` $\to$ `reopened`: Allowed within 14 calendar days of closure if the problem reoccurs.

---

## SECTION F: Business Logic, Algorithms & Invariants

### F.1 Target Service Level Agreement (SLA) Engine
In the target implementation, ticket priority dictates automated SLA response and resolution targets:

| Priority Level | First Response Target | Resolution Target | Target Categories |
|---|:---:|:---:|---|
| `urgent` | 1 hour | 4 hours | Platform outages, payment errors during live contest |
| `high` | 4 hours | 12 hours | Assessment grading failures, login lockouts |
| `medium` | 8 hours | 24 hours | Course content ambiguities, certificate discrepancies |
| `low` | 24 hours | 72 hours | General feedback, feature requests |

### F.2 Automated Escalation Invariants
- If `firstResponseAt` is null and `now() >= slaDueAt`, the system must mark the ticket `sla_breached: true` and escalate notification to `support_manager`.

---

## SECTION G: Authentication, Authorization & RBAC

### G.1 Required Support Roles & Permissions
Because no support roles exist in `permission.guard.ts`, the target implementation requires introducing:

| Role Name | Required Permissions | Scope of Authority |
|---|---|---|
| `student` | `create:ticket`, `read:ticket:own` | Create personal tickets and view own history. |
| `faculty` | `create:ticket`, `read:ticket:own` | Create academic/course tickets and view own history. |
| `college_admin` | `read:ticket:tenant`, `manage:ticket:tenant` | View and manage tickets across their institution. |
| `support_agent` | `read:ticket:all`, `manage:ticket:all`, `reply:ticket` | L1/L2 platform support staff resolving tickets. |
| `support_manager`| `manage:support:all`, `reassign:ticket`, `manage:sla` | Helpdesk administration, SLA policy overrides, reassignments. |

---

## SECTION H: Real-Time & Event-Driven Mechanics

### H.1 Target WebSocket Support Channels
- **Room:** `ticket:${ticketId}`
- **Events:**
  - `ticket:message`: Real-time bidirectional chat between user and staff agent.
  - `ticket:typing`: Agent/user typing indicator.
  - `ticket:status`: Immediate notification when status shifts to `resolved` or `closed`.

### H.2 Target Outbox Event Specifications
- `TicketCreated`: `{ ticketId, tenantId, userId, category, priority }`
- `TicketReplied`: `{ ticketId, messageId, authorId, isStaff }`
- `TicketSlaBreached`: `{ ticketId, priority, overdueMinutes }`
- `TicketResolved`: `{ ticketId, resolvedByUserId, durationMinutes }`

---

## SECTION I: Frontend Architecture & User Interface

### I.1 Target Frontend Page Catalog (Blueprint)

| Target Route Path | Proposed Component Name | Target File Path | Primary Functionality |
|---|---|---|---|
| `/support` | `SupportCenterPage` | `frontend/src/pages/support/SupportCenterPage.tsx` | User overview with search, FAQs, and "Open Ticket" button. |
| `/support/tickets` | `MyTicketsPage` | `frontend/src/pages/support/MyTicketsPage.tsx` | Student list of tickets with status badges and filter controls. |
| `/support/tickets/new`| `NewTicketPage` | `frontend/src/pages/support/NewTicketPage.tsx` | Ticket authoring form with category selection and dropzone. |
| `/support/tickets/:id`| `TicketThreadPage` | `frontend/src/pages/support/TicketThreadPage.tsx` | Full conversation thread, message composer, and resolution button. |
| `/platform/support` | `AdminHelpdeskPage` | `frontend/src/pages/platform/support/AdminHelpdeskPage.tsx` | Agent triage board, SLA countdowns, priority sorting, and quick macros. |

---

## SECTION J: Cross-Module Interactions & External Integrations

```
┌────────────────────────────────────────────────────────┐
│               TARGET MODULE 14 TICKETS                 │
└────────┬───────────────────────┬───────────────────────┘
         │                       │
         ▼                       ▼
┌──────────────────┐    ┌──────────────────┐
│ MODULE 13 NOTIF  │    │ MODULE 3 BILLING │
│ SES email update │    │ References to    │
│ when agent posts │    │ invoiceId /      │
│ reply to user.   │    │ transactionId.   │
└──────────────────┘    └──────────────────┘
```

---

## SECTION K: Edge Cases, Failure Modes & Resilience

1. **Malicious File Attachments:** Target system must enforce strict MIME-type allowlists (PDF, PNG, JPG, TXT) and scan uploads via AWS S3 Object Lambda or ClamAV.
2. **Ticket Flooding / Spam Denial of Service:** Rate limiting must restrict users to opening at most 5 tickets per hour and at most 10 active open tickets concurrently.
3. **Ghost Ticket Auto-Resolution:** Tickets in `waiting_for_user` for more than 7 days must transition automatically to `resolved`, with a warning email dispatched at day 5.

---

## SECTION L: Performance, Caching & Scaling

- **Read Throughput:** Ticket threads must be indexed by `@@index([ticketId, createdAt])` to ensure fast cursor-based message loading.
- **Agent Dashboard Optimization:** Active ticket counters (Open, Unassigned, Breached) must be cached in Redis with a 30-second TTL to avoid expensive database table scans.

---

## SECTION M: Security, Threat Model & Compliance

- **Multi-Tenant Data Segregation:** Tickets created by users in College A must be strictly inaccessible to staff in College B. Platform support agents must hold global permissions to cross tenant boundaries.
- **PII Data Protection & GDPR:** Ticket bodies containing personal phone numbers or payment information must be redactable by authorized compliance officers.

---

## SECTION N: Observability, Metrics & Audit Logging

The target implementation must record the following audit actions in `platform.audit_logs`:
- `ticket.create`: Initial ticket submission.
- `ticket.reply`: Staff message addition.
- `ticket.reassign`: Staff reassignment.
- `ticket.resolve`: Issue marked resolved.
- `ticket.close`: Issue closed.

---

## SECTION O: Negative Specification (What is NOT Implemented)

```
┌────────────────────────────────────────────────────────────────────────┐
│                   DEZOLVER MODULE 14 NEGATIVE MATRIX                   │
├──────────────────────────────────┬─────────────────────────────────────┤
│ Architectural Area               │ Forensic Legacy Status              │
├──────────────────────────────────┼─────────────────────────────────────┤
│ 1. Database Ticket Models        │ 0 models across 11 Prisma schemas   │
│ 2. Backend Ticket Controllers    │ 0 files in src/                     │
│ 3. Backend Ticket Services       │ 0 files in src/                     │
│ 4. Support RBAC Permissions      │ 0 permissions in permission.guard   │
│ 5. Frontend Support Routes       │ 0 routes in routes/index.tsx        │
│ 6. Frontend Support Components   │ 0 components in frontend/src/       │
│ 7. Support API Client            │ 0 clients in services/api/          │
│ 8. Ticket SLA Watcher Cron       │ 0 scheduled jobs in backend         │
│ 9. Support WebSocket Gateways    │ 0 gateways or channels              │
└──────────────────────────────────┴─────────────────────────────────────┘
```

---

## SECTION P: Forensic Code Verification & Traceability Matrix

| Target Verification Step | Command / Query Executed | Result | Forensic Proof |
|---|---|---|---|
| **Prisma Schemas Scan** | `Get-ChildItem -Path "prisma\*\schema.prisma" \| Select-String -Pattern "ticket\|Ticket"` | 0 matches | No models in database. |
| **Backend Code Scan** | `Get-ChildItem -Path "src" -Recurse -Filter "*.ts" \| Select-String -Pattern "ticket"` | 0 matches | No backend controllers, services, or entities. |
| **Frontend Code Scan** | `Get-ChildItem -Path "frontend\src" -Recurse -Filter "*.ts*" \| Select-String -Pattern "ticket"` | 1 match | Line 99 in `PersonDetailPage.tsx` (Advisory string only). |
| **Route Registry Scan** | `Select-String -Path "frontend\src\routes\index.tsx" -Pattern "support\|ticket"` | 0 matches | No routes registered in React Router. |
| **Sidebar Navigation** | `Select-String -Path "frontend\src\components\layouts\Sidebar.tsx" -Pattern "support\|ticket"` | 0 matches | No support navigation item in UI. |
| **RBAC Permissions** | `Select-String -Path "src\shared\auth\permission.guard.ts" -Pattern "ticket"` | 0 matches | No support permissions in NestJS guard. |

---

## SECTION Q: Target State Recommendations & Remediations

To construct Module 14 in future development phases, follow this sequential implementation roadmap:

### Q.1 Phase 1: Database & Backend Foundations
1. **Create `prisma/support/schema.prisma`:** Declare models `Ticket`, `TicketMessage`, and `TicketAttachment` adhering to Section C.2. Run Prisma migration.
2. **Implement `TicketsModule` in `src/support/`:**
   - Author `tickets.repository.ts` with tenant-scoped queries.
   - Author `tickets.service.ts` managing ticket lifecycles and validation.
   - Author `tickets.controller.ts` providing the REST endpoints specified in Section D.2.
3. **Register RBAC Permissions:** Add `create:ticket`, `read:ticket:own`, `read:ticket:all`, and `manage:ticket:all` to `src/shared/auth/permission.guard.ts`.

### Q.2 Phase 2: Asynchronous Workers & Real-Time Engine
1. **Implement `TicketSlaWatcherService`:** 5-minute scheduled cron checking `now() >= slaDueAt` and marking overdue tickets.
2. **Connect BullMQ Email Notifications:** Dispatch email notifications to users on agent replies via `emailQueue` (`templateId: 'ticket-reply'`).
3. **Implement WebSocket Support Gateway:** Author `TicketsGateway` to stream live message additions to active ticket threads.

### Q.3 Phase 3: Frontend Implementation
1. **Create API Client:** Author `frontend/src/services/api/ticket.service.ts`.
2. **Construct User Pages:** Build `SupportCenterPage.tsx`, `MyTicketsPage.tsx`, `NewTicketPage.tsx`, and `TicketThreadPage.tsx`.
3. **Construct Admin Helpdesk View:** Build `AdminHelpdeskPage.tsx` with triage boards, SLA indicators, and internal note toggles.
4. **Mount Routes & Sidebar Link:** Add `/support` and `/support/tickets` to `frontend/src/routes/index.tsx` and register `<LifeBuoy />` in `Sidebar.tsx`.

