# Module 9 — Assessments QA Test Cases

**Version:** 0.2 — companion to Functional Specification Standard v1.0  
**Status:** Draft; policy-dependent expected results remain decision-blocked  
**Related:** [Module 9 README](./README.md) · [Function Specifications](./FUNCTION-SPECIFICATIONS.md)

> Use QA cases as coverage and review prompts. Where a product decision is open, the expected result is marked **Decision-blocked**; it must not be guessed and treated as approved behavior.

## Test identities and data
- SuperAdmin: global administrative role.
- HOD-A: College-A / Department-A.
- Manager-A: College-A / Department-A / Section-A / permitted Group-A.
- Manager-B: a different section/group scope.
- Student-A: eligible Student.
- Student-B: different Student, foreign owner.
- Assessment-A: in HOD-A/Manager-A scope.
- Assessment-B: out of scope or foreign College.
- Problem-A: linked to Assessment-A.
- Problem-B: not linked to Assessment-A.
- Attempt-A: hypothetical active attempt; attempt storage model is not source-confirmed.
- Finance-A: Finance Team assigned to College-A.

## QA scenarios

| QA ID | Function | Scenario / steps | Expected result | Status / trace |
|---|---|---|---|---|
| ASMT-QA-001 | F001 | SuperAdmin lists Assessments | Global records available subject to approved UI permission | RBAC |
| ASMT-QA-002 | F001 | HOD-A lists Assessments | Results constrained to College-A + Department-A | RBAC |
| ASMT-QA-003 | F001 | Manager-A lists Assessments | Results constrained to permitted Section/Group | RBAC |
| ASMT-QA-004 | F001 | Manager-A queries Assessment-B by ID | Deny/safe not-found; no foreign record leakage | RBAC |
| ASMT-QA-005 | F001 | Finance-A opens Assessment API/page | Deny per RBAC Assessment matrix | RBAC |
| ASMT-QA-006 | F001 | Client changes departmentId filter | Cannot broaden scope | Security |
| ASMT-QA-007 | F001 | Empty scoped Assessment list | Safe empty state | UX TBD |
| ASMT-QA-008 | F002 | HOD creates valid Assessment in Department-A | Record saved with authorized scope/creator | Schema/RBAC |
| ASMT-QA-009 | F002 | Manager creates in assigned Section/Group scope | Allowed only if approved Assessment create scope supports it | Decision-blocked |
| ASMT-QA-010 | F002 | Student attempts create | Denied | RBAC |
| ASMT-QA-011 | F002 | Finance Team attempts create | Denied | RBAC |
| ASMT-QA-012 | F002 | Client supplies foreign College ID | Server rejects/overrides invalid scope | Security |
| ASMT-QA-013 | F002 | Unsupported Assessment type | Reject enum value; CODING is current source enum | Schema |
| ASMT-QA-014 | F002 | Title length exceeds 255 | Reject if input violates varchar constraint | Schema |
| ASMT-QA-015 | F002 | Missing fields required by publish/create policy | Block according to approved required field list | Decision-blocked |
| ASMT-QA-016 | F002 | Create request times out and client retries | Duplicate prevention/idempotency behavior TBD | Decision-blocked |
| ASMT-QA-017 | F003 | Authorized HOD opens Assessment-A | View permitted configuration | RBAC |
| ASMT-QA-018 | F003 | Manager-A edits in-scope Assessment-A | Allow only fields/actions authorized | RBAC |
| ASMT-QA-019 | F003 | Manager-A edits Assessment-B | Deny | RBAC |
| ASMT-QA-020 | F003 | Student directly calls update endpoint | Deny | RBAC |
| ASMT-QA-021 | F003 | Update status to unsupported enum | Reject | Schema |
| ASMT-QA-022 | F003 | Edit published Assessment after Student starts | Content mutation policy unresolved | Decision-blocked |
| ASMT-QA-023 | F003 | Concurrent edits by two staff | Conflict/no silent overwrite behavior TBD | Decision-blocked |
| ASMT-QA-024 | F004 | Add Problem-A to Assessment-A | Association created with valid IDs/order | Schema |
| ASMT-QA-025 | F004 | Add Problem-B outside permitted scope | Deny | RBAC |
| ASMT-QA-026 | F004 | Add duplicate Problem-A | Duplicate constraint/UX policy TBD | Decision-blocked |
| ASMT-QA-027 | F004 | Reorder multiple AssessmentProblems | Display order matches persisted orderIndex | Schema |
| ASMT-QA-028 | F004 | Remove Problem after attempt started | Mutation/snapshot policy TBD | Decision-blocked |
| ASMT-QA-029 | F004 | Attempt to set Problem weight | Weight field absent from source schema; feature blocked pending decision | Schema gap |
| ASMT-QA-030 | F004 | Assessment includes multiple Problems | Multi-Problem relationship supported conceptually; attempt/result flow TBD | Schema |
| ASMT-QA-031 | F005 | Assign Assessment-A to Group-A | Allowed if target assignment model approved | Decision-blocked |
| ASMT-QA-032 | F005 | Assign foreign Group-B | Deny | RBAC |
| ASMT-QA-033 | F005 | Configure multi-Group assignment | Support not confirmed by one groupId field | Decision-blocked |
| ASMT-QA-034 | F005 | Configure individual Student exception | Not source-confirmed; feature decision required | Decision-blocked |
| ASMT-QA-035 | F005 | Student joins assigned Group after publication | Eligibility update timing TBD | Decision-blocked |
| ASMT-QA-036 | F005 | Student removed from Group during active attempt | Active attempt policy TBD | Decision-blocked |
| ASMT-QA-037 | F005 | Group inactive at start | Eligibility rule TBD | Decision-blocked |
| ASMT-QA-038 | F005 | Manager assigns across Section scope | Deny unless approved exception | RBAC |
| ASMT-QA-039 | F006 | Publish valid configured Assessment | Transition to PUBLISHED if actor/action approved | Decision-blocked |
| ASMT-QA-040 | F006 | Student attempts publish | Deny | RBAC |
| ASMT-QA-041 | F006 | Publish Assessment with no Problems | Block if required by approved readiness rules | Decision-blocked |
| ASMT-QA-042 | F006 | Publish with invalid startsAt/endsAt | Block under approved timing validation | Decision-blocked |
| ASMT-QA-043 | F006 | Publish action repeated | Idempotency/no-op/error behavior TBD | Decision-blocked |
| ASMT-QA-044 | F006 | Publish event/notification fails | State/delivery recovery follows Module 13/18 | Cross-module |
| ASMT-QA-045 | F007 | Archive Assessment | Status ARCHIVED, if transition permitted | Schema |
| ASMT-QA-046 | F007 | Cancel Assessment | Cancel status/action model not source-confirmed | Decision-blocked |
| ASMT-QA-047 | F007 | Reschedule published Assessment | Date/notification/active attempt rules TBD | Decision-blocked |
| ASMT-QA-048 | F007 | Archive while active attempts exist | Active attempt handling TBD | Decision-blocked |
| ASMT-QA-049 | F007 | Archive then Student opens prior result | Historical access policy TBD | Decision-blocked |
| ASMT-QA-050 | F007 | Staff outside scope archives Assessment | Deny | RBAC |
| ASMT-QA-051 | F008 | Student-A opens Student Assessments page | Page exists in original and v2 mapping | Page mapping |
| ASMT-QA-052 | F008 | Student-A sees eligible Assessment-A | Visible under approved publication/assignment policy | Decision-blocked |
| ASMT-QA-053 | F008 | Student-A requests Assessment-B | Deny/omit | RBAC |
| ASMT-QA-054 | F008 | Student-A requests detail with Student-B ID | Identity must be auth-derived; deny spoof | Security |
| ASMT-QA-055 | F008 | Student requests hidden testcase inputs/expected output | Never return protected content | Security/Module 7 |
| ASMT-QA-056 | F008 | Student list is empty | Empty state | UX TBD |
| ASMT-QA-057 | F009 | Eligible Student starts published Assessment in window | Attempt/session created according to approved policy | Decision-blocked |
| ASMT-QA-058 | F009 | Student starts before startsAt | Deny | Schema/timing |
| ASMT-QA-059 | F009 | Student starts after endsAt | Deny or approved late-start rule | Decision-blocked |
| ASMT-QA-060 | F009 | Student not assigned to Assessment attempts Start | Deny | RBAC |
| ASMT-QA-061 | F009 | Student has reached attempt limit | Block under approved counting policy | Decision-blocked |
| ASMT-QA-062 | F009 | Start click repeated due network retry | No duplicate attempt if idempotency approved; behavior TBD | Decision-blocked |
| ASMT-QA-063 | F009 | AssessmentAttempt storage absent | Implementation blocked until architecture decision | Schema gap |
| ASMT-QA-064 | F009 | Start persistence fails | Do not open phantom active attempt | Reliability |
| ASMT-QA-065 | F010 | Student refreshes active workspace | Resume same attempt if resume policy approved | Decision-blocked |
| ASMT-QA-066 | F010 | Student attempts to resume Student-B attempt | Deny | Ownership |
| ASMT-QA-067 | F010 | Browser local timer is modified | Cannot extend server deadline | Recommendation |
| ASMT-QA-068 | F010 | Attempt already expired | Resume denied or expired-state view under policy | Decision-blocked |
| ASMT-QA-069 | F010 | Draft code lost on reconnect | Recovery/autosave policy TBD | Decision-blocked |
| ASMT-QA-070 | F011 | Student submits valid code for linked Problem-A | Module 6 submission accepted if policy permits | Module 6 |
| ASMT-QA-071 | F011 | Student submits for Problem-B not in Assessment | Reject | Relationship integrity |
| ASMT-QA-072 | F011 | Student sends Student-B ID | Use authenticated Student; reject spoof | Security |
| ASMT-QA-073 | F011 | Student submits after deadline | Apply approved deadline rule | Decision-blocked |
| ASMT-QA-074 | F011 | Unsupported language submitted | Module 5/6 validation rejects | Cross-module |
| ASMT-QA-075 | F011 | Code submission API duplicate request | Idempotency behavior per Module 6 contract | Cross-module |
| ASMT-QA-076 | F011 | Judge0 unavailable | Operational failure, not code verdict | Module 6/18 |
| ASMT-QA-077 | F011 | CodeSubmission save fails | Do not report accepted submission | Reliability |
| ASMT-QA-078 | F011 | Student bypasses DEZOLVER and calls Judge0 directly | Backend-only execution boundary; reject/unsupported | Security |
| ASMT-QA-079 | F012 | Submit exactly at endsAt | Inclusive/exclusive boundary TBD | Decision-blocked |
| ASMT-QA-080 | F012 | Student starts near endsAt | Duration vs global close precedence TBD | Decision-blocked |
| ASMT-QA-081 | F012 | durationMinutes expires before endsAt | Per-attempt deadline behavior TBD | Decision-blocked |
| ASMT-QA-082 | F012 | endsAt occurs before per-attempt duration | Global cutoff precedence TBD | Decision-blocked |
| ASMT-QA-083 | F012 | Client clock altered | Server time authoritative if approved | Recommendation |
| ASMT-QA-084 | F012 | Timezone/DST boundary | Timezone policy TBD | Decision-blocked |
| ASMT-QA-085 | F012 | Browser closes at expiry | Auto-submit/lock/save policy TBD | Decision-blocked |
| ASMT-QA-086 | F012 | Expiry job fails | Submission endpoint still checks deadline | Recommendation |
| ASMT-QA-087 | F013 | Module 7 sends completed evaluation for linked submission | Associate correct result/context | Cross-module |
| ASMT-QA-088 | F013 | Result belongs to wrong Assessment/Student | Reject mismatched relation | Integrity |
| ASMT-QA-089 | F013 | Evaluation remains pending | Do not mark final score | Cross-module |
| ASMT-QA-090 | F013 | Judge0 infrastructure error | Error distinct from wrong answer | Module 6/7 |
| ASMT-QA-091 | F013 | Same result event delivered twice | No duplicate score/progress side effect | Idempotency TBD |
| ASMT-QA-092 | F013 | Old rejudge result arrives after newer result | Stale event policy/version required | Decision-blocked |
| ASMT-QA-093 | F014 | Assessment has one fully evaluated Problem | Score formula still needs approved rule | Decision-blocked |
| ASMT-QA-094 | F014 | Assessment has multiple Problems with equal score | Aggregation formula TBD | Decision-blocked |
| ASMT-QA-095 | F014 | AssessmentProblem weights are configured | No weight field in source schema; block until approved | Schema gap |
| ASMT-QA-096 | F014 | One Problem unanswered | Missing-score policy TBD | Decision-blocked |
| ASMT-QA-097 | F014 | One Problem evaluation is pending | Do not calculate final grade unless approved | Decision-blocked |
| ASMT-QA-098 | F014 | Judge0 infrastructure error for one Problem | Error/score policy TBD; do not assume zero | Decision-blocked |
| ASMT-QA-099 | F014 | Score equals passingPercentage | Inclusive/exclusive pass boundary TBD | Decision-blocked |
| ASMT-QA-100 | F014 | Score decimal rounding boundary | Precision/rounding policy TBD | Decision-blocked |
| ASMT-QA-101 | F014 | Multiple attempts have different scores | Official attempt selection TBD | Decision-blocked |
| ASMT-QA-102 | F014 | passingPercentage outside valid range | Range validation must be approved and enforced | Schema |
| ASMT-QA-103 | F014 | Re-evaluation changes Problem score | Recalculation/history policy TBD | Decision-blocked |
| ASMT-QA-104 | F015 | Result release policy is immediate | Student sees allowed details only | Decision-blocked |
| ASMT-QA-105 | F015 | Result release is withheld | Student cannot bypass via API | Security |
| ASMT-QA-106 | F015 | Student-A accesses Student-B result | Deny | Ownership |
| ASMT-QA-107 | F015 | Student views hidden testcase details | No hidden input/expected output returned | Security |
| ASMT-QA-108 | F015 | Staff releases result without permission | Deny | RBAC |
| ASMT-QA-109 | F015 | Evaluation pending when results released | Pending/final state policy applied | Decision-blocked |
| ASMT-QA-110 | F015 | Legacy immediate-result behavior is proposed as default | Must remain recommendation until approved | Evidence rule |
| ASMT-QA-111 | F016 | HOD-A reviews Department-A roster | Only permitted Students/data | RBAC |
| ASMT-QA-112 | F016 | HOD-A requests Department-B Student | Deny | RBAC |
| ASMT-QA-113 | F016 | Manager-A requests Section-B record | Deny | RBAC |
| ASMT-QA-114 | F016 | Staff with View tries to edit score | Deny absent separate permission | RBAC |
| ASMT-QA-115 | F016 | Staff with View requests source code | Deny unless explicit source access permission | Security |
| ASMT-QA-116 | F016 | Staff requests hidden testcase outputs | Deny/mask | Security |
| ASMT-QA-117 | F017 | Authorized staff re-evaluates submission | Run Module 6/7 flow if feature approved | Decision-blocked |
| ASMT-QA-118 | F017 | Student requests re-evaluation | Deny unless explicitly approved | RBAC |
| ASMT-QA-119 | F017 | Re-evaluation creates extra Assessment attempt | Must follow explicit policy; recommended no extra attempt | Decision-blocked |
| ASMT-QA-120 | F017 | Two re-evaluations run concurrently | Serialize/reject policy TBD | Decision-blocked |
| ASMT-QA-121 | F017 | Re-evaluation infrastructure fails | No fabricated final score | Reliability |
| ASMT-QA-122 | F017 | Older result arrives last | Prevent stale overwrite after version policy approved | Decision-blocked |
| ASMT-QA-123 | F018 | Score adjustment feature not approved | Do not expose/enable action | Decision-blocked |
| ASMT-QA-124 | F018 | Unauthorized Manager adjusts out-of-scope score | Deny | RBAC |
| ASMT-QA-125 | F018 | Authorized adjustment lacks required reason | Reject if reason requirement approved | Decision-blocked |
| ASMT-QA-126 | F018 | Adjusted score exceeds permitted range | Reject per approved range | Decision-blocked |
| ASMT-QA-127 | F018 | Adjustment changes pass/fail | Recompute according to approved policy | Decision-blocked |
| ASMT-QA-128 | F018 | Concurrent score adjustments | Conflict/history policy TBD | Decision-blocked |
| ASMT-QA-129 | F018 | Adjustment history/audit write fails | Do not report adjustment complete | Reliability |
| ASMT-QA-130 | F019 | Integrity telemetry disabled | No collection | Decision-blocked |
| ASMT-QA-131 | F019 | Telemetry enabled without Student notice | Do not enable until notice/policy approved | Privacy |
| ASMT-QA-132 | F019 | Tab/window blur event occurs | Collect only if approved and disclosed | Decision-blocked |
| ASMT-QA-133 | F019 | Paste event occurs | Do not automatically fail Student based only on event | Recommendation |
| ASMT-QA-134 | F019 | Student attempts to view another Student integrity event | Deny | Ownership/privacy |
| ASMT-QA-135 | F019 | Staff reviewer outside scope opens event | Deny | RBAC |
| ASMT-QA-136 | F019 | Legacy flagging-path behavior is used as target requirement | Must not; legacy evidence only | Evidence rule |
| ASMT-QA-137 | F020 | HOD-A report requested for Department-A | Scoped data only | RBAC |
| ASMT-QA-138 | F020 | Manager-A report requested for Section-B | Deny | RBAC |
| ASMT-QA-139 | F020 | View-only actor attempts CSV export | Deny unless export permission approved | RBAC |
| ASMT-QA-140 | F020 | Export includes hidden testcase input/output | Must exclude | Security |
| ASMT-QA-141 | F020 | Large report generation times out | Background/pagination recovery policy TBD | Decision-blocked |
| ASMT-QA-142 | F020 | Client uses foreign scope filter | Reject or safely scope-filter | Security |
| ASMT-QA-143 | F021 | Assessment published and event is approved | Emit event for Module 13 | Cross-module |
| ASMT-QA-144 | F021 | Notification recipient belongs to foreign College | Reject recipient | Tenant scope |
| ASMT-QA-145 | F021 | Duplicate lifecycle event | Deduplication policy TBD | Decision-blocked |
| ASMT-QA-146 | F021 | Notification delivery fails | Assessment state remains consistent; Module 13/18 recovery | Cross-module |
| ASMT-QA-147 | F021 | Notification payload includes hidden test data/code | Exclude sensitive payload | Security |
| ASMT-QA-148 | F021 | Assessment event type not mapped to NotificationType enum | Resolve event/type mapping before implementation | Schema gap |
| ASMT-QA-149 | F022 | Student has attempts remaining | Permit per approved start/attempt policy | Decision-blocked |
| ASMT-QA-150 | F022 | Student reaches maxAttempts | Block under approved count semantics | Decision-blocked |
| ASMT-QA-151 | F022 | Client changes attempt counter | Server ignores/rejects manipulation | Security |
| ASMT-QA-152 | F022 | Staff grants extra attempt without permission | Deny | RBAC |
| ASMT-QA-153 | F022 | Extra attempt granted | Actor/reason/audit policy required | Decision-blocked |
| ASMT-QA-154 | F022 | Infrastructure failure occurs | Attempt refund/recovery policy TBD | Decision-blocked |
| ASMT-QA-155 | F022 | Rejudge occurs | Must not count as Student attempt absent explicit rule | Decision-blocked |
| ASMT-QA-156 | F022 | Workspace opened but no code submitted | Attempt consumption behavior TBD; legacy open did not consume | Decision-blocked |
| ASMT-QA-157 | F022 | Duplicate submit request | Attempt count/idempotency policy TBD | Decision-blocked |
| ASMT-QA-158 | F022 | Attempt record creation fails | No phantom attempt/count | Reliability |

## QA review notes
- 158 cases cover ASMT-F001–ASMT-F022.
- “Decision-blocked” cases are retained for the later product review as requested.
- These tests do not approve the recommendations or unresolved expected behaviors.
