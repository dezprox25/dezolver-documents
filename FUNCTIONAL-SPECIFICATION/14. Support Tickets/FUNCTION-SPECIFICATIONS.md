# Module 14 — Support Tickets: Function-Level Specifications

**Document ID:** TICKET-FS-FUNCTIONS-001 | **Version:** 0.1 | **Status:** Detailed draft; decision-dependent behavior open  
**Standard:** [Functional Specification Standard](../FUNCTIONAL-SPECIFICATION-STANDARD.md)  
**Parent:** [Module overview](./README.md) · [QA cases](./QA-TEST-CASES.md)

> Workflow behavior not explicit in the supplied sources is candidate behavior linked to decisions. It is not approved target behavior.

## Shared permissions and scope

| Role | Source permission | Scope |
|---|---|---|
| SUPER_ADMIN | V/C/U/D | Global |
| FINANCE_TEAM | V/C/U* | Assigned College |
| HOD | V/C/U* | Assigned College + Department |
| MANAGER | V/C/U* | College + Department + permitted Section/Group |
| STUDENT | C / V OWN | Own records |

RBAC is resource-level, not a ticket operation matrix. Backend checks role, action, scope, and ownership. Ticket and comment permissions must not be assumed identical.

## TICKET-F001 — Submit/create support ticket
**Status:** Student Create and staff resource Create source-confirmed; form details TBD. **Actors:** Student and scoped staff. **Entry:** Help & Support/API.

**Purpose:** Create a ticket linked to requester and College.

**Preconditions:** Authenticated actor; Create permission; valid requester/College context.

**Inputs**
| Field | Type | Required / validation |
|---|---|---|
| requesterId | varchar | Schema field; must be trusted identity/authorized requester |
| collegeId | varchar | Schema field; server validates College scope |
| title | varchar(255) | Requiredness TBD; max 255 |
| description | text | Requiredness/format/size TBD |
| category | TicketCategory | TECHNICAL, BILLING, ACADEMIC, ACCOUNT; required/default TBD |
| priority | TicketPriority | LOW, MEDIUM, HIGH, URGENT; required/default TBD |
| assignedToId | varchar | Optionality and assignment policy TBD; client cannot select arbitrary assignee |
| status | TicketStatus | Initial value not specified |
| timestamps | timestamp | System-managed policy TBD |

**Candidate main flow:** actor opens Help & Support → enters approved form fields → submits → server checks permission, identity, College scope, field constraints and enum values → server sets system-owned fields → record created → approved success response/ticket reference displayed.

**Rules/gaps:** Student Create is source-confirmed; staff creation is scope-restricted. Default status/category/priority, ticket number, attachments, notifications, and post-submit route are TBD.

**Failure:** invalid/out-of-scope identity: deny/no record; title >255: reject; invalid enum: reject; duplicate/retry behavior: Module 18 TBD; persistence failure: no false success.

**Acceptance:** valid requester/College association; no forged requester; enums validated; invalid requests create no ticket; successful response identifies ticket. **QA:** TICKET-QA-001–010.

## TICKET-F002 — List/view requester’s own tickets
**Status:** Student View Own and My Tickets page source-confirmed. **Actor:** Student/requester. **Entry:** My Tickets.

**Preconditions:** authenticated requester with V OWN. **Inputs:** session-derived requesterId; optional query filters/pagination only if approved.

**Flow:** requester opens page → server derives requester identity → queries own tickets → returns permitted summaries → UI renders list. Sort, filters, columns, pagination, and empty state TBD.

**Rules:** Students cannot supply requesterId to widen access. Staff list behavior is handled by F005. **Failures:** expired session per Module 1; unauthorized role deny; empty state TBD; service error Module 18.

**Acceptance:** own tickets only; forged ID cannot expose another user’s ticket; no unrelated data in empty/error results. **QA:** TICKET-QA-011–016.

## TICKET-F003 — View ticket detail and history
**Status:** Ticket entity/page mapping source-confirmed; detail/history behavior derived. **Actors:** requester and scoped staff. **Entry:** list item/direct API.

**Preconditions:** ticket exists and actor has view permission. **Flow:** select ticket → server loads ticket/comments → checks role/scope/ownership and comment visibility → returns approved fields → UI displays detail/history.

**Rules/gaps:** Student only own ticket; staff visibility model TBD; internal comments must not leak; history ordering/editing TBD.

**Failures:** missing ID safe not-found; out-of-scope deny/safe not-found; internal comment unauthorized deny/omit per approved rule; API failure Module 18.

**Acceptance:** guessed ID cannot bypass scope; requester only sees permitted records; internal confidentiality is enforced. **QA:** TICKET-QA-017–022.

## TICKET-F004 — Add ticket comment/reply
**Status:** TicketComment schema and resource Create permission source-confirmed; reply permissions TBD. **Actors:** requester/staff according to decision. **Entry:** ticket detail.

**Inputs:** ticketId required; authorId server-derived; message text (requiredness/size TBD); isInternal boolean (policy TBD); createdAt system timestamp.

