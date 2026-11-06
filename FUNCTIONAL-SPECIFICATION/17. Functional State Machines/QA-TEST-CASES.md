# Module 17 — Functional State Machines QA Test Cases

**Document ID:** STATE-MACHINES-QA-001 | **Version:** 0.1 | **Status:** QA draft; decision-dependent expected results  
**Related:** [Overview](./README.md) · [Function specifications](./FUNCTION-SPECIFICATIONS.md)

> State names and current behavior are prior forensic findings, not approved target behavior. Tests that depend on open decisions remain blocked until the decision log is resolved. Authorization, invalid-transition, idempotency, and integrity tests remain mandatory.

## Fixtures
- Tenants in pending/trial/active/suspended/expired/cancelled states with different scopes.
- Events in draft/published/registration_open/closed/live/grading/completed/cancelled; registration records in each reported state.
- Submission verdict fixtures and Judge0 callback/rejudge event versions.
- Paths/rooms with required/optional/skipped/completed states and certificate eligibility.
- Users/invitations in active/pending/suspended/deleted and pending/accepted/expired/revoked.
- Subscription/invoice/payment states and signed/forged/replayed provider webhooks.
- Content rooms and assessments/problems across lifecycle states and revisions.
- Certificates issued/revoked; launch phase A/B/C; outbox pending/dispatched/failed/dead_letter.

## Cases

