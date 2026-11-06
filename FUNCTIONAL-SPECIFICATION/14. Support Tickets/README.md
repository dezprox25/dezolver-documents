# DEZOLVER — Module 14: Support Tickets

**Document ID:** TICKET-FS-001  
**Version:** 0.1  
**Status:** Detailed draft — source-aligned; ticket workflow and lifecycle decisions remain open  
**Governing standard:** [Functional Specification Standard v1.0](../FUNCTIONAL-SPECIFICATION-STANDARD.md)  
**Repository inclusion:** Draft only; not approval of target behavior.

## 1. Purpose and module outcome

Module 14 documents the Support Tickets pages, ticket/comment data model, role permissions, and the ticket lifecycle vocabulary visible in the supplied sources.

The source documents establish that DEZOLVER has a support-ticket capability. They do **not** describe the end-to-end ticket workflow, comment visibility, assignment rules, status transition rules, service-level expectations, or notifications. This specification preserves those gaps and identifies candidate functions without presenting their detailed behavior as approved product requirements.

## 2. Scope

### In scope
- Student/Manager Help & Support and My Tickets page mappings.
- Finance Team Support Tickets page mapping.
- HOD Ticket Verification and Help & Support page mappings.
- Ticket submission and requester-owned ticket access.
- Staff ticket access within RBAC scope.
- Ticket comments and isInternal field.
- Assignment, status, priority, resolution, escalation, reopen, and verification as schema/page/RBAC-related function areas.
- Source-based data reference, validations, QA cases, dependencies, and decision log.

### Out of scope / not source-defined
- Exact page layout, navigation routes, ticket list columns, filters, sorting, pagination, search, and UI copy.
- SLA rules, working hours, response/resolution targets, queue ordering, and automatic escalation.
- Email/SMS/push notifications and notification trigger/recipient policy.
- Attachments, screenshots, file uploads, rich text, markdown, and message formatting.
- Ticket merge, duplicate detection, spam handling, rating/satisfaction surveys, knowledge base, chatbot, and analytics.
- Assignment algorithms, agent availability, round-robin, workload balancing, and automatic routing.
- Status transition policy, reopening limits, closure timers, and deletion/retention policy.
- Any behavior not explicitly present in the source docs.

## 3. Source documents and evidence

| Source | Evidence used | Limitations |
|---|---|---|
| [Business Flow v2](../../docs/source-text/Business%20Flow%20v2.md) | Reviewed for ticket/support workflow | No support-ticket workflow is shown |
| [Business Flow original](../../docs/source-text/Business%20Flow.md) | Reviewed for ticket/support workflow | No support-ticket workflow is shown |
| [Page–DB Mapping v2](../../docs/source-text/Page%20-%20DB%20Mapping%20v2.md) | Role-based Support Tickets, Ticket Verification, Help & Support, My Tickets pages and entities | Page/entity mapping only; no ticket actions or lifecycle |
| [Page–DB Mapping original](../../docs/source-text/Page%20-%20DB%20Mapping.md) | Corresponding role-based ticket/support pages | Mapping does not define detailed interactions |
| [RBAC](../../docs/source-text/RBAC.md) | Support/Tickets role permissions, role scopes, API authorization rule | Does not define TicketComment-specific permission or workflow states |
| [Schema](../../docs/source-text/scheema.md) | SupportTicket, TicketComment, TicketStatus, TicketPriority, TicketCategory, relationships | Schema does not establish lifecycle transition rules or legacy implementation |
| [Functional Specification Standard](../FUNCTIONAL-SPECIFICATION-STANDARD.md) | Function-level detail, evidence labels, decision log, acceptance criteria, QA requirements | Authoring standard, not product behavior |

### Evidence labels
- **Source-confirmed:** explicitly present in supplied source.
- **Derived:** logical consequence of source facts, not direct UI/workflow statement.
- **Recommendation:** proposal only; not approved.
- **TBD / Decision Required:** missing, ambiguous, or conflicting source behavior.

