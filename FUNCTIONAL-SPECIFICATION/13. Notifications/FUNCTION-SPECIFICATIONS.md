# Module 13 — Notifications: Function-Level Specifications

**Document ID:** NOTIF-FS-FUNCTIONS-001 | **Version:** 0.2 | **Status:** Detailed draft; decisions open  
**Standard:** [Functional Specification Standard](../FUNCTIONAL-SPECIFICATION-STANDARD.md)  
**Parent:** [Overview](./README.md) · [QA cases](./QA-TEST-CASES.md)

> Notification behavior not explicitly documented in the sources is labeled as candidate/decision-required behavior. No recommendation below is approved target behavior.

## Shared actor and permission context

| Role | Source permission | Scope |
|---|---|---|
| SUPER_ADMIN | V/C/U/D | Global |
| FINANCE_TEAM | V/C/U* | Assigned College |
| HOD | V/C/U* | Assigned College + Department |
| MANAGER | V/C/U* | College + Department + permitted Section/Group |
| STUDENT | V OWN | Own records |

RBAC resource-level C/U permissions do not define whether staff can send, author, edit, or manage recipient notifications. Student access is own-only. Backend authorization must enforce role, scope, and ownership independently of UI visibility.

## Historical function/channel crosswalk — legacy implementation is not target behavior

**Evidence status: LEGACY-SPEC-REPORTED.** The historical Module 13 report identifies the following behavior in the old codebase. This review inspected the old-code report only; raw application files were not independently retrieved. The route, channel, queue and event names below are traceability data, not target contracts.

| Target function / concern | Legacy routes or implementation reported | Target boundary |
|---|---|---|
| NOTIF-F001 List | No backend notification list endpoint; Zustand store held only frames received in current browser memory. | Target schema provides persistent per-recipient Notification data. API/list behavior remains recommendation; do not retain the memory-only design or 50-item cap by assumption. |
| NOTIF-F002 Detail | No backend notification detail endpoint was reported. | Target detail read must recheck recipient/role/scope. The route and projection are not sourced. |
| NOTIF-F003 Read state | Legacy client used `read` in RAM, mark-one/all-read and cleared the list; no persisted read API. | Target field is `isRead`; initial value, mark-all, auto-read, unread and history behavior remain decisions. Do not map client-only `read` directly into a new schema field. |
| NOTIF-F004 Create | No Notification REST create endpoint/table was reported. WebSocket frames could be pushed by backend producers; email jobs were separate. | Target RBAC grants C to Super Admin/Finance/HOD/Manager but does not define whether this means announcements, recipient-record creation, or system event creation. Do not invent a broadcast model. |
| NOTIF-F005 Update | No persisted Notification update API was reported; client read changes were transient only. | Target update permission ambiguity remains NOTIF-D04/D06/D07; recipientId and createdAt are not client-authoritative. |
| NOTIF-F006 Delete | Client `clearAll` removed items from Zustand memory; no backend delete endpoint or retention mechanism was reported. | Client clear is not equivalent to Super Admin D permission or target deletion. Hard delete/archive/dismiss/retention remain NOTIF-D17/D19. |
| NOTIF-F007 Link | Old transient payload used optional `link`; target schema uses `linkUrl`. | These fields are not interchangeable contracts. Validate scheme/host and require destination module to reauthorize access. |
| NOTIF-F008 Dashboard | Old UI report describes a header Bell popover, unread badge, 320px popover, empty state and Sonner toasts; not a target Dashboard API. | Target Student Dashboard Notification entity mapping exists in v2 but not original Page–DB Mapping. Widget/list/count/realtime behavior remains NOTIF-D14/D22. |
| NOTIF-F009 Event/Email/WebSocket | Legacy event labels included `UserCreated`, `UserInvitationAccepted`, `PasswordResetRequested`, `SubmissionGraded`, `CertificateIssued`, Event reminder, and dunning. Socket frames included `notification`, `certificate:issued`, `submission:graded`, `event:started/ended/extended/reminder`. | Use current owner-module event contracts only. Do not import legacy schedules, event payloads, severity, channels or source modules as the target catalogue. |

### Historical email delivery details reported