| Test ID | Function | Scenario | Preconditions / steps | Expected result | Trace |
| FSM-QA-001 | FSM-F001 | Allowed tenant transition | Tenant state and authorized actor → Request allowed state change | Transition succeeds and effects match policy | FSM-F001 |
| FSM-QA-002 | FSM-F001 | Illegal tenant transition | Tenant in active state → Request disallowed target state | Rejected; state unchanged | FSM-F001 |
| FSM-QA-003 | FSM-F001 | Tenant transition by non-admin | Scoped user → Request global transition | Denied | FSM-F001 |
| FSM-QA-004 | FSM-F001 | Suspend tenant | Tenant active → Authorized suspend | Tenant status and access/session effects follow policy | FSM-F001 |
| FSM-QA-005 | FSM-F001 | Cancelled tenant recovery | Tenant cancelled → Request reactivate/restore | Allowed only if approved; otherwise explicit denial | FSM-F001 |
| FSM-QA-006 | FSM-F001 | Purge tenant | Tenant has retained records → Request purge | Retention/legal hold/dependency guard applied | FSM-F001 |
| FSM-QA-007 | FSM-F001 | Concurrent tenant state updates | Two admins transition same tenant → Submit concurrently | Conflict detected; no lost update | FSM-F001 |
| FSM-QA-008 | FSM-F001 | Tenant downstream access | Tenant suspended/expired → Attempt protected operation | Access enforcement matches approved tenant state | FSM-F001 |
| FSM-QA-009 | FSM-F002 | Publish event | Event draft configured → Publish event | State changes only after guard/authorization | FSM-F002 |
| FSM-QA-010 | FSM-F002 | Scheduled registration open | Event scheduled for opening → Watcher evaluates due event | Registration opens once at approved time | FSM-F002 |
| FSM-QA-011 | FSM-F002 | Watcher delayed/replayed | Cron delayed or repeated → Process event transitions | No duplicate side effects; timing semantics followed | FSM-F002 |
| FSM-QA-012 | FSM-F002 | Registration close time boundary | Event at close time → Watcher executes around boundary | UTC/timezone and boundary rule applied | FSM-F002 |
| FSM-QA-013 | FSM-F002 | Start event | Registration closed/event start reached → Authorized/manual or watcher transition | Event becomes live per approved guard | FSM-F002 |
| FSM-QA-014 | FSM-F002 | Cancel live event | Event live with participants → Admin cancels | Cancellation effects/refunds follow approved policy | FSM-F002 |
| FSM-QA-015 | FSM-F002 | Complete event | Grading pending → Final grading completion | Completion occurs only after approved result finalization | FSM-F002 |
| FSM-QA-016 | FSM-F002 | Watcher overlapping runs | Concurrent cron instances → Process same event | Lock/idempotency prevents double transition | FSM-F002 |
| FSM-QA-017 | FSM-F003 | Register eligible participant | Registration open, capacity available → Submit registration | Registered/pending payment state as policy requires | FSM-F003 |
| FSM-QA-018 | FSM-F003 | Registration closed | Event closed → Submit registration | Rejected; no registration record improperly created | FSM-F003 |
| FSM-QA-019 | FSM-F003 | Capacity race | Last seat, multiple applicants → Register concurrently | Capacity not exceeded; waitlist policy followed | FSM-F003 |
| FSM-QA-020 | FSM-F003 | Duplicate registration | Participant already registered → Register again | Idempotent/duplicate response; no second active registration | FSM-F003 |
| FSM-QA-021 | FSM-F003 | Payment pending timeout | Registration pending_payment → Timeout job runs | Expired/cancelled state per approved policy | FSM-F003 |
| FSM-QA-022 | FSM-F003 | Payment webhook replay | Same provider event repeated → Process twice | No duplicate payment/registration transition | FSM-F003 |
| FSM-QA-023 | FSM-F003 | Cancel registration | Eligible registration → Cancel | Cancellation deadline/refund rules applied | FSM-F003 |
| FSM-QA-024 | FSM-F003 | Refund failure | Cancellation triggers provider refund → Provider refund fails | Registration/refund states reflect unresolved refund; not falsely refunded | FSM-F003 |
| FSM-QA-025 | FSM-F004 | Submission accepted by API | Valid code submission → Submit code | Submission begins in pending state | FSM-F004 |
| FSM-QA-026 | FSM-F004 | Judge0 returns accepted | Pending execution → Worker receives accepted verdict | Trusted worker updates verdict/result | FSM-F004 |
| FSM-QA-027 | FSM-F004 | Student attempts to set verdict | Student request includes verdict=accepted → Submit/modify request | Client cannot set authoritative verdict | FSM-F004 |
| FSM-QA-028 | FSM-F004 | Judge0 system error | Execution service fails → Worker receives system_error | Retry/rejudge policy followed; terminal state not assumed prematurely | FSM-F004 |
| FSM-QA-029 | FSM-F004 | Duplicate worker callback | Same evaluation result twice → Process callback twice | Idempotent result; no duplicate side effect | FSM-F004 |
| FSM-QA-030 | FSM-F004 | Rejudge older result arrives late | Rejudge version 2 pending; version 1 callback delayed → Process callback | Stale result cannot overwrite latest approved evaluation | FSM-F004 |
| FSM-QA-031 | FSM-F004 | Partial verdict/score | Some tests pass → Evaluate | Partial status and score follow Module 7 contract | FSM-F004 |
| FSM-QA-032 | FSM-F004 | Verdict terminality | Submission already final → Attempt new verdict | Only rejudge/correction policy can change result | FSM-F004 |
| FSM-QA-033 | FSM-F005 | Start room | Room not_started → Student starts | State becomes in_progress | FSM-F005 |
| FSM-QA-034 | FSM-F005 | Complete room | Room eligible to complete → Student completes | Completed state and progress event applied | FSM-F005 |
| FSM-QA-035 | FSM-F005 | Skip room | Room is required → Student skips | Skipped state is distinct from completed; credential policy respected | FSM-F005 |
| FSM-QA-036 | FSM-F005 | Skip all rooms | Path required steps all skipped → Aggregator processes | No unauthorized certificate eligibility | FSM-F005 |
| FSM-QA-037 | FSM-F005 | Room access forged | Room not in learner path/tenant → Student mutates progress | Forbidden; no state update | FSM-F005 |
| FSM-QA-038 | FSM-F005 | Duplicate progress event | Same event delivered twice → Aggregator consumes | Aggregate not double-counted | FSM-F005 |
| FSM-QA-039 | FSM-F005 | Path zero steps | Empty path → Aggregate progress | Defined zero-step behavior | FSM-F005 |
| FSM-QA-040 | FSM-F005 | Path step removed after progress | Progress exists; content changed → Reaggregate | Approved invalidation/reconciliation applies | FSM-F005 |
| FSM-QA-041 | FSM-F006 | Accept valid invitation | Pending invitation, valid token → Invitee accepts | Invitation accepted; account activation follows policy | FSM-F006 |
| FSM-QA-042 | FSM-F006 | Accept expired invitation | Expired token → Accept invite | Rejected; account not activated from expired token | FSM-F006 |
| FSM-QA-043 | FSM-F006 | Accept revoked invitation | Revoked invite → Accept | Rejected | FSM-F006 |
| FSM-QA-044 | FSM-F006 | Duplicate invite acceptance | Invite already accepted → Accept again | Idempotent/safe response; no duplicate membership | FSM-F006 |
| FSM-QA-045 | FSM-F006 | Suspend user | Active user → Authorized admin suspends | Status and session access follow Module 1 policy | FSM-F006 |
| FSM-QA-046 | FSM-F006 | Deleted user login | User deleted → Attempt login | Denied | FSM-F006 |
| FSM-QA-047 | FSM-F006 | Reinvite existing user | User/invitation already exists → Send invitation | Conflict/reinvite policy applied | FSM-F006 |
| FSM-QA-048 | FSM-F006 | Unauthorized user status change | Non-admin actor → Attempt suspend/delete | Denied | FSM-F006 |
| FSM-QA-049 | FSM-F007 | Payment success webhook | Valid signed payment event → Process callback | Payment/subscription/invoice updated consistently | FSM-F007 |
| FSM-QA-050 | FSM-F007 | Invalid webhook signature | Forged event → Submit webhook | Rejected; no billing state mutation | FSM-F007 |
| FSM-QA-051 | FSM-F007 | Duplicate webhook | Provider event already handled → Replay event | Idempotent processing | FSM-F007 |
| FSM-QA-052 | FSM-F007 | Out-of-order webhook | Payment failure then success callbacks arrive out of order → Process callbacks | Provider event ordering/version policy prevents stale state | FSM-F007 |
| FSM-QA-053 | FSM-F007 | Subscription past_due | Payment failure → Apply dunning/status transition | Entitlement/grace behavior follows policy | FSM-F007 |
| FSM-QA-054 | FSM-F007 | Subscription cancellation | Active subscription → Cancel subscription | Effective date/renewal/entitlement behavior follows policy | FSM-F007 |
| FSM-QA-055 | FSM-F007 | Refund status | Payment refund requested/failed/succeeded → Process provider result | Payment and invoice statuses consistent; no premature refunded state | FSM-F007 |
| FSM-QA-056 | FSM-F007 | Provider subscription update absent | Provider changes subscription → Check synchronization | Webhook/poll/reconciliation requirement is defined and gaps observable | FSM-F007 |
| FSM-QA-057 | FSM-F008 | Submit room for review | Draft room → Author submits | State becomes in_review if authorized | FSM-F008 |
| FSM-QA-058 | FSM-F008 | Unauthorized reviewer approval | User lacks reviewer permission → Approve content | Denied | FSM-F008 |
| FSM-QA-059 | FSM-F008 | Approve reviewed revision | Room in review → Reviewer approves | Approved state tied to reviewed revision | FSM-F008 |
| FSM-QA-060 | FSM-F008 | Publish unapproved content | Draft/review state → Attempt publish | Denied unless approved transition policy explicitly allows | FSM-F008 |
| FSM-QA-061 | FSM-F008 | Edit published content | Published room → Author modifies | Revision/freeze policy applied; active references protected | FSM-F008 |
| FSM-QA-062 | FSM-F008 | Archive room used by active path | Published room referenced by learners → Archive | Dependency warning/guard and learner impact policy applied | FSM-F008 |
| FSM-QA-063 | FSM-F008 | Concurrent reviewer decisions | Two reviewers act on same revision → Approve/reject concurrently | Conflict detection; one canonical result | FSM-F008 |
| FSM-QA-064 | FSM-F008 | Stale approval | Revision changes after review → Publish older approved revision | Stale revision blocked or version policy applied | FSM-F008 |
| FSM-QA-065 | FSM-F009 | Publish assessment | Assessment draft valid → Publish | Authorized transition; configuration validation enforced | FSM-F009 |
| FSM-QA-066 | FSM-F009 | Activate assessment at schedule | Published assessment due → Scheduler executes | Active state at approved time/timezone | FSM-F009 |
| FSM-QA-067 | FSM-F009 | Archive assessment with active attempt | Learner attempt in progress → Archive | Attempt preservation/termination policy applied | FSM-F009 |
| FSM-QA-068 | FSM-F009 | Edit published problem | Problem published → Modify problem | Version/freeze policy applied | FSM-F009 |
| FSM-QA-069 | FSM-F009 | Publish invalid assessment | Required settings missing → Publish | Validation errors; state unchanged | FSM-F009 |
| FSM-QA-070 | FSM-F009 | Archive problem referenced in assessment | Problem in use → Archive | Dependency/active assessment policy applied | FSM-F009 |
| FSM-QA-071 | FSM-F009 | Concurrent assessment status update | Admin and scheduler act together → Transition concurrently | Conflict handling prevents lost state | FSM-F009 |
| FSM-QA-072 | FSM-F009 | Withdraw assessment | Published/active assessment → Admin withdraws | Learner notification and attempt/result policy applied | FSM-F009 |
| FSM-QA-073 | FSM-F010 | Issue eligible certificate | Eligibility validated by Module 12 → Issue credential | Certificate issued once with immutable identity | FSM-F010 |
| FSM-QA-074 | FSM-F010 | Issue certificate from duplicate event | Eligibility event replayed → Process twice | No duplicate credential unless explicit reissue workflow | FSM-F010 |
| FSM-QA-075 | FSM-F010 | Attempt certificate without eligibility | Ineligible learner → Request issue | Denied | FSM-F010 |
| FSM-QA-076 | FSM-F010 | Revoke certificate | Issued certificate → Authorized revoke with reason | Status revoked and verification reflects revoked state | FSM-F010 |
| FSM-QA-077 | FSM-F010 | Verify revoked certificate | Revoked credential → Public verification request | Not reported as valid | FSM-F010 |
| FSM-QA-078 | FSM-F010 | Reissue/replacement certificate | Credential revoked/lost → Request reissue | Policy and lineage/reason applied | FSM-F010 |
| FSM-QA-079 | FSM-F010 | Unknown certificate revocation | Invalid certificate ID → Revoke | Safe not-found; no mutation | FSM-F010 |
| FSM-QA-080 | FSM-F010 | Certificate verification cache stale | Certificate revoked; cache old → Verify | Cache invalidated or freshness guarantees applied | FSM-F010 |
| FSM-QA-081 | FSM-F011 | Advance launch phase | Current phase A; authorized operator → Confirm target B | Transition accepted per guard | FSM-F011 |
| FSM-QA-082 | FSM-F011 | Wrong confirmation code | Operator submits wrong code → Request phase advance | Rejected; phase unchanged | FSM-F011 |
| FSM-QA-083 | FSM-F011 | Unauthorized launch phase update | Unprivileged actor → Request advance | Denied | FSM-F011 |
| FSM-QA-084 | FSM-F011 | Concurrent phase advances | Two operators act → Advance phase simultaneously | Conflict/lock prevents invalid phase skip | FSM-F011 |
| FSM-QA-085 | FSM-F011 | Rollback launch phase | Current phase B/C → Request rollback | Approved rollback policy applied; no undocumented irreversible assumption | FSM-F011 |
| FSM-QA-086 | FSM-F011 | Phase update partial failure | Config update fails midway → Advance phase | Atomicity/recovery; no mixed phase configuration | FSM-F011 |
| FSM-QA-087 | FSM-F011 | Confirmation code expires/reused | Code old or already used → Submit | Validity/replay policy enforced | FSM-F011 |
| FSM-QA-088 | FSM-F011 | Audit launch transition | Authorized phase transition → Inspect audit | Actor, phase, time and outcome audited safely | FSM-F011 |
| FSM-QA-089 | FSM-F012 | Dispatch pending outbox event | Valid pending event → Dispatcher publishes | State moves to dispatched on ack | FSM-F012 |
| FSM-QA-090 | FSM-F012 | Queue publish failure | Pending event; queue unavailable → Dispatch | Failure recorded and retry policy applied | FSM-F012 |
| FSM-QA-091 | FSM-F012 | Publish succeeds but ack lost | Event published, dispatcher timeout → Retry dispatch | Duplicate delivery safely handled | FSM-F012 |
| FSM-QA-092 | FSM-F012 | Consumer duplicate delivery | Same message delivered twice → Consumer processes | Consumer idempotency prevents repeated side effects | FSM-F012 |
| FSM-QA-093 | FSM-F012 | Poison event | Invalid payload repeatedly fails → Process until threshold | Dead-letter handling and alert follows policy | FSM-F012 |
| FSM-QA-094 | FSM-F012 | Dead-letter replay | Dead-letter item reviewed → Authorized operator replays | Replay permission and duplicate safety enforced | FSM-F012 |
| FSM-QA-095 | FSM-F012 | Outbox lease/claim expiry | Dispatcher worker crashes after claim → Another worker claims | No permanently stuck event; concurrency guard works | FSM-F012 |
| FSM-QA-096 | FSM-F012 | Event ordering/version | Related events delivered out of order → Consume | Ordering/version contract protects dependent state | FSM-F012 |

## Coverage / gates
- **Total:** 96 sequential tests, FSM-QA-001–096.
- Coverage: tenant, event, registration, submission, learning progress, user/invitation, billing/payment, content approval, assessment/problem, certificates, launch phase, and outbox dispatch.
- FSM-D01–D22 and related module decisions must be incorporated before target acceptance.
- Module 18 must define retry, ordering, concurrency, dead-letter, and partial-failure expectations.
