# DEZOLVER — Module 13: Notifications

**Document ID:** NOTIF-FS-001  
**Version:** 0.2 — legacy Notifications cross-check and cross-module reconciliation  
**Status:** Detailed draft — source-aligned; notification triggers, mutation semantics, and delivery decisions remain open  
**Governing standard:** [Functional Specification Standard v1.0](../FUNCTIONAL-SPECIFICATION-STANDARD.md)  
**Repository inclusion:** Draft only; not approval of target behavior.

## 1. Purpose and outcome

Module 13 specifies notification records and the Notifications pages identified in the supplied Page–DB Mapping documents. It documents source-confirmed role permissions and the Notification schema, then separates known behavior from missing rules.

The intended functional area is notification access and handling. The supplied sources do **not** define notification event triggers, delivery channels, message templates, preferences, real-time delivery, retry policies, or detailed page interactions. These must remain decision items.

## 2. Scope

### In scope
- Notification list/inbox and detail access.
- Recipient ownership and role/scope authorization.
- Read/unread state based on isRead.
- Notification create/update/delete operations reflected in RBAC, while recording missing action semantics.
- linkUrl behavior as a possible notification destination.
- Student Dashboard notification data reference.
- Notification types in the schema enum.
- QA, cross-module dependencies, privacy, and open decisions.

### Out of scope / not source-defined
- Email, SMS, mobile push, browser push, WebSocket, or other external delivery channels.
- Notification event catalog and event-to-recipient mapping.
- Notification templates, localization, preference/opt-out settings, digest scheduling.
- Delivery status, retries, dead-letter behavior, read receipts beyond isRead.
- SLA, retention, archiving, rate limits, and deduplication.
- Exact UI labels, ordering, pagination, filters, unread badges, and notification sound.
- Product behavior not present in supplied sources.

## 3. Source documents and evidence

| Source | Evidence used | Limitations |
|---|---|---|
| [Business Flow v2](../../docs/source-text/Business%20Flow%20v2.md) | Reviewed for notification-related business flow | No notification-specific workflow or trigger definitions found |
| [Business Flow original](../../docs/source-text/Business%20Flow.md) | Reviewed for notification-related business flow | No notification-specific workflow or trigger definitions found |
| [Page–DB Mapping v2](../../docs/source-text/Page%20-%20DB%20Mapping%20v2.md) | Notifications pages for Finance Team, HOD, Manager, Student; Student Dashboard references Notification | Page/entity mapping only; does not specify interactions |
| [Page–DB Mapping original](../../docs/source-text/Page%20-%20DB%20Mapping.md) | Notifications pages for Finance Team, HOD, Manager, Student | Older page mapping; Student Dashboard entity mapping differs from v2 |
| [RBAC](../../docs/source-text/RBAC.md) | Notification V/C/U/D permission matrix and role scope rules | Does not define exact CRUD semantics or whether permissions apply to own/issued/recipient notifications |
| [Schema](../../docs/source-text/scheema.md) | Notification fields, User recipient relationship, NotificationType enum | No delivery state, sender, read timestamp, channel, or trigger model |
| [Functional Specification Standard](../FUNCTIONAL-SPECIFICATION-STANDARD.md) | Required function-level detail, evidence labels, decision log, QA/review structure | Governs authoring; does not define product behavior |
| [Historical Module 13 forensic specification](../../functional-spec-from-old%20code/module-13-notifications/SPECIFICATION.md) and [legacy summary](../../functional-spec-from-old%20code/module-13-notifications/README.md) | Legacy email/SES pipeline, WebSocket frames, in-memory Zustand NotificationCenter, missing persistence/API, reported outbox/queue/email defects | LEGACY-SPEC-REPORTED only; this pass reviewed the report artifacts, not the raw application source files |

