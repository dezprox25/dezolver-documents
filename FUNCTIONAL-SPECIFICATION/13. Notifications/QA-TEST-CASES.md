# Module 13 — Notifications QA Test Cases

**Version:** 0.2 | **Status:** Detailed QA draft; decision-dependent tests are blocked until policy approval  
**Related:** [Overview](./README.md) · [Function specifications](./FUNCTION-SPECIFICATIONS.md)

> For decision-dependent tests, the expected result is to apply the approved rule. This document does not select the product rule.

## Test fixtures
- SUPER_ADMIN global.
- Finance-A assigned College A.
- HOD-A College A / Department A; HOD-B Department B.
- Manager-A Section A; Manager-B Section B.
- Student-A, Student-B, and Student-C in separate user/scope contexts.
- Notification-A addressed to Student-A; Notification-B addressed to Student-B; Notification-Staff addressed to a staff recipient if supported; notification records with each NotificationType.
- Test records with read/unread state and safe/unsafe/missing linkUrl.

## QA test cases

| Test ID | Function | Scenario / steps | Expected result | Trace |
|---|---|---|---|---|
| NOTIF-QA-001 | F001 | Student-A opens Notifications | Only recipientId=Student-A records returned | RBAC own |
| NOTIF-QA-002 | F001 | Student-A submits Student-B recipientId | Server ignores/rejects forged identity; no B records | RBAC |
| NOTIF-QA-003 | F001 | HOD-A lists notifications under authorized department | Results restricted to approved HOD scope | RBAC / D05 |
| NOTIF-QA-004 | F001 | HOD-A requests record from Department B | Deny/safe not-found; no data leak | RBAC scope |
| NOTIF-QA-005 | F001 | Manager-A requests Section B notification | Deny/safe not-found | RBAC scope |
| NOTIF-QA-006 | F001 | Finance-A requests notification outside College A | Deny/safe not-found | RBAC scope |
| NOTIF-QA-007 | F001 | Super Admin accesses authorized global notification records | Allow within global scope | RBAC |
| NOTIF-QA-008 | F001 | Unauthorized role/API requests notification list | Deny; no records returned | RBAC |
| NOTIF-QA-009 | F001 | No notifications exist for recipient | Approved empty state; no unrelated records | UX TBD |
| NOTIF-QA-010 | F002 | Student-A opens Notification-A | Own notification details returned | Ownership |
| NOTIF-QA-011 | F002 | Student-A opens Notification-B by guessed ID | Deny/safe not-found | Ownership |
| NOTIF-QA-012 | F002 | Staff user opens detail outside permitted scope | Deny/safe not-found | D05 |
| NOTIF-QA-013 | F002 | Notification opens while read state false | Auto-read only if approved; otherwise unchanged | D10 |
| NOTIF-QA-014 | F002 | Notification does not exist | Safe not-found | Shared error |
| NOTIF-QA-015 | F003 | Student-A marks own notification read | Allowed only under approved permission; persisted isRead=true | D06 |
| NOTIF-QA-016 | F003 | Student-A marks Student-B notification read | Denied; B record unchanged | Ownership |
| NOTIF-QA-017 | F003 | Student-A marks own notification unread | Apply approved mark-unread policy | D06/D10 |
| NOTIF-QA-018 | F003 | Staff marks recipient's read state | Apply approved staff policy; no assumed authority | D06 |
| NOTIF-QA-019 | F003 | Client sends isRead as string or null | Reject malformed value | Schema validation |
| NOTIF-QA-020 | F003 | Request repeats same isRead value | Apply approved idempotency behavior | Module 18 |
| NOTIF-QA-021 | F003 | Notification deleted before read update | Safe not-found; no mutation | Concurrency |
| NOTIF-QA-022 | F003 | Two read-state updates concurrently | Shared conflict policy; final state consistent | Module 18 |
| NOTIF-QA-023 | F004 | Super Admin creates valid notification for permitted target | Created only under approved creation model | D07 |
| NOTIF-QA-024 | F004 | Finance Team creates notification for College-A recipient | Apply approved create/target policy within scope | RBAC/D04 |
| NOTIF-QA-025 | F004 | HOD creates notification for Department-A target | Apply approved create/target policy within scope | RBAC/D04 |
| NOTIF-QA-026 | F004 | Manager targets a Student outside permitted Section/Group | Deny | RBAC scope |
| NOTIF-QA-027 | F004 | Student attempts to create notification | Deny under supplied RBAC | RBAC |
| NOTIF-QA-028 | F004 | Client forges recipientId outside actor scope | Deny; no notification row | Security |
| NOTIF-QA-029 | F004 | title exceeds 255 characters | Reject; no record | Schema |
| NOTIF-QA-030 | F004 | type not in NotificationType enum | Reject | Schema |
| NOTIF-QA-031 | F004 | linkUrl exceeds 1024 chars | Reject if URL submitted | Schema |
| NOTIF-QA-032 | F004 | Event creation retries after timeout | Apply approved idempotency/dedup policy; no accidental duplicate | D16 |
| NOTIF-QA-033 | F005 | Super Admin updates allowed notification fields | Apply approved update contract | RBAC/D04 |
| NOTIF-QA-034 | F005 | Finance/HOD/Manager update out-of-scope notification | Deny | RBAC |
| NOTIF-QA-035 | F005 | Student attempts content update | Deny under source RBAC (V OWN only) | RBAC |
| NOTIF-QA-036 | F005 | Staff attempts to change recipientId | Reject/ignore unless explicitly approved; identity cannot be silently changed | Security/D19 |
| NOTIF-QA-037 | F005 | Update title/message with invalid data | Reject; record unchanged | Validation |
| NOTIF-QA-038 | F005 | Notification already read by recipient then content updated | Apply approved edit-after-read policy | D04/D07 |
| NOTIF-QA-039 | F005 | Concurrent update | Shared stale-update/conflict policy | Module 18 |
| NOTIF-QA-040 | F006 | Super Admin deletes a notification | Apply approved deletion semantics | RBAC/D17 |
| NOTIF-QA-041 | F006 | HOD attempts delete | Deny; RBAC does not grant HOD D | RBAC |
| NOTIF-QA-042 | F006 | Finance/Manager/Student attempts delete | Deny | RBAC |
| NOTIF-QA-043 | F006 | Delete request repeats | Apply shared idempotent/not-found behavior | Module 18 |
| NOTIF-QA-044 | F006 | Delete conflicts with concurrent read/update | Apply approved conflict and retention policy | D17/Module 18 |
| NOTIF-QA-045 | F007 | Notification has safe internal link | Navigate only if link policy approved; target performs auth | D18 |
| NOTIF-QA-046 | F007 | linkUrl is javascript/data scheme or unsafe URL | Block; do not execute | Security/D18 |
| NOTIF-QA-047 | F007 | Link points to foreign tenant resource | Destination denies access | RBAC |
| NOTIF-QA-048 | F007 | linkUrl absent | Apply approved no-link UI | D09 |
| NOTIF-QA-049 | F007 | External URL is supplied | Apply allowlist/external-link policy | D18 |
| NOTIF-QA-050 | F007 | Link points to expired/removed resource | Destination/shared failure behavior | Module 18 |
| NOTIF-QA-051 | F008 | Student-A opens Dashboard | Notification display only if approved; data must be own-only | D14 |
| NOTIF-QA-052 | F008 | Student dashboard notification widget displays Student-B item | Must not occur | Ownership |
| NOTIF-QA-053 | F008 | Dashboard unread badge/count requested | Apply approved widget/count rule | D14 |
| NOTIF-QA-054 | F008 | Dashboard refreshed after notification changes | Refresh/realtime behavior per approved policy | D22 |
| NOTIF-QA-055 | F008 | Compare original and v2 page mapping for Dashboard | Conflict recorded; do not treat entity map alone as UI behavior | Source conflict |
| NOTIF-QA-056 | F009 | Assignment event occurs | Create notification only if event catalog approves trigger and recipient | D01/D02 |
| NOTIF-QA-057 | F009 | Grade event occurs | Type/recipient/content follow approved mapping | D01–D03 |
| NOTIF-QA-058 | F009 | Contest event occurs | Type/recipient/content follow approved mapping | D01–D03 |
| NOTIF-QA-059 | F009 | Billing event occurs | Recipient/Finance scope and type follow approved mapping | D01–D03 |
| NOTIF-QA-060 | F009 | Ticket event occurs | Ticket event/recipient mapping follows approved rule | D01–D03 |
| NOTIF-QA-061 | F009 | Event maps to invalid enum type | Reject/handle according to approved producer contract | Schema |
| NOTIF-QA-062 | F009 | Notification row created but email/push delivery fails | Row creation and delivery status separated; apply approved channel/retry rule | D11/D13 |
| NOTIF-QA-063 | F009 | External delivery retries | Apply approved retry and idempotency; no uncontrolled duplicate | D13/D16 |
| NOTIF-QA-064 | F009 | User opted out of channel, if preferences are approved | Respect approved preference behavior | D15 |
| NOTIF-QA-065 | F009 | Delivery contains sensitive academic/financial information | Apply approved privacy/content/channel policy | D12/D15 |