## 4. Terminology

| Term | Definition / evidence |
|---|---|
| Support ticket | Schema record in SupportTicket; exact business workflow is TBD |
| Requester | requesterId field and schema relationship user_creates_support_ticket identify the ticket-creating user |
| Assignee | assignedToId field; assignment meaning/rules are not defined |
| Ticket comment | TicketComment record related to a ticket and authored by a User |
| Internal comment | Comment with isInternal boolean; intended visibility and role semantics are TBD |
| Ticket category | Enum: TECHNICAL, BILLING, ACADEMIC, ACCOUNT |
| Ticket priority | Enum: LOW, MEDIUM, HIGH, URGENT |
| Ticket status | Enum: SUBMITTED, RESOLVED, ESCALATED, MORE_INFO_REQUESTED, CLOSED, REOPENED |
| Ticket verification | HOD page name in Page–DB Mapping; exact verification purpose/actions are not defined |
| Resolution / escalation notes | Schema text fields; requiredness, visibility, and editing rules are TBD |

## 5. Actors, permissions, and scope

### 5.1 RBAC source matrix

| Role | Support / Ticket resource permission | Source scope |
|---|---|---|
| SUPER_ADMIN | V/C/U/D | Global |
| FINANCE_TEAM | V/C/U* | Assigned College |
| HOD | V/C/U* | Assigned College + Department |
| MANAGER | V/C/U* | Assigned College + Department + assigned Section / permitted Group |
| STUDENT | C / V OWN | Own ticket records / permitted academic data |

* means the action must additionally pass authorized data-scope checks.

**Important:** RBAC names the Support/Ticket resource. It does not specify whether CRUD permissions apply identically to SupportTicket and TicketComment, nor which staff actions (assign, escalate, resolve, verify, or request more information) are permitted. Those details remain open.

### 5.2 Security and scope
- **Source-confirmed:** SUPER_ADMIN has global scope; FINANCE_TEAM is limited to assigned College; HOD to assigned College + Department; MANAGER to assigned College + Department + permitted Section/Group; STUDENT to own records and permitted academic resources.
- **Source-confirmed:** RBAC requires authentication, role, page/module permission, data scope, and ownership checks where applicable. Backend must independently enforce authorization.
- **Derived:** Student access to ticket records must be constrained to the authenticated requester’s own tickets, consistent with V OWN.
- **Derived:** collegeId, requesterId, and assignedToId received from a client must not be trusted as proof of authorization.
- **TBD:** Whether a Manager can see every ticket in their section, tickets assigned to them, tickets they created, or a combination.
- **TBD:** Whether Finance/HOD staff see all tickets in their scope, only assigned tickets, or specific categories.
- **TBD:** Whether Ticket Verification is a HOD-only permission, a page-level workflow, or a distinct operation.

### 5.3 Page / entry point inventory

| Role | Page | Entities mapped | Evidence |
|---|---|---|---|
| FINANCE_TEAM | Support Tickets | SupportTicket, TicketComment, User, College | Page–DB Mapping v2 and original |
| HOD | Ticket Verification | SupportTicket, User, College | Page–DB Mapping v2 and original |
| HOD | Help & Support | SupportTicket, TicketComment, User, College | Page–DB Mapping v2 and original |
| MANAGER | Help & Support | SupportTicket, TicketComment, User, College | Page–DB Mapping v2 and original |
| MANAGER | My Tickets | SupportTicket, TicketComment, User, College | Page–DB Mapping v2 and original |
| STUDENT | Help & Support | SupportTicket, TicketComment, User, College | Page–DB Mapping v2 and original |
| STUDENT | My Tickets | SupportTicket, TicketComment, User, College | Page–DB Mapping v2 and original |
| SUPER_ADMIN | Support/Ticket resource permission | SupportTicket resource per RBAC | No dedicated page listed in Page–DB Mapping |

The page mapping does not define exact routes, page layout, actions, or the distinction between Help & Support and My Tickets.

## 6. Functional inventory