### Evidence labels
- **Source-confirmed:** explicitly present in the supplied source.
- **Derived:** security/functional consequence logically derived from source, not directly stated as UI behavior.
- **Recommendation:** proposal for review only; not approved.
- **TBD / Decision Required:** missing or conflicting behavior.
- **LEGACY-SPEC-REPORTED:** a behavior stated in the old-code Module 13 forensic specification; this update inspected that report, not the raw application source files, and does not claim fresh code verification.

### Historical Module 13 cross-check — legacy email, WebSocket, and ephemeral inbox

**Evidence status: LEGACY-SPEC-REPORTED.** The old-code Module 13 forensic report labels itself `COMPLETED & FORENSICALLY VERIFIED` and cites backend, worker, WebSocket and frontend files. This pass reviewed the report artifacts in the repository; raw application source files were not independently retrieved. Findings below are attributed to the report, not freshly code-verified, and do not establish target requirements.

| Area | Current target evidence | Historical behavior/defect reported | Target interpretation |
|---|---|---|---|
| In-app persistence | Target conceptual schema defines `Notification` with id, recipientId, title, message, type, linkUrl, isRead, createdAt; Page–DB Mapping associates Notification with role notification pages and Student Dashboard v2. | Old report says none of the legacy Prisma schemas had Notification/UserNotification models; the browser used a Zustand memory store only. | Target is a persisted per-recipient Notification record. Do not copy the old ephemeral-only architecture or infer that a UI frame equals a saved inbox record. |
| Ingress/API | Target Page mapping and schema imply notification access, but the supplied sources do not define route names or API behavior. | Old report says no backend list/detail/read/delete REST endpoints existed. | No legacy endpoints are available as target contracts. List/detail/read/create/update/delete API shapes remain recommendations/decisions. |
| Client behavior | Target source defines `isRead` but not readAt, delivery status, unread count, ordering, mark-all, dismiss or clear semantics. | Old NotificationCenter displayed a bell/popover, 50-item newest-first memory buffer, unread counter, mark-all-read and clear-all actions; refresh erased all items. | UI actions are historical only. A target UI must read persisted state; retention, unread and dismissal behavior remain open. |
| Email | Main target sources define no EmailJob or delivery entity and no fixed channel. Module 1 separately approves security-sensitive password/email-change notifications; Module 3 owns billing reminder terms. | Old report describes BullMQ `email` queue, `EmailConsumer`, AWS SES, managed SES templates or inline fallback, with legacy templates `welcome`, `password-reset`, `invitation`, `submission-graded`, `certificate-issued`, event reminders and payment recovery. | Email is a separate delivery channel from the in-app Notification row. Provider, template catalog, recipient address source, delivery state and retry require decisions; do not copy SES/BullMQ configuration or old templates as target defaults. |
| Real-time | Main target sources do not specify WebSocket channels, push frames, unread counter synchronization or reconnect behavior. | Old report describes user/event WebSocket rooms, socket frames including `notification`, `certificate:issued`, `submission:graded`, `event:started/ended/extended/reminder`, and client toast/store integration. | WebSocket is a possible transport, not the record of truth. Any target channel must authenticate the principal and enforce recipient/scope authorization. Event names and payloads require target contracts. |
| Identity/security email | Module 1 target decisions require email verification for appropriate roles, email password recovery, 15-minute single-use reset token and security emails for password changed/reset and email changed. | Old report says `PasswordResetRequested` was emitted to outbox but had no consumer, so no reset email was queued. | Treat this as a legacy defect warning. Module 13 must provide the approved delivery integration; Module 1 remains authority for reset-token and security-event semantics. Never log or place reset tokens in Notification content. |
| Billing email | Module 3 owns renewal reminders, invoice/payment-failure triggers and configurable reminder settings; Module 13 owns delivery. | Old report says dunning sent `sub.tenantId` (UUID) as the SES recipient address, causing email send failure. | Resolve a verified recipient email from Module 3's approved College/contact source; never treat tenant/College IDs as email addresses. Legacy dunning cron intervals are not target policy. |
| Notification fanout | Target Notification entity is a recipient record; no fanout queue/worker is documented by target sources. | Old report says `notification-fanout` queue was registered but had no processors and no jobs enqueued. | Do not assume that queue or claim fanout is implemented. Recipient expansion, delivery fanout and idempotency are target design decisions. |
| Event routing / triggers | Target NotificationType enum is SYSTEM, ASSIGNMENT, GRADE, CONTEST, BILLING, TICKET; it contains categories only. | Old report maps user creation/invitation, password reset request, contest reminders, grading, certificate issue and dunning to email/WS events. | Enum values do not establish triggers, recipients, channels, timings or content. Use each owning module's approved durable event contract; keep unapproved triggers open. |