**Candidate flow:** open authorized ticket → enter message/visibility if permitted → server checks ticket/comment permission → derives author → validates content and internal visibility → persists comment → returns result → history refresh.

**Rules:** schema fields include ticketId, authorId, message, isInternal, createdAt. Who can comment, edit/delete, choose internal/public, and receive notifications are TBD.

**Failures:** ticket out of scope deny; forged author reject/ignore; internal comment by unauthorized actor deny; invalid message reject; no comment on failure.

**Acceptance:** correct ticket/actor relation; no impersonation; internal content protected. **QA:** TICKET-QA-023–030.

## TICKET-F005 — Staff ticket queue/list and review
**Status:** Staff Support/Ticket View permission and pages source-confirmed; queue model TBD. **Actors:** Super Admin, Finance, HOD, Manager. **Entry:** Support Tickets / Help & Support / Ticket Verification.

**Flow:** staff opens page → server checks role/scope → applies approved queue/assignment policy → returns permitted summaries → UI displays ticket list and actions.

**Scope:** Super Admin global; Finance College; HOD College+Department; Manager College+Department+permitted Section/Group. Assigned-only vs scope-wide vs own-created queue is undecided.

**Failures:** out-of-scope deny; no permission deny; query failure Module 18. **Acceptance:** list obeys role scope; queue population is policy-based, not guessed; direct API checks match UI. **QA:** TICKET-QA-031–037.

## TICKET-F006 — Assign/reassign ticket
**Status:** assignedToId source-confirmed; workflow TBD. **Actor:** staff with approved assignment permission. **Entry:** queue/detail.

**Inputs:** ticketId and assignee User ID; assignment note only if approved.

**Candidate flow:** actor chooses assignment → server checks permission/scope → validates ticket and eligible assignee (role, active status, org scope) → persists assignment → updates timestamp under policy → returns current assignee.

**Gaps:** eligible roles, self-assignment, unassign/reassign, transfer, assignment history, and notifications TBD. Do not accept requester-selected arbitrary assignee by assumption.

**Failures:** invalid/inactive/out-of-scope assignee reject; unauthorized actor deny; concurrency Module 18. **Acceptance:** only authorized assignment; valid assignee; result persisted and returned. **QA:** TICKET-QA-038–044.

## TICKET-F007 — Update ticket status
**Status:** TicketStatus enum source-confirmed; transitions TBD. **Actor:** authorized staff; requester rights TBD. **Entry:** detail/queue.

**Inputs:** ticketId, status enum, authenticated actor.

**Candidate flow:** choose status → server loads current ticket → checks role/scope and canonical Module 17 transition → updates status and timestamp fields only according to approved rule → audit/notification if approved → returns persisted state.

**Enum:** SUBMITTED, RESOLVED, ESCALATED, MORE_INFO_REQUESTED, CLOSED, REOPENED. Enum does not define transitions. resolvedAt/escalatedAt semantics TBD.

**Failures:** invalid enum or transition reject/no state change; unauthorized deny; stale state Module 18. **Acceptance:** server validates transitions; timestamps consistent; invalid changes do not mutate. **QA:** TICKET-QA-045–052.

## TICKET-F008 — Set/update category and priority
**Status:** enum values source-confirmed; edit rights/defaults TBD. **Actors:** requester on creation and/or staff. **Entry:** form/detail.

**Inputs:** category = TECHNICAL/BILLING/ACADEMIC/ACCOUNT; priority = LOW/MEDIUM/HIGH/URGENT.

**Flow:** submit field values → check permission/scope → validate enum → apply approved default/edit rule → return persisted values.

**Gaps:** who initially chooses priority, who can override, priority criteria, category routing, and change history are TBD.

**Failures:** invalid enum reject; unauthorized update deny; out-of-scope ticket deny. **Acceptance:** allowed actors only; values are enum-valid; no default invented. **QA:** TICKET-QA-053–058.

## TICKET-F009 — Request more information and respond
**Status:** MORE_INFO_REQUESTED enum source-confirmed; process TBD. **Actors:** staff/requester under approved rights. **Entry:** ticket detail.

**Candidate flow:** staff requests info and supplies text → server validates action/transition → status updates if approved → requester receives information through approved channel → requester responds through approved comment mechanism → next state follows Module 17.

**Gaps:** request text storage, role authority, automatic status change, requester response action, next status, and notification policy are unresolved.

**Failures:** unauthorized or invalid transition deny; notification failure handled separately from ticket persistence. **Acceptance:** no guessed state/channel; reply linked to ticket. **QA:** TICKET-QA-059–064.

## TICKET-F010 — Resolve, close, or reopen ticket
**Status:** enum values and resolvedAt source-confirmed; workflow TBD. **Actors:** authorized actor; requester rights TBD. **Entry:** detail/queue.

**Candidate flow:** choose resolve/close/reopen → server checks actor and transition → resolution notes/timestamp handled per policy → state persisted → approved audit/notification applied → result shown.