- BullMQ queue `email`, job name `send`, worker `EmailConsumer`, and AWS SES delivery were reported. The worker supported managed SES templates when `SES_USE_TEMPLATES === 'true'` and inline text fallback otherwise; non-production mock mode logged job payload variables.
- Reported inline template keys included `welcome`, `password-reset`, `invitation`, `submission-graded`, `certificate-issued`, plus event reminder and `payment_recovery` templates in source-specific paths. These names/content are not approved target template identifiers.
- The report states an outbox `PasswordResetRequested` event had no consumer, so the reset email was not enqueued. It reports `notification-fanout` queue registration without worker processors/jobs.
- The report describes a dunning defect where `sub.tenantId` (UUID) was passed as the email `to` address. It also reports the in-app Notification object was capped to 50 items in browser RAM and disappeared on refresh/device change.
- The report identifies live socket rooms including `user:${userId}` and event rooms. It reports user-room authorization, but the target channel handshake, event scope, payload schema and reconnect guarantees are not specified here.

## NOTIF-F001 — List/view notifications

**Requirement status:** Notifications page/entity source-confirmed; list behavior derived  
**Specification status:** Detailed draft; decisions open  
**Actors:** All five roles per RBAC; page mapping identifies Finance Team, HOD, Manager, Student pages  
**Entry:** Notifications page/API

### Purpose
Display notification records available to authenticated actor.

### Preconditions
- Authenticated account allowed by Module 1.
- Notification view permission.
- Role scope resolved through Module 2.

### Inputs
| Input | Source/type | Required | Validation |
|---|---|---|---|
| actor identity | Session | System | Server-derived |
| recipientId | Query/context | Not trusted | Student must be derived from authenticated user |
| pagination/filter/sort | Query | TBD | Shared API contract |
| notificationId | Route for detail | Required for detail | Exists and is in permitted scope |

### Main flow
1. Actor opens Notifications.
2. Server resolves identity, role, and scope.
3. Server checks view permission.
4. Server queries only records within approved recipient/scope model.
5. Server returns permitted Notification fields.
6. UI displays the list; ordering, filters, pagination, loading/empty states are TBD.

### Data
Read: id, recipientId, title, message, type, linkUrl, isRead, createdAt.

### Rules
- Student list is own-only.
- Client recipientId cannot broaden access.
- Staff scope applies server-side; staff inbox vs recipient-wide visibility is NOTIF-D05.
- Super Admin global access follows RBAC.

### Failure/alternate
- Expired session: Module 1.
- Unauthorized role: deny.
- Foreign recipient/resource: deny or safe not-found under shared policy.
- Empty list: show approved empty state.
- Invalid query: shared API validation.
- Backend error: Module 18.

### Acceptance
- Student cannot list another user’s notifications.
- Role/scope checks are server-enforced.
- Forged recipient/scope input does not reveal data.
- Empty, forbidden, and failure responses are distinguishable.

### QA
NOTIF-QA-001–009.

## NOTIF-F002 — View notification detail

**Status:** Derived candidate; detailed view is not expressly defined  
**Actor:** Recipient; staff detail scope TBD  
**Entry:** Select notification or direct API request

### Purpose
Show notification details if the product supports a detail view.

### Preconditions and inputs
Authenticated actor with view permission; notificationId required; recipientId cannot serve as authorization proof.

### Candidate flow
1. Actor selects notification.
2. Server retrieves record.
3. Server repeats role/scope/recipient authorization.
4. Server returns permitted detail fields.
5. UI displays title, message, type, creation time, read state, and link if approved.

### Rules / gaps
- Student owner check is required.
- Staff detail scope is NOTIF-D05.
- Whether opening marks read is NOTIF-D10; do not mutate automatically by assumption.
- Exact page, modal, and field projection are TBD.

### Failure
Missing record: safe not-found. Foreign/out-of-scope: deny/safe not-found. Expired session: Module 1. Backend failure: Module 18.

### Acceptance
Direct/guessed IDs cannot bypass scope. Opening does not mutate isRead unless approved.

### QA
NOTIF-QA-010–014.

## NOTIF-F003 — Mark notification read/unread

**Status:** isRead field source-confirmed; action derived candidate  
**Actor:** Recipient; staff mutation policy TBD  
**Entry:** Notification read-state action or auto-read candidate

### Purpose
Change a Notification’s isRead boolean if product approves this action.

### Inputs
- notificationId: required.
- isRead: boolean, required for explicit state update.
- actor identity: server-derived.

### Candidate flow
1. Actor requests read/unread change.
2. Server loads notification and checks access.
3. Server validates boolean and action permission.
4. Server updates isRead.
5. Server returns persisted state.
6. UI reflects server result.