**Historical notification findings/defects retained only as regression warnings:** no legacy Notification table/API; loss on refresh/device change; no channel preferences; no notification-fanout worker; unhandled password-reset outbox event; dunning recipient UUID defect; 50-item client-memory cap; and notification delivery not audited in the old report. These are not copied as target architecture or requirements.
## 4. Terminology

| Term | Meaning / status |
|---|---|
| Notification | Schema record with recipient, title, message, type, link, read flag, and creation timestamp |
| Recipient | recipientId identifies the User who receives a Notification (schema relationship) |
| Read / unread | isRead boolean in schema; user action and transition details are not documented |
| Notification type | Enum value: SYSTEM, ASSIGNMENT, GRADE, CONTEST, BILLING, or TICKET |
| Link destination | linkUrl field; navigation and authorization semantics are TBD |
| Delivery | Whether/how notification is surfaced to the recipient; no delivery channel or delivery state is specified |

## 5. Actors, permissions, and scope

### 5.1 RBAC source matrix

| Role | Notification permissions | Source scope |
|---|---|---|
| SUPER_ADMIN | V/C/U/D | Global |
| FINANCE_TEAM | V/C/U* | Assigned College |
| HOD | V/C/U* | Assigned College + Department |
| MANAGER | V/C/U* | Assigned College + Department + permitted Section/Group |
| STUDENT | V OWN | Own records + permitted academic data |

* means the operation is further restricted by authorized data scope in RBAC.

**Important:** The matrix is source-confirmed at the Notification resource level. It does not say whether Finance/HOD/Manager C/U applies to creating announcements, modifying a notification addressed to another user, managing their own inbox, or some other resource-level action. Do not implement an interpretation without resolving NOTIF-D04 through NOTIF-D07.

### 5.2 Scope and security rules
- **Source-confirmed:** Super Admin is global; Finance Team is restricted to its assigned College; HOD to assigned College + Department; Manager to College + Department + assigned Section/permitted Group; Student to own records and permitted academic resources.
- **Derived:** For recipient-owned notifications, server-side access must be based on authenticated recipient identity; client-supplied recipientId must not grant access to another user's inbox.
- **Derived:** Every API must independently check role, permission, scope, and recipient ownership. Hiding controls in the UI is not sufficient.
- **TBD:** Whether a notification has both a creator/sender and recipient; schema only exposes recipientId, not createdById or senderId.
- **Legacy distinction:** The old client `Notification` object (`severity`, `read`, `link`) is not the target schema entity (`type`, `isRead`, `linkUrl`). The old `EmailJobData` and WebSocket frames are not target Notification records.

### 5.3 Page / entry point inventory

| Role | Page or location | Source evidence |
|---|---|---|
| FINANCE_TEAM | Notifications | Page–DB Mapping v2 and original |
| HOD | Notifications | Page–DB Mapping v2 and original |
| MANAGER | Notifications | Page–DB Mapping v2 and original |
| STUDENT | Notifications | Page–DB Mapping v2 and original |
| STUDENT | Student Dashboard references Notification | Page–DB Mapping v2 |
| SUPER_ADMIN | Notification resource/API permission | RBAC; no dedicated Super Admin Notifications page is listed in Page–DB Mapping |

No exact route, page layout, notification drawer, unread badge, or dashboard widget interaction is source-defined.

## 6. Module functional inventory