| Function ID | Function | Primary actor | Entry point | Type | Status |
|---|---|---|---|---|---|
| TICKET-F001 | Submit/create support ticket | Student; staff create permission per RBAC | Help & Support / API | Create | Detailed draft; workflow gaps |
| TICKET-F002 | List/view requester’s own tickets | Student / requester | My Tickets | Read | Source-aligned |
| TICKET-F003 | View ticket detail and history | Requester / scoped staff | Ticket item | Read | Decision required |
| TICKET-F004 | Add ticket comment/reply | Requester / authorized staff | Ticket detail | Create | Decision required |
| TICKET-F005 | Staff ticket queue/list and review | Finance/HOD/Manager/Super Admin per RBAC | Support Tickets / Help & Support | Read | Decision required |
| TICKET-F006 | Assign/reassign ticket | Authorized staff | Ticket detail/queue | Update | Schema supports assignee; rules TBD |
| TICKET-F007 | Update ticket status | Authorized staff/requester as approved | Ticket detail/queue | Update/workflow | Status enum source-confirmed; transitions TBD |
| TICKET-F008 | Set/update category and priority | Authorized staff / requester policy TBD | Ticket form/detail | Update | Enum values source-confirmed; edit rules TBD |
| TICKET-F009 | Request more information and respond | Staff/requester | Ticket detail | Workflow/comment | Enum status supports value; flow TBD |
| TICKET-F010 | Resolve, close, or reopen ticket | Authorized actor TBD | Ticket detail/queue | Workflow | Status enum source-confirmed; transitions TBD |
| TICKET-F011 | Escalate ticket | Authorized actor TBD | Ticket detail/queue | Workflow | Schema fields/status support terminology; policy TBD |
| TICKET-F012 | HOD ticket verification | HOD | Ticket Verification page | Review/verification | Page is source-confirmed; behavior TBD |
| TICKET-F013 | Add/view internal comments and notes | Staff | Ticket detail | Comment/update | Schema has isInternal and notes; visibility TBD |

## 7. Data and state reference

### 7.1 SupportTicket entity

| Field | Type | Source meaning / known constraint | Unknowns |
|---|---|---|---|
| id | varchar | Ticket identifier | ID format/generation |
| collegeId | varchar | College association | Requiredness and tenant assignment rule |
| requesterId | varchar | User who requests/creates ticket | Requester eligibility; relationship constraints |
| assignedToId | varchar | Assigned user reference | Optionality, allowed assignee roles, assignment policy |
| category | TicketCategory | Ticket category enum | Default, editability, category routing |
| priority | TicketPriority | Ticket priority enum | Default, who sets it, priority criteria |
| status | TicketStatus | Ticket status enum | Default and valid transitions |
| title | varchar(255) | Ticket title | Requiredness, blank handling |
| description | text | Ticket description | Requiredness, format/size |
| resolutionNotes | text | Resolution notes | Requiredness, visibility, editability |
| escalationNotes | text | Escalation notes | Requiredness, visibility, editability |
| resolvedAt | timestamp | Resolution timestamp | When set/cleared, timezone |
| escalatedAt | timestamp | Escalation timestamp | When set/cleared, timezone |
| createdAt | timestamp | Creation timestamp | Default/immutability/timezone |
| updatedAt | timestamp | Update timestamp | Update semantics |

### 7.2 TicketComment entity

| Field | Type | Source meaning / known constraint | Unknowns |
|---|---|---|---|
| id | varchar | Comment identifier | ID generation |
| ticketId | varchar | Ticket association | FK/requiredness |
| authorId | varchar | User who authored comment | Eligible author roles |
| message | text | Comment content | Requiredness, format/size, attachments |
| isInternal | boolean | Internal/public visibility flag indicator | Default, allowed roles, requester visibility, editing |
| createdAt | timestamp | Comment creation time | Timestamp policy |

