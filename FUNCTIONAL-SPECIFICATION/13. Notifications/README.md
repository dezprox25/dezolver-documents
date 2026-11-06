# DEZOLVER — Module 13: Notifications

**Document ID:** NOTIF-FS-001  
**Version:** 0.1  
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

### Evidence labels
- **Source-confirmed:** explicitly present in the supplied source.
- **Derived:** security/functional consequence logically derived from source, not directly stated as UI behavior.
- **Recommendation:** proposal for review only; not approved.
- **TBD / Decision Required:** missing or conflicting behavior.

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

- **Module 1 — Authentication & User Lifecycle:** authenticated user identity, active/suspended account behavior, session expiry.
- **Module 2 — College / Department / Section Management:** role scope and organizational context.
- **Module 3 — Subscription & Seat Management:** possible billing notification event sources (not defined here).
- **Modules 4–12:** possible course, assignment/lab, grade, contest, group, ticket, certificate event sources; actual triggers and recipient rules are TBD.
- **Module 14 — Support Tickets:** potential TICKET notifications; event/recipient mapping TBD.
- **Module 15 — Audit Logging:** central audit definitions; notification create/update/delete/read events require decision.
- **Module 17 — Functional State Machines:** notification lifecycle if more than read/unread/delete is approved.
- **Module 18 — Edge Cases & Failure Handling:** shared error response, retries, idempotency, stale updates, and concurrency.

## 10. Source gaps and known conflicts

1. RBAC permits V/C/U for Finance/HOD/Manager within scope, but Student is V OWN and Super Admin is V/C/U/D. It does not define whether C/U refers to sending, editing, or recipient-owned records.
2. The Notification schema has recipientId, but no sender/creator field. The source does not explain whether administrative users create notifications for other users, groups, sections, or whole Colleges.
3. Page mappings list Notifications for Finance, HOD, Manager, Student, but not Super Admin. RBAC nevertheless grants Super Admin full CRUD.
4. Student Dashboard uses Notification in Page–DB Mapping v2; original Page–DB Mapping does not list Notification on Student Dashboard. This discrepancy is preserved.
5. Business Flow documents do not identify notification events or triggers.
6. No external delivery channels, preferences, delivery status, templates, retry behavior, or notification retention are documented.
7. No default isRead value or read/unread UI actions are documented.
8. No notification-to-resource URL contract or security rules are documented.

## 11. Decision log

See [NOTIF decision log](./FUNCTION-SPECIFICATIONS.md#decision-log). All open items remain unresolved until later product review.

## 12. QA coverage and review

- [Function-level specifications](./FUNCTION-SPECIFICATIONS.md)
- [QA test cases](./QA-TEST-CASES.md)

**Readiness:** Detailed draft. It is suitable for source-based module completion and later review, but not implementation approval. Notification trigger catalog, sender/recipient semantics, mutation permissions, delivery channels, and URL policy must be decided before those behaviors are implemented.

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