| Function ID | Function | Primary actor | Entry point | Type | Status |
|---|---|---|---|---|---|
| NOTIF-F001 | List/view recipient notifications | Finance/HOD/Manager/Student; Super Admin per RBAC | Notifications page/API | Read | Detailed draft |
| NOTIF-F002 | View notification detail | Authorized recipient / scoped actor | Notification item | Read | Decision required |
| NOTIF-F003 | Mark notification read/unread | Recipient; admin scope subject to decision | Notification action | Update | Derived from isRead |
| NOTIF-F004 | Create notification record | Super Admin/Finance/HOD/Manager per RBAC | UI/API/system trigger TBD | Create | Decision required |
| NOTIF-F005 | Update notification record | Super Admin/Finance/HOD/Manager per RBAC | UI/API TBD | Update | Decision required |
| NOTIF-F006 | Delete notification record | Super Admin per RBAC | UI/API TBD | Delete | Decision required |
| NOTIF-F007 | Open linked destination | Notification recipient | linkUrl | Navigation | Decision required |
| NOTIF-F008 | Show notifications on Student Dashboard | Student | Dashboard | Read / summary | Source entity mapping; UI behavior TBD |
| NOTIF-F009 | Generate/distribute event-driven notifications | System / event source | Event trigger TBD | Workflow | Not source-confirmed |

## 7. Data and state reference

### 7.1 Notification entity (schema source)

| Field | Type | Source meaning | Known constraints | Unknowns |
|---|---|---|---|---|
| id | varchar | Notification identifier | Type only | ID generation/format |
| recipientId | varchar | User recipient relation | References recipient User in schema relationship | Requiredness/FK constraints/recipient rules |
| title | varchar(255) | Notification title | Maximum storage type length indicated | Requiredness, blank handling, uniqueness |
| message | text | Notification message | Text field | Requiredness, format, HTML/plain text, size policy |
| type | NotificationType | Category | Enum: SYSTEM, ASSIGNMENT, GRADE, CONTEST, BILLING, TICKET | Requiredness, type-to-role/event mapping |
| linkUrl | varchar(1024) | Link associated with notification | Maximum storage type length indicated | Optionality, URL allowlist, internal/external, access checks |
| isRead | boolean | Read-state flag | Boolean | Default value, transition permissions, unread reset rules |
| createdAt | timestamp | Creation timestamp | Timestamp type | Timezone, immutable/default semantics |

### 7.2 NotificationType enum
Source-confirmed values:
- SYSTEM
- ASSIGNMENT
- GRADE
- CONTEST
- BILLING
- TICKET

The enum does not define which event produces each type, which roles may receive each type, or what text/link should be included.

### 7.3 Relationship
Schema relationship: **User receives Notification** (user_receives_notifications). recipientId is the recipient reference.

### 7.4 State reference
The schema provides only isRead boolean, not a formal lifecycle/status enum.

| Current value | Candidate action | Next value | Evidence/status |
|---|---|---|---|
| false | Mark as read | true | Derived candidate from boolean; action UX TBD |
| true | Mark as unread | false | Derived candidate; availability TBD |
| Either | Notification deleted | Record absent | RBAC Super Admin D; deletion semantics TBD |

No sent, delivered, failed, archived, dismissed, scheduled, or expired state is present in the supplied schema. Do not assume such fields/states exist.

## 8. Business rules and validations