Schema relationships:
- College has SupportTickets (college_has_support_tickets).
- User creates SupportTicket (user_creates_support_ticket).
- SupportTicket has TicketComments (ticket_has_comments).
- User writes TicketComment (user_writes_ticket_comment).

The schema does not define comment editing/deletion, ordering, attachment support, or visibility rules.

### 7.3 Enum values

**TicketStatus**
- SUBMITTED
- RESOLVED
- ESCALATED
- MORE_INFO_REQUESTED
- CLOSED
- REOPENED

**TicketPriority**
- LOW
- MEDIUM
- HIGH
- URGENT

**TicketCategory**
- TECHNICAL
- BILLING
- ACADEMIC
- ACCOUNT

The source schema provides enum values, not transition rules, default values, business definitions, or allowed role actions for each value.

### 7.4 Candidate lifecycle vocabulary — not an approved state machine

| Status | Source support | Known limitation |
|---|---|---|
| SUBMITTED | Enum value | Initial/default state not explicitly stated |
| MORE_INFO_REQUESTED | Enum value | Who requests information, response behavior, and next status unknown |
| ESCALATED | Enum value; escalatedAt and escalationNotes fields | Escalation trigger, destination, SLA, and allowed transitions unknown |
| RESOLVED | Enum value; resolvedAt and resolutionNotes fields | Resolution authority and required notes unknown |
| CLOSED | Enum value | Closure rules and whether resolution is prerequisite unknown |
| REOPENED | Enum value | Who can reopen, time window, and next state unknown |

Do not assume that the enum order is a transition sequence. Canonical lifecycle decisions belong in Module 17.

## 8. Business rules and validations

### 8.1 Source-confirmed / derived rules

| Rule ID | Rule | Evidence |
|---|---|---|
| TICKET-BR-001 | SupportTicket has College, requester, assignee, category, priority, status, title, description, notes, timestamps fields | Schema |
| TICKET-BR-002 | TicketComment is associated with a ticket and author | Schema relationships |
| TICKET-BR-003 | Category values are TECHNICAL, BILLING, ACADEMIC, ACCOUNT | Schema enum |
| TICKET-BR-004 | Priority values are LOW, MEDIUM, HIGH, URGENT | Schema enum |
| TICKET-BR-005 | Status values are SUBMITTED, RESOLVED, ESCALATED, MORE_INFO_REQUESTED, CLOSED, REOPENED | Schema enum |
| TICKET-BR-006 | Student Support/Ticket permission is Create and View Own | RBAC |
| TICKET-BR-007 | Staff Support/Ticket actions require role permission and scope checks | RBAC |
| TICKET-BR-008 | Frontend visibility does not replace backend authorization | RBAC security rule |
| TICKET-BR-009 | isInternal is a boolean field on TicketComment; visibility policy must be explicitly resolved | Schema + missing policy |
| TICKET-BR-010 | Business Flow does not define ticket workflow | Source review finding |

### 8.2 Validation reference

| Validation ID | Condition | Validation point | Expected behavior | User feedback |
|---|---|---|---|---|
| TICKET-VAL-001 | Actor is unauthenticated or session expired | Module 1 / API | Deny; no ticket mutation | Shared auth response |
| TICKET-VAL-002 | Actor lacks action permission | Server/API | Deny; no unauthorized change | Shared authorization response |
| TICKET-VAL-003 | Ticket belongs to another requester / out of scope | Server/API | Deny or safe not-found | Message policy TBD |
| TICKET-VAL-004 | Client forges collegeId/requesterId/assignedToId | Server/API | Do not trust input; validate against identity and scope | Message TBD |
| TICKET-VAL-005 | Title exceeds 255 characters | Ticket create/update | Reject | Exact copy TBD |
| TICKET-VAL-006 | Category outside TicketCategory enum | Server/API | Reject | Exact copy TBD |
| TICKET-VAL-007 | Priority outside TicketPriority enum | Server/API | Reject | Exact copy TBD |
| TICKET-VAL-008 | Status outside TicketStatus enum | Server/API | Reject | Exact copy TBD |
| TICKET-VAL-009 | Comment has invalid author/ticket association | Server/API | Reject | Exact copy TBD |
| TICKET-VAL-010 | Comment visibility (isInternal) changed by unauthorized actor | Server/API | Deny/reject; protect private content | Exact copy TBD |
| TICKET-VAL-011 | Status transition not allowed by approved lifecycle | Server/API | Reject and leave state unchanged | Transition message TBD |
| TICKET-VAL-012 | Assignee is invalid, inactive, or out of scope | Server/API | Reject; exact eligible-assignee policy TBD | Message TBD |

