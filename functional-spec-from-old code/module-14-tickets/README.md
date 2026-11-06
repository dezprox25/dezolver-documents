# Module 14 — Support Tickets

## 1. Module Executive Summary

Module 14 governs the user helpdesk, issue ticketing, student grievance logging, inquiry management, and customer support infrastructure of the Dezolver platform.

A forensic audit of the entire legacy codebase (`prisma/`, `src/`, and `frontend/src/`) delivers an unambiguous, conclusive discovery:

> [!IMPORTANT]
> **Forensic Verdict: 100% NOT IMPLEMENTED**  
> There is **zero implementation** of a support ticketing, helpdesk, or issue resolution subsystem in the legacy codebase. No database models exist in PostgreSQL, no controllers or services exist in NestJS, no permissions exist in RBAC guards, and no pages or routes exist in the frontend application.

### Key Forensic Findings:
1. **Zero Database Models:** Across all 11 PostgreSQL Prisma schemas in `prisma/`, there are 0 models for `Ticket`, `SupportTicket`, `TicketMessage`, `TicketCategory`, `SLA`, or `AgentAssignment`.
2. **Zero Backend Code:** Across all 14 NestJS domain modules in `src/`, there are 0 controllers, services, repositories, DTOs, BullMQ workers, or Outbox events related to tickets or helpdesk support.
3. **Zero RBAC Permissions:** Neither `src/shared/auth/permission.guard.ts` nor `frontend/src/lib/permissions/roles.ts` contains permissions such as `create:ticket`, `read:ticket`, or `manage:ticket`.
4. **Zero Frontend Views:** The frontend router (`frontend/src/routes/index.tsx`) contains 0 routes for support (`/support`, `/tickets`, or `/helpdesk`). The sidebar navigation (`Sidebar.tsx`) contains 0 links to support.
5. **Solitary Code Mention:** The sole occurrence of the word "ticket" in the entire codebase is an advisory string in `PersonDetailPage.tsx` (Line 99): *"This action is audited. A justification is required and a case ID is recommended for support tickets."*

---

## 2. Directory Layout & Document Navigation

```
docs/functional-spec/module-14-tickets/
├── README.md               # Executive summary, negative verification, and navigational guide (this file)
└── SPECIFICATION.md        # Exhaustive 17-Section (A–Q) Authoritative Negative Specification & Target Blueprint
```

- **Detailed Functional Specification:** [`SPECIFICATION.md`](./SPECIFICATION.md)
- **Authoring Standard:** [`FUNCTIONAL-SPECIFICATION-STANDARD.md`](../../FUNCTIONAL-SPECIFICATION-STANDARD.md)
- **Master Index & Registry:** [`INDEX.md`](../INDEX.md)

---

## 3. Negative Verification Audit Matrix

| System Layer | Searched Patterns | Target Path | Result | Forensic Status |
|---|---|---|:---:|:---:|
| **PostgreSQL Schemas** | `Ticket`, `Support`, `Helpdesk`, `Issue` | `prisma/*/schema.prisma` (11 schemas) | **0 models found** | `NOT IMPLEMENTED` |
| **Backend Controllers** | `tickets.controller`, `support.controller` | `src/**/controllers/` | **0 files found** | `NOT IMPLEMENTED` |
| **Backend Services** | `tickets.service`, `support.service` | `src/**/services/` | **0 files found** | `NOT IMPLEMENTED` |
| **Queue Workers** | `@Processor('ticket')`, `@Processor('support')`| `src/worker/` | **0 processors found** | `NOT IMPLEMENTED` |
| **RBAC Matrix** | `*:ticket`, `*:support` | `src/shared/auth/permission.guard.ts` | **0 permissions found**| `NOT IMPLEMENTED` |
| **Frontend Router** | `/tickets`, `/support`, `/help` | `frontend/src/routes/index.tsx` | **0 routes found** | `NOT IMPLEMENTED` |
| **Frontend UI Pages** | `Ticket*.tsx`, `Support*.tsx` | `frontend/src/pages/` | **0 pages found** | `NOT IMPLEMENTED` |
| **API Client** | `ticket.service.ts`, `support.service.ts` | `frontend/src/services/api/` | **0 services found** | `NOT IMPLEMENTED` |

---

## 4. Target Architecture Blueprint

Because Module 14 is completely absent from the legacy codebase, [`SPECIFICATION.md`](./SPECIFICATION.md) provides both the forensic negative proof and an authoritative Target Architecture Blueprint defining the required production entities, REST APIs, SLA state machines, email integrations, and role-based permissions needed to build the subsystem in subsequent refactoring phases.