| Rule ID | Rule | Evidence |
|---|---|---|
| NOTIF-BR-001 | Notification is associated with a recipient User | Schema |
| NOTIF-BR-002 | Notification title is stored as varchar(255) | Schema |
| NOTIF-BR-003 | Notification message is stored as text | Schema |
| NOTIF-BR-004 | Type must use the schema NotificationType enum if persisted as that enum | Schema |
| NOTIF-BR-005 | isRead is boolean | Schema |
| NOTIF-BR-006 | RBAC permissions are restricted by role and scope | RBAC |
| NOTIF-BR-007 | Student notification read access is own-only | RBAC |
| NOTIF-BR-008 | Finance Team, HOD, and Manager notification access is scope-limited | RBAC |
| NOTIF-BR-009 | Frontend page visibility does not replace server-side authorization | RBAC security principle |
| NOTIF-BR-010 | Notification event triggers, recipients, delivery channels, and creation policies are not source-defined | Source review finding |
| NOTIF-BR-011 | Target Notification is a per-recipient record with only the supplied schema fields; the legacy client object is a distinct transient shape | Target schema vs LEGACY-SPEC-REPORTED |
| NOTIF-BR-012 | Recipient identity and organizational scope must be validated server-side; client recipientId cannot grant access | RBAC / derived |
| NOTIF-BR-013 | A persistent Notification row, email delivery attempt and realtime frame are distinct outcomes; one does not prove the others succeeded | Derived from target schema and Module 18 ownership |
| NOTIF-BR-014 | Password reset tokens, access/refresh tokens, source code, hidden testcase data, and raw provider credentials must not be placed in notification content or logs | AUTH target decisions / Modules 6–7 security |
| NOTIF-BR-015 | A NotificationType enum value is a category only; it does not define event trigger, recipient, timing or channel | Target schema / source review finding |

### Validation table

| Validation ID | Condition | Validation point | Expected behavior | User feedback |
|---|---|---|---|---|
| NOTIF-VAL-001 | Notification ID does not exist | Server on detail/update/delete | Not-found/safe response under shared policy | Exact message TBD |
| NOTIF-VAL-002 | Actor lacks Notification action permission | Server/API | Deny; no unauthorized data mutation | Generic authorization feedback per shared policy |
| NOTIF-VAL-003 | Notification belongs to another Student | Server/API | Deny/safe not-found; do not disclose details | Exact message TBD |
| NOTIF-VAL-004 | recipientId supplied by client conflicts with authenticated identity | Server/API | Do not allow scope escalation; derive/check identity | Message TBD |
| NOTIF-VAL-005 | Title exceeds 255 characters | Create/update validation | Reject; no invalid data saved | Message intent: title too long; exact copy TBD |
| NOTIF-VAL-006 | Type is outside enum | Server validation | Reject invalid type | Exact message TBD |
| NOTIF-VAL-007 | linkUrl exceeds 1024 characters | Server validation | Reject if field is submitted; exact URL validation TBD | Message TBD |
| NOTIF-VAL-008 | isRead is not a boolean | Server validation | Reject malformed state update | Message TBD |
| NOTIF-VAL-009 | URL uses unsupported/unsafe destination | Validation policy TBD | Must not open unsafe destination; allowlist/security rule TBD | Message TBD |

## 9. Dependencies