### Rules / validation
- Schema defines isRead boolean; default is not specified.
- Student own read-state permissions, staff mutation rights, mark-unread support, and auto-read behavior are NOTIF-D06/D10.
- No readAt timestamp exists in schema.

### Failure
Unauthorized owner: deny and leave state unchanged. Invalid boolean: reject. Repeated state request: idempotency TBD. Concurrent update: Module 18.

### Acceptance
Only authorized state transitions occur; failed requests do not change the record; UI does not show unpersisted state as success.

### QA
NOTIF-QA-015–022.

## NOTIF-F004 — Create notification record

**Status:** RBAC Create for SUPER_ADMIN/FINANCE_TEAM/HOD/MANAGER source-confirmed; workflow decision-required  
**Actors:** SUPER_ADMIN, FINANCE_TEAM, HOD, MANAGER  
**Entry:** UI/API/system event TBD

### Purpose
Create a Notification for an approved recipient.

### Preconditions
Authenticated actor/system; create permission; recipient and target scope valid; creation policy approved.

### Inputs
| Field | Type | Required | Validation |
|---|---|---|---|
| recipientId | varchar | Required by recipient relation | User exists and target permitted |
| title | varchar(255) | TBD | Length <= 255; blank policy TBD |
| message | text | TBD | format/size/security TBD |
| type | NotificationType | TBD | SYSTEM, ASSIGNMENT, GRADE, CONTEST, BILLING, TICKET |
| linkUrl | varchar(1024) | TBD | URL policy TBD |
| isRead | boolean | System value TBD | Do not trust client default |
| createdAt | timestamp | System | Timestamp policy TBD |

### Candidate flow (not approved)
1. Actor or system event requests creation.
2. Server checks permission/scope.
3. Server validates recipient eligibility.
4. Server validates title/message/type/link.
5. System sets id, createdAt, and isRead according to approved policy.
6. Notification record is created.
7. Creation result returned. External delivery is separate and not implied by row creation.

### Rules / gaps
- C permission does not establish announcements or staff-to-student sending.
- Schema has no sender/creator field.
- Recipient target granularity (user/section/department/College/group) is unknown.
- Trigger/type mapping, initial isRead value, and delivery are TBD.

### Failure
Unauthorized target: reject. Invalid input/type: reject. Duplicate event/retry: NOTIF-D16. Persist failure: no false success (Module 18).

### Acceptance
Only permitted recipient targeting; server validates role/scope; invalid type rejected; row creation is distinguished from delivery.

### QA
NOTIF-QA-023–032.

## NOTIF-F005 — Update notification

**Status:** RBAC Update for Super Admin/Finance/HOD/Manager source-confirmed; update semantics TBD  
**Actors:** SUPER_ADMIN, FINANCE_TEAM, HOD, MANAGER  
**Entry:** UI/API action TBD

### Inputs
notificationId; editable field set TBD among title/message/type/linkUrl/isRead. id, recipientId, and createdAt are system/identity fields unless explicitly approved otherwise.

### Candidate flow
1. Actor selects edit.
2. Server checks Update permission and scope.
3. UI submits approved fields.
4. Server validates and updates.
5. Server returns updated record.

### Rules
- RBAC does not define editing a notification after recipient delivery.
- No creator field exists for creator-based editing.
- Whether staff may alter read state is NOTIF-D06; editable content is NOTIF-D04/D07.

### Failure
Unauthorized/out-of-scope: deny. Invalid field: reject. Stale update: Module 18. Missing record: safe not-found.

### Acceptance
Server authorization; only approved fields editable; failure leaves data unchanged.

### QA
NOTIF-QA-033–039.

## NOTIF-F006 — Delete notification

**Status:** RBAC Delete source-confirmed only for SUPER_ADMIN; semantics TBD  
**Actor:** SUPER_ADMIN  
**Entry:** UI/API action not mapped

### Candidate flow
1. Super Admin requests delete.
2. Server verifies global permission and target.
3. Approved deletion semantics applied.
4. Result returned.

### Rules / gaps
- Hard delete vs soft delete/archive is unspecified.
- Schema has no deletedAt/archive field.
- Effects on recipient history/delivery and retention are NOTIF-D17.

### Failure
Other roles denied; missing target safe not-found; concurrency per Module 18.

### Acceptance
Non-Super Admin cannot delete; no invented archive fields/cascade behavior; retention/audit rules applied only when approved.