| NOTIF-QA-066 | F001/F002 | Student receives a Notification, reloads the page and signs in on a second device | Target list/detail reads persisted recipient records; do not reproduce legacy RAM-only loss on reload/device change | Target schema / LEGACY-SPEC-REPORTED |
| NOTIF-QA-067 | F001/F003 | Recipient marks a record read on Device A and opens inbox on Device B | Both clients reflect persisted isRead after refresh/reconciliation; realtime is not the source of truth | D10/D22/D27 |
| NOTIF-QA-068 | F001 | Inbox contains more than 50 records | Do not impose legacy Zustand 50-item ring-buffer eviction as retention; apply approved pagination/retention policy | LEGACY-SPEC-REPORTED / D08/D17 |
| NOTIF-QA-069 | F001/F002 | Client assumes the old UI's absence of REST endpoints means target inbox is only socket-driven | Do not infer target API contract from legacy missing endpoints; implement only an approved persistent read interface | LEGACY-SPEC-REPORTED / target schema / D08 |
| NOTIF-QA-070 | F001/F004/F009 | Socket client subscribes to another user's `user:{userId}` channel or submits a forged recipientId | Server binds personal channel/recipient to authenticated principal; deny cross-user delivery and record access | RBAC / D02/D05/D27 |
| NOTIF-QA-071 | F004/F009 | Same durable source event is replayed or delivered twice to the same recipient/channel | Apply approved eventId/recipient/channel deduplication; no duplicate inbox rows or duplicate external side effects | D16/D25/D28 / Module 18 |
| NOTIF-QA-072 | F009 | Submission grading event arrives before Module 7 final result transaction commits | No grade notification until canonical SubmissionGraded is committed; operational/incomplete evaluation produces no false grade notice | Module 6/7 / D01/D25 |
| NOTIF-QA-073 | F004/F009 | Notification or email payload is built from hidden testcase inputs/outputs, source code, Judge0 token or raw diagnostics | Remove sensitive content; deliver only approved safe result summary to authorized recipient | Module 6/7 / D12/D26 |
| NOTIF-QA-074 | F009 | Worker emits SubmissionGraded while an older consumer listens for SubmissionAccepted | Target consumers use one canonical owner-approved committed event contract; do not copy the legacy event-name mismatch | LEGACY-SPEC-REPORTED / Module 6/7 / D25 |
| NOTIF-QA-075 | F009 | Lab event is emitted before Lab/LabScore transaction is committed or is replayed | Consume only safe committed Lab events; deduplicate and use Module 8's authoritative Lab score/history | Module 8 LAB-D24/D35 / D25 |
| NOTIF-QA-076 | F009 | Course enrollment/completion notice is triggered by a Problem/Lab/Contest completion rather than HOD-confirmed Course completion | Respect Module 4's adopted Course completion rule; no automatic Course completion notice from activity completion alone | Module 4 / D01/D02 |
| NOTIF-QA-077 | F009 | Assessment notice is sent based on a Module 9 event before its decisions are reviewed in the all-modules pass | Treat Assessment trigger as decision-blocked until cross-module approval; no guessed event/recipient/channel | Module 9 workflow caveat / D01–D03 |
| NOTIF-QA-078 | F009 | Contest reminder is sent 24 hours or 1 hour before start solely because legacy cron did so | No fixed legacy reminder schedule; use only Module 11-approved trigger and recipient policy | LEGACY-SPEC-REPORTED / CON-D22 / D01/D11 |
| NOTIF-QA-079 | F009 | Contest start/end/extend WebSocket event is delivered to a user without approved participation/scope check | Resolve recipient/authorized Contest scope from Module 11; no legacy event-room broadcast is assumed | LEGACY-SPEC-REPORTED / CON-D22 / D02/D27 |
| NOTIF-QA-080 | F009 | Dunning email destination is populated from College/tenant UUID | Resolve a verified email address from Module 3's approved College billing-contact source; reject UUID/non-email destination | LEGACY-SPEC-REPORTED / Module 3 / D26 |
| NOTIF-QA-081 | F009 | Password reset request emits outbox event but no email consumer processes it | Wire the approved Module 1 password-recovery event to delivery; reset token remains single-use/15-minute and is not stored in Notification or logs | AUTH decisions / LEGACY-SPEC-REPORTED / D25–D26 |
| NOTIF-QA-082 | F009 | Email notification is sent for MFA toggle, role/scope change, suspension/deactivation or new-device login | Do not send those email events under current Module 1 approved security-notification policy; send only password changed/reset and email changed where specified | Module 1 approved decisions / D01/D26 |
| NOTIF-QA-083 | F009 | Target email delivery is assumed to use SES managed templates or old inline fallback solely because legacy worker supported both | Email provider/template mode follows explicit target configuration; legacy SES environment switch is not a target default | LEGACY-SPEC-REPORTED / D11/D12/D26 |
| NOTIF-QA-084 | F009 | Unknown email template falls back to a raw JSON dump containing variables/secrets | Unknown template fails safely or uses an approved safe fallback; never log or email raw unfiltered job payload | LEGACY-SPEC-REPORTED / AUTH / D12/D26 |
| NOTIF-QA-085 | F009 | Producer enqueues work to legacy `notification-fanout` queue with no registered worker | Do not treat legacy queue as implemented; only dispatch through an approved durable consumer/worker contract | LEGACY-SPEC-REPORTED / D25/D28 / Module 18 |
| NOTIF-QA-086 | F009 | Certificate issuance is assumed to emit `CertificateIssued` email/socket notice from the old Credentials pipeline | Wait for Module 12 eligibility/issuance and certificate-notification decisions; CERT-D21 remains open | LEGACY-SPEC-REPORTED / Module 12 / D01/D24 |
| NOTIF-QA-087 | F009 | WebSocket delivery succeeds but the user reloads or misses the frame | Persist/retrieve the approved Notification record independently; realtime is an optional presentation channel, not durable inbox evidence | LEGACY-SPEC-REPORTED / target schema / D22/D24/D27 |
| NOTIF-QA-088 | F007 | linkUrl uses javascript/data scheme, an unapproved external host, or redirects outside allowlist | Block unsafe schemes/open redirects; destination rechecks its own authorization | D18 |
| NOTIF-QA-089 | F004/F009 | Staff producer broadcasts to an entire College/Department/Section/Group using client-supplied audience IDs | Re-resolve recipient set from trusted membership/scope in owner module; reject forged or out-of-scope fanout | RBAC / D02/D04/D05/D25 |
| NOTIF-QA-090 | F009 | Notification row is persisted but email provider fails (or email succeeds but inbox persistence fails) | Track independent channel outcomes according to approved consistency policy; do not report both channels delivered or fail the source domain transaction silently | D13/D24/D25/D28 / Module 18 |

## Coverage summary
90 tests cover:
- recipient/role/scope access;
- notification detail and read state;
- creation, update, delete permissions;
- schema validation;
- URL/link safety and destination authorization;
- Student Dashboard source discrepancy;
- event triggers, recipients, delivery channels, retry, and privacy decisions;
- target persistence vs legacy memory-only behavior, email/SES, WebSocket, outbox/fanout, event-owner alignment, identity security, billing recipient and cross-channel partial-failure regression.

NOTIF-QA-066–090 are target-vs-legacy and cross-module regression cases. Test cases are defined only; they have not been executed. Tests tied to unresolved decisions must remain blocked until the relevant policy is approved.
