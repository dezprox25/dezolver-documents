# Module 14 — Support Tickets QA Test Cases

**Version:** 0.1 | **Status:** Detailed QA draft; decision-dependent tests blocked until policy is approved  
**Related:** [Overview](./README.md) · [Function specifications](./FUNCTION-SPECIFICATIONS.md)

> Tests dependent on open decisions use the approved decision as their expected result; they do not resolve the product policy.

## Test fixtures
- SUPER_ADMIN global.
- Finance user assigned College A.
- HOD-A College A/Department A; HOD-B Department B.
- Manager-A Section A; Manager-B Section B.
- Student-A, Student-B, Student-C.
- Ticket-A requester Student-A/College A; Ticket-B requester Student-B/College B; scoped tickets assigned/unassigned.
- Comments on tickets with isInternal=true and false.
- Ticket fixtures for every TicketStatus, TicketPriority, TicketCategory.

## QA cases

| Test ID | Function | Scenario / steps | Expected result | Trace |
|---|---|---|---|---|
| TICKET-QA-001 | F001 | Student-A creates valid ticket for self | Create succeeds under Student C permission | RBAC |
| TICKET-QA-002 | F001 | Student-A submits requesterId=Student-B | Deny/reject; no ticket for B | Ownership |
| TICKET-QA-003 | F001 | Student-A submits College B ticket | Deny; College scope not forged | Scope |
| TICKET-QA-004 | F001 | Staff creates ticket within authorized scope | Apply approved staff-create policy | RBAC/D03 |
| TICKET-QA-005 | F001 | Staff creates ticket outside scope | Deny; no record | RBAC |
| TICKET-QA-006 | F001 | Title exceeds 255 characters | Reject | Schema |
| TICKET-QA-007 | F001 | Category invalid enum | Reject | Schema |
| TICKET-QA-008 | F001 | Priority invalid enum | Reject | Schema |
| TICKET-QA-009 | F001 | Status invalid enum or client tries to force initial state | Reject/apply approved initial status rule | D02 |
| TICKET-QA-010 | F001 | Create request times out and client retries | Apply approved idempotency policy; no accidental duplicate | D23 |
| TICKET-QA-011 | F002 | Student-A opens My Tickets | Only requesterId=Student-A records returned | RBAC own |
| TICKET-QA-012 | F002 | Student-A forges requesterId=Student-B | No B tickets returned | Ownership |
| TICKET-QA-013 | F002 | Student-A accesses Ticket-B by guessed ID | Deny/safe not-found | Ownership |
| TICKET-QA-014 | F002 | Student has no tickets | Approved empty state | UX TBD |
| TICKET-QA-015 | F002 | Student requests out-of-College ticket | Deny/safe not-found | Scope |
| TICKET-QA-016 | F002 | Unauthenticated user requests My Tickets | Deny/auth flow Module 1 | Authentication |
| TICKET-QA-017 | F003 | Student-A opens Ticket-A detail | Own authorized fields returned | Ownership |
| TICKET-QA-018 | F003 | Student-A opens Ticket-B detail | Deny/safe not-found | Ownership |
| TICKET-QA-019 | F003 | Manager-A opens Section-B ticket | Deny/safe not-found | RBAC |
| TICKET-QA-020 | F003 | HOD-A opens Department-B ticket | Deny/safe not-found | RBAC |
| TICKET-QA-021 | F003 | Requester tries to read internal comment | Internal comment not disclosed under approved policy | D07 |
| TICKET-QA-022 | F003 | Ticket detail includes comments | Only authorized/visible comments returned | D07 |
| TICKET-QA-023 | F004 | Student-A comments on own ticket | Apply approved requester-comment permission | D04 |
| TICKET-QA-024 | F004 | Student-A comments on Ticket-B | Deny; no comment created | Ownership |
| TICKET-QA-025 | F004 | Client forges authorId | Server derives authenticated author or rejects | Security |
| TICKET-QA-026 | F004 | Staff adds permitted public reply | Apply approved staff comment rule | D07 |
| TICKET-QA-027 | F004 | Staff adds internal comment | Apply approved role/visibility policy | D07 |
| TICKET-QA-028 | F004 | Student sets isInternal=true | Deny unless explicitly approved | D07 |
| TICKET-QA-029 | F004 | Empty/oversized message | Apply approved message validation | D07 |
| TICKET-QA-030 | F004 | Comment API response includes internal comment to requester | Must not leak internal content | Privacy |
| TICKET-QA-031 | F005 | Super Admin opens ticket queue | Global access per RBAC; page contract TBD | RBAC |
| TICKET-QA-032 | F005 | Finance user lists College-A tickets | Restricted to authorized College scope | RBAC |
| TICKET-QA-033 | F005 | Finance user requests College-B ticket | Deny/safe not-found | Scope |
| TICKET-QA-034 | F005 | HOD-A lists department tickets | Restrict to College A/Department A under approved queue model | RBAC |
| TICKET-QA-035 | F005 | HOD-A requests Department B ticket | Deny | Scope |
| TICKET-QA-036 | F005 | Manager-A lists section/group tickets | Restrict to permitted operational scope | RBAC |
| TICKET-QA-037 | F005 | Staff queue filters/assignment scope | Apply approved queue policy, not inferred defaults | D05 |
| TICKET-QA-038 | F006 | Authorized staff assigns eligible user | Assignment succeeds if policy allows | D06 |
| TICKET-QA-039 | F006 | Student attempts assignment | Deny | RBAC |
| TICKET-QA-040 | F006 | Staff assigns inactive user | Reject | D06 |
| TICKET-QA-041 | F006 | Staff assigns user outside College/department/section policy | Reject | D06 |
| TICKET-QA-042 | F006 | Client supplies invalid assignedToId | Reject; no invalid assignee | Validation |
| TICKET-QA-043 | F006 | Unauthorized staff reassigns ticket | Deny | D06 |
| TICKET-QA-044 | F006 | Two agents assign concurrently | Apply Module 18 conflict policy | Concurrency |
| TICKET-QA-045 | F007 | Authorized actor changes to valid status transition | Apply approved state machine | D01 |
| TICKET-QA-046 | F007 | Actor attempts unsupported status transition | Reject; unchanged | D01 |
| TICKET-QA-047 | F007 | Student changes status without permission | Deny | D04 |
| TICKET-QA-048 | F007 | Status outside enum | Reject | Schema |
| TICKET-QA-049 | F007 | RESOLVED status applied | resolvedAt behavior follows approved state rule | D10 |
| TICKET-QA-050 | F007 | ESCALATED status applied | escalatedAt behavior follows approved state rule | D12 |
| TICKET-QA-051 | F007 | Status update with stale ticket version | Apply Module 18 concurrency behavior | Concurrency |
| TICKET-QA-052 | F007 | Status update fails | No partial status/timestamp mutation | Atomicity |
| TICKET-QA-053 | F008 | Create ticket with each valid category | All enum values accepted | Schema |
| TICKET-QA-054 | F008 | Create ticket with each valid priority | All enum values accepted | Schema |
| TICKET-QA-055 | F008 | Client sends invalid category | Reject | Schema |
| TICKET-QA-056 | F008 | Client sends invalid priority | Reject | Schema |
| TICKET-QA-057 | F008 | Student changes priority after creation | Apply approved permission policy | D08 |
| TICKET-QA-058 | F008 | Staff changes category/priority outside scope | Deny | RBAC |
| TICKET-QA-059 | F009 | Staff requests more information | Apply approved transition and response workflow | D09 |
| TICKET-QA-060 | F009 | Unauthorized user requests more info | Deny | RBAC |
| TICKET-QA-061 | F009 | Requester responds to MORE_INFO_REQUESTED ticket | Apply approved comment/status behavior | D09 |
| TICKET-QA-062 | F009 | Requester response attempts unauthorized status transition | Deny status mutation | D01 |
| TICKET-QA-063 | F009 | Notification delivery fails after request persisted | Ticket and delivery outcome remain distinct | D16 |
| TICKET-QA-064 | F009 | Request-more-info operation is retried | Apply idempotency/duplicate policy | D23 |
| TICKET-QA-065 | F010 | Authorized actor resolves ticket | Apply approved transition, notes, timestamp policy | D10 |
| TICKET-QA-066 | F010 | Resolve without required resolution notes | Apply approved requiredness rule | D10 |
| TICKET-QA-067 | F010 | Actor closes ticket directly from disallowed state | Reject | D01/D10 |
| TICKET-QA-068 | F010 | Requester closes ticket | Apply approved requester permission | D04 |
| TICKET-QA-069 | F010 | Requester reopens resolved/closed ticket | Apply approved actor/time-window policy | D11 |
| TICKET-QA-070 | F010 | Reopen outside allowed window | Apply approved policy; no guessed limit | D11 |
| TICKET-QA-071 | F010 | Reopen a ticket repeatedly | Apply approved reopen count/policy | D11 |
| TICKET-QA-072 | F010 | Concurrent resolve/reopen actions | Apply canonical state/conflict behavior | Module 17/18 |
| TICKET-QA-073 | F011 | Authorized staff escalates ticket | Apply approved escalation policy | D12 |
| TICKET-QA-074 | F011 | Student escalates ticket | Apply approved requester escalation rule | D12 |
| TICKET-QA-075 | F011 | Escalation from invalid state | Reject | D01 |
| TICKET-QA-076 | F011 | Escalation without required reason | Apply approved note requiredness | D12 |
| TICKET-QA-077 | F011 | Escalation target unavailable | Apply approved fallback/failure policy | D12 |
| TICKET-QA-078 | F011 | Automatic SLA escalation threshold | No automatic escalation unless approved | D12 |
| TICKET-QA-079 | F011 | Escalation notification fails | Ticket escalation and delivery status handled separately | D16 |
| TICKET-QA-080 | F012 | HOD-A opens Ticket Verification page | Access limited to HOD role/scope; action semantics TBD | RBAC |
| TICKET-QA-081 | F012 | Manager opens HOD Ticket Verification | Deny unless explicitly authorized | RBAC |
| TICKET-QA-082 | F012 | HOD-A attempts verify Department-B ticket | Deny | Scope |
| TICKET-QA-083 | F012 | HOD attempts verification before policy is defined | No unapproved mutation/verification behavior | D13 |
| TICKET-QA-084 | F012 | Verification status/action requested | Apply approved schema/workflow contract | D13 |
| TICKET-QA-085 | F012 | Verification attempt should be audited | Apply approved Module 15 policy | D17 |
| TICKET-QA-086 | F013 | Staff creates internal comment | Apply approved staff permission | D07 |
| TICKET-QA-087 | F013 | Requester views ticket with internal comments | Internal comments must not leak | Privacy |
| TICKET-QA-088 | F013 | Unauthorized role requests internal comment API | Deny/omit per approved contract | D07 |
| TICKET-QA-089 | F013 | Internal comment is changed to public by unauthorized actor | Reject | Security |
| TICKET-QA-090 | F013 | Internal comment authorId is forged | Server derives author | Security |
| TICKET-QA-091 | F013 | resolutionNotes and escalationNotes are accessed | Apply distinct approved visibility policy | D20 |
| TICKET-QA-092 | F013 | Comment edit/delete requested | Apply approved immutable/edit policy; no assumption | D20 |

## Coverage summary
92 test cases cover creation, requester ownership, staff scope, ticket detail, comments/internal confidentiality, assignment, enum validation, status changes, request-more-information, resolve/close/reopen, escalation, HOD verification, and concurrent/retry cases.

Tests dependent on open decisions must remain blocked until the decision is approved.