- **Module 1 — Authentication & User Lifecycle:** authenticated user identity, active/suspended account behavior and session expiry. Approved security-email events are password changed/reset and email changed; password reset tokens are single-use and expire after 15 minutes. Do not add notifications for MFA toggles, role/scope change, suspension/deactivation or new-device login under current AUTH decisions.
- **Module 2 — College / Department / Section Management:** role scope and organizational context.
- **Module 3 — Subscription & Seat Management:** owns renewal reminders, invoices, payment failures and their configurable billing terms; Module 13 owns notification recipients/channels/content/delivery. Never use College/tenant IDs as email destinations.
- **Module 4 — Course & Enrollment:** adopted enrollment/withdrawal/HOD-verified completion events may produce notices; Course completion is not inferred from Lab/Assessment/Contest work.
- **Modules 6–8 — Submission/Evaluation/Labs:** Module 6/7 emit safe committed SubmissionGraded events after result persistence; Module 8 emits safe committed Lab events. Module 13 resolves recipients/channels; payloads must not include source code, Judge0 tokens, hidden testcase data or privileged diagnostics.
- **Module 9 — Assessments:** notification event definitions are present in its draft, but Module 9's decisions were prematurely marked adopted before the all-modules review; treat these triggers as candidates pending cross-module decision review.
- **Module 10 — Groups:** membership/Group events require approved producer contracts; do not infer broadcasts from Group membership alone.
- **Module 11 — Contests:** notification triggers, recipients, audit and reminders remain open under CON-D22; legacy 24h/1h contest reminders are not target defaults.
- **Module 12 — Certificates:** current target sources do not define certificate-issued/revoked notification triggers; CERT-D21 remains open.
- **Module 14 — Support Tickets:** NotificationType includes TICKET, but ticket events/recipients/channels are not source-defined; do not send on create/comment/assign/status change without approval.
- **Module 14 — Support Tickets:** potential TICKET notifications; event/recipient mapping TBD.
- **Module 15 — Audit Logging:** owns canonical audit event names, fields, metadata allowlists, PII redaction and retention. Notification delivery logs are not a substitute for audit events; AUD-D02/D06/D07/D17/D18/D21 coordination remains open.
- **Module 17 — Functional State Machines:** owns state transitions for identity, course, submission, contest, certificate, ticket and outbox; Module 13 consumes only committed events and does not redefine source state.
- **Module 18 — Edge Cases & Failure Handling:** owns outbox durability/replay, retries, idempotency, duplicate delivery, provider outage, dead-letter/recovery and partial failure. A Notification row and an email/WebSocket delivery attempt are separate outcomes.

## 10. Source gaps and known conflicts

1. RBAC permits V/C/U for Finance/HOD/Manager within scope, but Student is V OWN and Super Admin is V/C/U/D. It does not define whether C/U refers to sending, editing, or recipient-owned records.
2. The Notification schema has recipientId, but no sender/creator field. The source does not explain whether administrative users create notifications for other users, groups, sections, or whole Colleges.
3. Page mappings list Notifications for Finance, HOD, Manager, Student, but not Super Admin. RBAC nevertheless grants Super Admin full CRUD.
4. Student Dashboard uses Notification in Page–DB Mapping v2; original Page–DB Mapping does not list Notification on Student Dashboard. This discrepancy is preserved.
5. Business Flow documents do not identify notification events or triggers.
6. No external delivery channels, preferences, delivery status, templates, retry behavior, or notification retention are documented.
7. No default isRead value or read/unread UI actions are documented.
8. No notification-to-resource URL contract or security rules are documented.
9. Historical email and WebSocket transport are not current target API/channel contracts; raw old source has not been independently inspected here.
10. Main target schema has no senderId, channel, deliveryStatus, attemptCount, deliveredAt, readAt, dismissedAt, or archivedAt fields. Do not invent them without a schema decision.
11. The NotificationType enum does not map event → type, recipient, channel or template.
12. Module 13 must reconcile durable in-app persistence with external email and realtime channels without making an external send result authoritative for the saved inbox.

## 11. Decision log

See [NOTIF decision log](./FUNCTION-SPECIFICATIONS.md#decision-log). All open items remain unresolved until later product review.

## 12. QA coverage and review

- [Function-level specifications](./FUNCTION-SPECIFICATIONS.md)
- [QA test cases](./QA-TEST-CASES.md)

**Readiness:** Source-first detailed draft. The target Notification fields and RBAC/page evidence are documented; historical email/WebSocket/memory-store behavior is separately labeled LEGACY-SPEC-REPORTED. NOTIF-D01–D28 remain open. No legacy queue, route, template, trigger, schedule, payload, persistence limitation or bug is adopted as target behavior. Completion of all modules precedes the cross-module/product-owner decision pass; no ADOPTED-DECISIONS file is created.

### Review checklist
- [x] Source docs identified and reviewed.
- [x] Source facts separated from derived behavior and decisions.
- [x] RBAC and scope matrix included.
- [x] Schema fields and enum values documented.
- [x] Function inventory and function-level drafts included.
- [x] QA cases with expected outcomes included.
- [x] Source discrepancies preserved.
- [ ] Product decisions resolved.
- [ ] Target behavior approved.