**Gaps:** RESOLVED vs CLOSED semantics; notes requiredness/visibility; reopen window/limit/eligibility; timestamp set/clear behavior.

**Failures:** disallowed transition reject; unauthorized deny; missing ticket safe not-found; concurrency Module 18. **Acceptance:** canonical state machine enforced; timestamp policy consistent; unauthorized closure/reopen denied. **QA:** TICKET-QA-065–072.

## TICKET-F011 — Escalate ticket
**Status:** ESCALATED enum, escalatedAt and escalationNotes source-confirmed; policy TBD. **Actor:** authorized staff/requester rights TBD.

**Candidate flow:** invoke escalation → check actor/scope/current state → capture reason → update status/timestamp per approved lifecycle → determine escalation target/notification only from approved policy → display result.

**Gaps:** destination, trigger (manual/automatic), SLA timer, required reason, de-escalation, and notification are unknown.

**Failures:** unauthorized deny; invalid transition reject; missing required target reject; delivery failure separated from ticket record. **Acceptance:** no target/SLA guessed; status/time follow policy. **QA:** TICKET-QA-073–079.

## TICKET-F012 — HOD Ticket Verification
**Status:** page name source-confirmed; meaning and operation TBD. **Actor:** HOD. **Entry:** Ticket Verification.

**Candidate flow:** HOD opens page → server checks HOD and College+Department scope → loads tickets eligible under approved policy → HOD reviews fields → performs verification only if action and persistence are approved.

**Gaps:** what verification means (legitimacy, metadata, resolution, or other), whether read-only, verification outcomes, schema fields, and audit requirements unknown.

**Failures:** out-of-scope deny; non-HOD deny unless permission is approved; no defined verification contract means action remains unimplemented. **Acceptance:** HOD scope enforced; no verification behavior invented. **QA:** TICKET-QA-080–085.

## TICKET-F013 — Internal comments and notes
**Status:** TicketComment.isInternal and ticket notes fields source-confirmed; visibility/edit policy TBD. **Actors:** authorized staff. **Entry:** ticket detail.

**Candidate flow:** staff opens authorized ticket → server checks internal-comment permissions → actor adds internal comment/note → server derives author and validates ticket/message/visibility → stores comment → response only includes comments for authorized viewers.

**Gaps:** default isInternal, permitted authors/viewers, requester visibility, comment edit/delete, and whether resolutionNotes/escalationNotes differ from comments.

**Failures:** requester attempts internal note: deny unless approved; forged isInternal: reject/ignore per policy; unauthorized read: deny. **Acceptance:** internal content protected; actor is session-derived; notes fields not conflated. **QA:** TICKET-QA-086–092.

## Decision log

| ID | Question / conflict | Impact | Status |
|---|---|---|---|
| TICKET-D01 | Full lifecycle and transition graph? | Workflow | Open |
| TICKET-D02 | Initial status/default category/priority; required fields? | Create contract | Open |
| TICKET-D03 | Ticket creation eligibility by role/category? | Access | Open |
| TICKET-D04 | Can requester comment, edit, close, reopen, or cancel? | Student workflow | Open |
| TICKET-D05 | Staff visibility: own, assigned, queue, or all within scope? | Privacy | Open |
| TICKET-D06 | Assignment rights, eligible assignee roles, unassign/reassign? | Routing | Open |
| TICKET-D07 | TicketComment permissions and public/internal visibility? | Confidentiality | Open |
| TICKET-D08 | Category/priority defaults and who can change them? | Triage | Open |
| TICKET-D09 | MORE_INFO_REQUESTED flow and next status? | Response process | Open |
| TICKET-D10 | RESOLVED vs CLOSED semantics, notes and timestamps? | Lifecycle | Open |
| TICKET-D11 | Reopen eligibility, window, limit, and transitions? | Lifecycle | Open |
| TICKET-D12 | Escalation destination, reason, trigger, SLA, timer? | Operations | Open |
| TICKET-D13 | HOD Ticket Verification meaning, action, fields? | Page/schema | Open |
| TICKET-D14 | Help & Support vs My Tickets page behavior? | UX | Open |
| TICKET-D15 | Attachments, limits, scanning, access? | Security/data | Open |
| TICKET-D16 | Ticket notification events, recipients, channels? | Module 13 | Open |
| TICKET-D17 | Ticket/comment audit and retention policy? | Module 15 | Open |
| TICKET-D18 | SLA/support hours/priority meaning/response targets? | Operations | Open |
| TICKET-D19 | Ticket deletion/archive/retention? | Privacy/lifecycle | Open |
| TICKET-D20 | Ticket/comment edit/delete and immutable history? | Integrity | Open |
| TICKET-D21 | Finance/HOD/Manager category and scope specifics? | RBAC | Open |
| TICKET-D22 | Schema needs ticket number, verification fields, comment update time, assignment history? | Schema | Open |
| TICKET-D23 | Retry/idempotency, duplicates, concurrent updates? | Reliability | Open |