### QA
NOTIF-QA-040–044.

## NOTIF-F007 — Open notification link

**Status:** linkUrl schema field source-confirmed; click-through behavior derived  
**Actor:** Authorized notification viewer  
**Entry:** linkUrl link/action

### Candidate flow
1. Actor selects link.
2. System validates URL against approved policy.
3. Browser/app navigates.
4. Target module rechecks its own authorization.

### Security
Notification link is not an access token. Destination authorization remains authoritative. Open redirect and unsafe schemes need explicit policy. Internal-only vs external links is NOTIF-D18.

### Failure
Missing/invalid link: controlled behavior/message TBD. Target denied/unavailable: destination/shared policy.

### Acceptance
No unsafe URL execution; destination permission is enforced; linkUrl cannot bypass auth.

### QA
NOTIF-QA-045–050.

## NOTIF-F008 — Student Dashboard notification area

**Status:** Notification entity mapping in v2 source-confirmed; Dashboard UX decision-required  
**Actor:** STUDENT  
**Entry:** Student Dashboard

### Candidate flow
1. Student opens Dashboard.
2. Dashboard loads approved sources.
3. If notification preview/count is approved, query own records only.
4. Display approved notification summary.
5. Link opens approved notification route/detail.

### Gaps
- v2 mapping lists Notification on Student Dashboard; original mapping omits it.
- Whether dashboard shows list, unread count, badge, or preview is unknown.
- Refresh/polling/realtime behavior unknown.

### Acceptance
Do not treat entity mapping as proof of widget UI. If approved, dashboard data is Student-own only.

### QA
NOTIF-QA-051–055.

## NOTIF-F009 — Event-driven notification creation/delivery

**Status:** Not source-confirmed; candidate system capability  
**Actor:** System / event-producing module  
**Entry:** Event trigger not documented

### Source facts
NotificationType enum contains SYSTEM, ASSIGNMENT, GRADE, CONTEST, BILLING, TICKET. Business Flow documents do not define notification events.

### Candidate flow (not approved)
1. Approved source event occurs.
2. Approved event catalog resolves type and recipient.
3. Approved content template produces title/message/link.
4. Notification record created.
5. External channel delivery, if approved, occurs separately.
6. Delivery/retry/dedup outcomes follow approved shared rules.

### Gaps
Trigger catalog, recipient selection, type mapping, channels, templates, status/retry, preference, idempotency, event durability, email security, WebSocket authorization and per-channel delivery persistence are NOTIF-D01–D03, D11–D16 and D24–D28.

### Acceptance
No trigger or recipient guessed; row creation is not treated as delivery success; delivery rules must be approved before enabling.

### QA
NOTIF-QA-056–065.

## Proposed target handling and cross-module event contracts

**Status: RECOMMENDATION ONLY.** The following defaults are proposed to make the target design coherent. They are not approved and must remain blocked by their linked NOTIF decisions.

| Target function | Proposed direction | Decision gate |
|---|---|---|
| NOTIF-F001 / F002 | Persist and query a recipient's own inbox by default; derive recipient identity from the authenticated principal. Staff views of other recipients' records require a separately defined scoped-management purpose and permission. Use pagination and server-side scope before filters. | D05/D08/D19 |
| NOTIF-F003 | Only the recipient can mark their own record read/unread; server persists `isRead`; repeated same-state requests are idempotent. Initial state recommendation: `false`. Mark-all-read is a batch operation scoped to the authenticated recipient. Opening detail does not auto-read unless approved. | D06/D10/D16 |
| NOTIF-F004 | System producers create one Notification record per resolved recipient after the owning domain transaction commits. Do not accept client-set `createdAt`, `isRead`, or arbitrary `recipientId`. Staff broadcast/announcement creation should be a separately modeled workflow because the current Notification schema has no sender or audience fields. | D01–D04/D07/D19 |
| NOTIF-F005 | Treat recipientId and createdAt as immutable. Do not edit content after it is visible to a recipient; use a correction/new record if approved. Staff C/U scope needs an explicit model before implementation. | D04/D05/D07/D19 |
| NOTIF-F006 | Do not make user-facing clear/dismiss equivalent to hard delete. Preserve history by default; define a separate dismissal/retention state if required. Keep Super Admin delete permission distinct from recipient read/dismiss behavior. | D17/D19/D20 |
| NOTIF-F007 | Prefer internal relative paths for product navigation. If external HTTPS links are allowed, use an explicit allowlist and safe-open behavior; never allow javascript/data schemes or rely on linkUrl for authorization. | D09/D18 |
| NOTIF-F008 | If the Dashboard widget is approved, query only the Student's own persisted notifications. Widget preview, unread count, badge, refresh and sorting remain separate UX decisions. | D14/D22 |
| NOTIF-F009 | Treat each owner module's committed durable event as the source trigger. Resolve recipient(s) from trusted event/relationship data, map to NotificationType, produce a minimal content projection, persist the in-app record, and dispatch approved email/realtime channels independently. Dedupe by stable event/recipient/channel identity; never let delivery failure alter the source domain result. | D01–D03/D11–D16/D24–D28 |

