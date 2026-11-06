# Module 13 — Notifications: Function-Level Specifications

**Document ID:** NOTIF-FS-FUNCTIONS-001 | **Version:** 0.1 | **Status:** Detailed draft; decisions open  
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
Trigger catalog, recipient selection, type mapping, channels, templates, status/retry, preference, and idempotency are NOTIF-D01–D03, D11–D16.

### Acceptance
No trigger or recipient guessed; row creation is not treated as delivery success; delivery rules must be approved before enabling.

### QA
NOTIF-QA-056–065.

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