## 9. Notifications, audit, and integrations

- **Module 13 — Notifications:** Ticket notifications may be relevant because NotificationType includes TICKET, but no ticket event trigger, recipient, channel, or message content is source-defined. Do not assume notifications are sent on create/comment/assign/status changes.
- **Module 15 — Audit Logging:** AuditLog is a separate module/schema entity. Ticket audit requirements (create, assign, status changes, internal comments, escalation, resolution) are not enumerated in the supplied sources and require a shared policy.
- **Module 1 — Authentication & User Lifecycle:** identity, active account, and session behavior.
- **Module 2 — Organization Management:** college/department/section scope and ticket ownership.
- No external support platform, email gateway, or service desk integration is named in the supplied sources.

## 10. Cross-module dependencies

- **Module 1:** authentication, account state, actor identity.
- **Module 2:** College/Department/Section/Group scope and staff assignment eligibility.
- **Module 13:** ticket notification events, recipients, delivery.
- **Module 15:** audit action catalog, actor/target/scope, retention.
- **Module 17:** canonical ticket lifecycle/state transitions.
- **Module 18:** shared validation, authorization, concurrency, retry, partial-failure, and error handling.

## 11. Source gaps and conflicts

1. Business Flow v2 and original do not include ticket workflow steps.
2. Page–DB Mapping identifies Support Tickets, Help & Support, My Tickets, and HOD Ticket Verification, but does not define what each page allows.
3. RBAC provides resource-level permissions; it does not map individual operations such as assign, escalate, resolve, close, reopen, verify, or request more information.
4. Student has C/V OWN. It is not stated whether a Student may comment, edit the initial ticket, change category/priority, reopen, or close.
5. TicketComment.isInternal is present, but who can create internal comments and who can see them is not defined.
6. assignedToId is present, but eligible assignee roles, queue/assignment model, and reassign permissions are not specified.
7. TicketStatus enum does not define initial state or allowed transitions.
8. resolvedAt and escalatedAt fields do not define timestamp setting/clearing rules.
9. Category/priority defaults, edit permissions, and operational definitions are absent.
10. Ticket Verification is a page name only; verification outcome, verification criteria, and permissions are unknown.
11. No SLA, escalation timer, support-hours, retention, deletion, attachment, or satisfaction policy is documented.
12. Finance Team, HOD, Manager, and Super Admin ticket visibility/assignment scopes require more precise resource-level policy.

## 12. Decision log

See [TICKET decision log](./FUNCTION-SPECIFICATIONS.md#decision-log). All decisions remain open for the later product review.

## 13. QA and review readiness

- [Function-level specifications](./FUNCTION-SPECIFICATIONS.md)
- [QA test cases](./QA-TEST-CASES.md)

**Readiness:** Detailed draft for source-based module completion. It is not an approved target workflow or implementation specification until the ticket lifecycle, role-specific actions, comment visibility, assignment, verification, notification, and audit decisions are reviewed.

### Review checklist
- [x] Source documents identified and reviewed.
- [x] Business Flow absence of ticket process recorded.
- [x] RBAC role/scope matrix included.
- [x] Ticket and comment fields/enums documented.
- [x] Function inventory and detailed drafts included.
- [x] QA cases include expected results.
- [x] Source gaps preserved and linked to decisions.
- [ ] Ticket lifecycle and action policy approved.
- [ ] Internal comment visibility approved.
- [ ] Target behavior approved.