### Candidate owner-module trigger domains (not a final event catalogue)

| Producer / module | Current main-document evidence | Target notification treatment proposed |
|---|---|---|
| Module 1 — Identity | Approved AUTH decisions: email verification/recovery; security email on password changed/reset and email changed. Reset token is single-use and expires after 15 minutes. MFA toggle, role/scope change, suspension/deactivation and new-device emails are explicitly not selected in current target decisions. | Module 13 delivers only the security-email triggers approved by Module 1; reset token stays in the secure email workflow and never in an inbox Notification or log. |
| Module 3 — Billing | SUB baseline owns configurable renewal reminders, invoice/payment and failure events. | Module 3 owns event/recipient business facts; Module 13 applies approved College contact, channel and template rules. Do not copy legacy dunning cron/address behavior. |
| Module 4 — Course & Enrollment | Adopted target workflow has automatic Section enrollment, staff withdrawal and HOD-confirmed Course completion; it emits Module 13/15 events. | Consume safe committed events; Course completion remains HOD-verified and must not be inferred from solving Problems/Labs/Assessments/Contests. |
| Modules 6–7 — Submission/Evaluation | Module 7 publishes SubmissionGraded only after final result persistence; Module 6/7 contracts define safe result summaries and owner modules. | Notify only after committed final result; no code, Judge0 token, hidden testcase details or infrastructure-only outcome leak. |
| Module 8 — Labs | Adopted LAB contract emits safe committed Lab events; Module 13 owns recipient, channel, reminder and delivery configuration. | Consume only configured events; Student-specific Lab scores/feedback require recipient and privacy checks. |
| Module 9 — Assessments | Module 9 documents safe committed events and assigns notification delivery to Module 13, but its decisions were prematurely marked adopted before the whole-module review. | Treat Assessment triggers as pending cross-module validation; do not use them as approved target event contracts yet. |
| Module 11 — Contests | CON-D22 leaves notification and audit events/recipients/retention open. | No fixed 24h/1h reminders, event state push or Contest result notification until Module 11 decisions are reconciled. |
| Module 12 — Certificates | CERT-D21 keeps audit/notification requirements open; no target issuance trigger is approved. | No automatic CertificateIssued notification until Module 12 eligibility/issuance and Module 13 delivery decisions are approved. |
| Module 14 — Support Tickets | NotificationType.TICKET exists, but no Ticket event/recipient/channel mapping is source-defined. | No automatic ticket create/comment/assignment/status notification until Module 14 and NOTIF rules are approved. |

### Proposed type mapping for review (not approved)

| NotificationType enum | Candidate event family | Evidence status |
|---|---|---|
| SYSTEM | Approved identity/security and platform notices | Recommendation; AUTH restricts which security emails are selected |
| ASSIGNMENT | Approved Lab/assignment publication or assignment to recipient | Recommendation; event semantics and recipients need Module 8/NOTIF decisions |
| GRADE | Committed SubmissionGraded or approved score change | Recommendation aligned to Module 7 result contract |
| CONTEST | Approved Contest registration/state/result event | Open under CON-D22 and NOTIF-D01–D03 |
| BILLING | Invoice, renewal, payment success/failure | Module 3 owns business triggers; Module 13 channel/template remains open |
| TICKET | Ticket event | Not source-defined; Module 14 + NOTIF decisions required |

### Channel and delivery boundary

- **In-app record:** `Notification` is the target durable recipient record; the source schema supplies only id, recipientId, title, message, type, linkUrl, isRead and createdAt.
- **Email:** separate transactional delivery channel. Do not infer an EmailDelivery model, SES provider, BullMQ queue, template names, retries or send status from old code. Module 1 security rules and Module 3 billing recipient sources remain authoritative.
- **Real-time:** optional delivery/presentation channel. A WebSocket frame is not the persisted notification and must not be the only way for a Student to discover an event. Realtime authorization and reconnect behavior remain decision-bound.
- **Durability:** proposed sequence is domain transaction → durable outbox event → idempotent Notification creation → independently retriable channel delivery. Module 18 owns retry/replay; Module 15 owns audit. Exact transaction/outbox and event envelope remain technical decisions.
- **Privacy:** notification title/message/link and email variables must use allowlisted summary data; no reset tokens, access/refresh tokens, code submissions, Judge0 tokens, hidden testcase inputs/outputs, or unredacted billing/payment payloads.

## Decision log
## Decision log

| ID | Question / conflict | Impact | Status |
|---|---|---|---|
| NOTIF-D01 | Which business events trigger notifications? | No event automation | Open |
| NOTIF-D02 | Recipient rules by event/type (student, staff, group, section, College, etc.) | Privacy/targeting | Open |
| NOTIF-D03 | Event-to-NotificationType mapping | Inconsistent types | Open |
| NOTIF-D04 | Meaning of staff C/U: create/send, edit, manage own inbox, or other? | RBAC ambiguity | Open |
| NOTIF-D05 | Staff list/detail scope: own inbox, scoped users' notifications, or created items? | Visibility | Open |
| NOTIF-D06 | Who may mark read/unread? Can staff mutate recipient read state? | State permission | Open |
| NOTIF-D07 | Who creates notifications: user, event service, or both? | Workflow | Open |
| NOTIF-D08 | List/detail UX, filters, ordering, pagination, empty states | UX/API | Open |
| NOTIF-D09 | Row-click vs linkUrl navigation contract | Navigation | Open |
| NOTIF-D10 | Auto-read on open, mark unread, and initial isRead default | Read state | Open |
| NOTIF-D11 | Delivery channels: in-app, email, SMS, push, etc. | Architecture | Open |
| NOTIF-D12 | Templates, localization, content format, message/title rules | Content | Open |
| NOTIF-D13 | Delivery status, retry, failure tracking, partial delivery | Operations | Open |
| NOTIF-D14 | Student Dashboard notification widget; v2 mapping differs from original | Scope conflict | Open |
| NOTIF-D15 | Preferences, opt-out, digest/channel settings | Product/data model | Open |
| NOTIF-D16 | Duplicate events, idempotency, deduplication | Data/UX | Open |
| NOTIF-D17 | Retention, archive, deletion, dismiss/read history | Lifecycle | Open |
| NOTIF-D18 | URL allowlist, internal/external links, open redirect prevention | Security | Open |
| NOTIF-D19 | Need sender/creator, delivery status, readAt, channel, deletedAt fields? | Schema | Open |
| NOTIF-D20 | Super Admin page vs API/resource permission | Navigation mismatch | Open |
| NOTIF-D21 | Audit events for create/update/delete/read/delivery | Module 15 | Open |
| NOTIF-D22 | Real-time vs polling/manual refresh/unread count | UX/technical | Open |
| NOTIF-D23 | Access after User/College/Section deactivation | Lifecycle | Open |
| NOTIF-D24 | Does each approved source event create an in-app Notification, send email, emit realtime, or fan out to several channels? Define per-event channel policy and precedence when a recipient/channel is unavailable. | Channel architecture and duplicate/missing notices | Open |
| NOTIF-D25 | What is the canonical event envelope and durability boundary between source-module transaction, outbox, Notification creation, and external delivery? Define event identity/version and producer ownership with Modules 15/18. | Lost/duplicate notices, cross-module coupling and recovery | Open |
| NOTIF-D26 | Define the transactional email contract: trusted recipient address source, template identifier/version, variable allowlist, locale, sender/reply-to, secret handling, and security-email separation under Module 1. | Account security, deliverability, privacy and support | Open |
| NOTIF-D27 | If realtime delivery is selected, define authenticated private-channel identity, tenant/group/event scope, payload minimization, reconnect/replay behavior and authoritative fallback to persisted inbox. | Cross-user data leakage and missed events | Open |
| NOTIF-D28 | Should per-channel delivery attempts/status/retry history be stored in a separate model, embedded in Notification, or omitted? Define retention and correlation without overloading the recipient inbox record. | Schema, observability, partial delivery and retention | Open |

