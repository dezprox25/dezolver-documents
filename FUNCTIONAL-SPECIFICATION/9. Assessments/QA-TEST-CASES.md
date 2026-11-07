# Module 9 — Assessments QA Test Cases

**Version:** 1.0 — adopted target baseline QA companion  
**Status:** 188 target cases defined; implementation execution not yet performed  
**Related:** [Module 9 README](./README.md) · [Function Specifications](./FUNCTION-SPECIFICATIONS.md)

> QA cases define the adopted Module 9 target expectations. They are test specifications, not execution evidence; source/legacy observations are preserved separately.

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
| ASMT-QA-005 | F001 | Finance Assessment API/page access | FINANCE_TEAM | Deny; Assessment is not a Finance module | ASMT-F001; ASMT-D01 |
| ASMT-QA-006 | F001 | Client changes departmentId filter | Cannot broaden scope | Security |
| ASMT-QA-007 | F001 | Empty scoped Assessment list | Safe empty state | UX See ASMT-D01–ASMT-D36 |
| ASMT-QA-008 | F002 | HOD creates valid Assessment in Department-A | Record saved with authorized scope/creator | Schema/RBAC |
| ASMT-QA-009 | F002 | Manager creates in assigned Section/Group scope | Allowed only if approved Assessment create scope supports it | Target policy defined by ASMT-D01–ASMT-D36 |
| ASMT-QA-010 | F002 | Student attempts create | Denied | RBAC |
| ASMT-QA-011 | F002 | Finance Team attempts create | Denied | RBAC |
| ASMT-QA-012 | F002 | Client supplies foreign College ID | Server rejects/overrides invalid scope | Security |
| ASMT-QA-013 | F002 | Unsupported Assessment type | Reject enum value; CODING is current source enum | Schema |
| ASMT-QA-014 | F002 | Manager creates Assessment draft | Manager assigned Section-A | Allow scoped DRAFT create only; server sets collegeId/departmentId/createdById | ASMT-F002; ASMT-D01/D06 |
| ASMT-QA-015 | F002 | Missing fields required by publish/create policy | Block according to approved required field list | target policy defined by ASMT-D01–ASMT-D36 |
| ASMT-QA-016 | F002 | Create request times out and client retries | Duplicate prevention/idempotency behavior TBD | target policy defined by ASMT-D01–ASMT-D36 |
| ASMT-QA-017 | F003 | Published Assessment configuration edit | Published revision exists | Create a new draft revision; never mutate current published revision | ASMT-F003; ASMT-D11/D13 |
| ASMT-QA-018 | F003 | Manager-A edits in-scope Assessment-A | Allow only fields/actions authorized | RBAC |
| ASMT-QA-019 | F003 | Manager-A edits Assessment-B | Deny | RBAC |
| ASMT-QA-020 | F003 | Student directly calls update endpoint | Deny | RBAC |
| ASMT-QA-021 | F003 | Update status to unsupported enum | Reject | Schema |
| ASMT-QA-022 | F004 | Duplicate Problem association | Problem-A already linked | Reject duplicate Problem ID in same revision | ASMT-F004; ASMT-D03/D04 |
| ASMT-QA-023 | F004 | Problem ordering | Several Problems added/reordered | Persist zero-based contiguous unique orderIndex | ASMT-F004; ASMT-D04 |
| ASMT-QA-024 | F004 | Add Problem-A to Assessment-A | Association created with valid IDs/order | Schema |
| ASMT-QA-025 | F004 | Add Problem-B outside permitted scope | Deny | RBAC |
| ASMT-QA-026 | F004 | Add duplicate Problem-A | Duplicate constraint/UX policy TBD | target policy defined by ASMT-D01–ASMT-D36 |
| ASMT-QA-027 | F005 | Assessment assignment outside HOD scope | HOD-A; Assessment targets Department-B | Deny cross-Department target | ASMT-F005; ASMT-D01/D06 |
| ASMT-QA-028 | F004 | Remove Problem after attempt started | Mutation/snapshot policy TBD | target policy defined by ASMT-D01–ASMT-D36 |
| ASMT-QA-029 | F004 | Attempt to set Problem weight | Weight field absent from source schema; feature blocked pending decision | Schema gap |
| ASMT-QA-030 | F004 | Assessment includes multiple Problems | Multi-Problem relationship supported conceptually; attempt/result flow TBD | Schema |
| ASMT-QA-031 | F005 | Assign Assessment-A to Group-A | Allowed if target assignment model approved | target policy defined by ASMT-D01–ASMT-D36 |
| ASMT-QA-032 | F005 | Assign foreign Group-B | Deny | RBAC |
| ASMT-QA-033 | F006 | Manager attempts publish | Manager owns scoped DRAFT | Deny; only HOD/SUPER_ADMIN publish | ASMT-F006; ASMT-D01/D10 |
| ASMT-QA-034 | F005 | Configure individual Student exception | Not source-confirmed; feature decision required | target policy defined by ASMT-D01–ASMT-D36 |
| ASMT-QA-035 | F007 | Archive Assessment with active attempts | PUBLISHED with active attempts | Archive blocks new starts; existing attempts retain pinned revision/deadline and history | ASMT-F007; ASMT-D09/D10/D11 |
| ASMT-QA-036 | F005 | Student removed from Group during active attempt | Active attempt policy TBD | target policy defined by ASMT-D01–ASMT-D36 |
| ASMT-QA-037 | F005 | Group inactive at start | Eligibility rule TBD | target policy defined by ASMT-D01–ASMT-D36 |
| ASMT-QA-038 | F005 | Manager assigns across Section scope | Deny unless approved exception | RBAC |
| ASMT-QA-039 | F006 | Publish valid configured Assessment | Transition to PUBLISHED if actor/action approved | target policy defined by ASMT-D01–ASMT-D36 |
| ASMT-QA-040 | F008 | Student opens Assessment detail before startsAt | Eligible Student; PUBLISHED; startsAt in future | Show safe details/Upcoming state; Start remains disabled until startsAt | ASMT-F008; ASMT-D14/D15 |
| ASMT-QA-041 | F006 | Publish Assessment with no Problems | Block if required by approved readiness rules | target policy defined by ASMT-D01–ASMT-D36 |
| ASMT-QA-042 | F006 | Publish with invalid startsAt/endsAt | Block under approved timing validation | target policy defined by ASMT-D01–ASMT-D36 |
| ASMT-QA-043 | F006 | Publish action repeated | Idempotency/no-op/error behavior TBD | target policy defined by ASMT-D01–ASMT-D36 |
| ASMT-QA-044 | F006 | Publish event/notification fails | State/delivery recovery follows Module 13/18 | Cross-module |
| ASMT-QA-045 | F009 | Student opens workspace without Start | Assessment eligible | No AssessmentAttempt is created or consumed until explicit successful Start | ASMT-F009; ASMT-D18 |
| ASMT-QA-046 | F007 | Cancel Assessment | Cancel status/action model not source-confirmed | target policy defined by ASMT-D01–ASMT-D36 |
| ASMT-QA-047 | F007 | Reschedule published Assessment | Date/notification/active attempt rules TBD | target policy defined by ASMT-D01–ASMT-D36 |
| ASMT-QA-048 | F010 | Student refreshes active workspace | Active AssessmentAttempt exists | Resume same attempt and original deadline; no new attempt | ASMT-F010; ASMT-D18 |
| ASMT-QA-049 | F007 | Archive then Student opens prior result | Historical access policy TBD | target policy defined by ASMT-D01–ASMT-D36 |
| ASMT-QA-050 | F010 | Student reconnects after deadline | Attempt deadline passed | Show expired/read-only attempt; do not reopen or reset timer | ASMT-F010/F012; ASMT-D15/D16 |
| ASMT-QA-051 | F008 | Student-A opens Student Assessments page | Page exists in original and v2 mapping | Page mapping |
| ASMT-QA-052 | F008 | Student-A sees eligible Assessment-A | Visible under approved publication/assignment policy | target policy defined by ASMT-D01–ASMT-D36 |
| ASMT-QA-053 | F008 | Student-A requests Assessment-B | Deny/omit | RBAC |
| ASMT-QA-054 | F008 | Student-A requests detail with Student-B ID | Identity must be auth-derived; deny spoof | Security |
| ASMT-QA-055 | F008 | Student requests hidden testcase inputs/expected output | Never return protected content | Security/Module 7 |
| ASMT-QA-056 | F008 | Student list is empty | Empty state | UX TBD |
| ASMT-QA-057 | F009 | Eligible Student starts published Assessment in window | Attempt/session created according to approved policy | target policy defined by ASMT-D01–ASMT-D36 |
| ASMT-QA-058 | F011 | Student submits for Problem not in pinned revision | Problem-B not in AssessmentRevisionProblem | Reject before Module 6 CodeSubmission creation | ASMT-F011; ASMT-D03/D12/D34 |
| ASMT-QA-059 | F011 | Duplicate code submit request | Same Idempotency-Key and payload | Return same accepted CodeSubmission; do not create another submission/attempt | ASMT-F011; ASMT-D20/D35 |
| ASMT-QA-060 | F011 | Submission arrives after attempt deadline | Server receivedAt > deadlineAt | Reject before Module 6 execution; preserve last accepted submission | ASMT-F011/F012; ASMT-D16 |
| ASMT-QA-061 | F009 | Student has reached attempt limit | Block under approved counting policy | target policy defined by ASMT-D01–ASMT-D36 |
| ASMT-QA-062 | F009 | Start click repeated due network retry | No duplicate attempt if idempotency approved; behavior TBD | target policy defined by ASMT-D01–ASMT-D36 |
| ASMT-QA-063 | F009 | AssessmentAttempt storage absent | Implementation blocked until architecture decision | Schema gap |
| ASMT-QA-064 | F009 | Start persistence fails | Do not open phantom active attempt | Reliability |
| ASMT-QA-065 | F010 | Student refreshes active workspace | Resume same attempt if resume policy approved | target policy defined by ASMT-D01–ASMT-D36 |
| ASMT-QA-066 | F012 | Submit at exact attempt deadline | Server receivedAt equals deadlineAt | Accept; deadline comparison is inclusive | ASMT-F012; ASMT-D15/D16 |
| ASMT-QA-067 | F012 | Assessment endsAt before duration deadline | Attempt deadline is later than revision endsAt | Use endsAt as the effective deadline | ASMT-F012; ASMT-D14/D15 |
| ASMT-QA-068 | F012 | Browser timer altered | Client time is wrong | Server deadline remains authoritative; no extension | ASMT-F012; ASMT-D15 |
| ASMT-QA-069 | F012 | Expiry with unsent editor code | No CodeSubmission accepted for that code | Lock attempt; do not auto-submit browser/editor code | ASMT-F012; ASMT-D16 |
| ASMT-QA-070 | F011 | Student submits valid code for linked Problem-A | Module 6 submission accepted if policy permits | Module 6 |
| ASMT-QA-071 | F011 | Student submits for Problem-B not in Assessment | Reject | Relationship integrity |
| ASMT-QA-072 | F011 | Student sends Student-B ID | Use authenticated Student; reject spoof | Security |
| ASMT-QA-073 | F013 | Infrastructure evaluation error | Accepted CodeSubmission has Module 7 infrastructure failure | Keep attempt FINALIZING/RECOVERY_REQUIRED; no fake zero for that Problem | ASMT-F013/F014; ASMT-D17/D35 |
| ASMT-QA-074 | F011 | Unsupported language submitted | Module 5/6 validation rejects | Cross-module |
| ASMT-QA-075 | F013 | Duplicate Module 7 result event | Same eventId/runId delivered twice | Idempotently process once; no duplicate aggregate/history | ASMT-F013; ASMT-D27/D35 |
| ASMT-QA-076 | F011 | Judge0 unavailable | Operational failure, not code verdict | Module 6/18 |
| ASMT-QA-077 | F011 | CodeSubmission save fails | Do not report accepted submission | Reliability |
| ASMT-QA-078 | F011 | Student bypasses DEZOLVER and calls Judge0 directly | Backend-only execution boundary; reject/unsupported | Security |
| ASMT-QA-079 | F012 | Submit exactly at endsAt | Inclusive/exclusive boundary TBD | target policy defined by ASMT-D01–ASMT-D36 |
| ASMT-QA-080 | F012 | durationMinutes expires before endsAt | Attempt started before global endsAt | Deadline is min(startedAt+durationMinutes, endsAt) | ASMT-F012; ASMT-D15 |
| ASMT-QA-081 | F012 | durationMinutes expires before endsAt | Per-attempt deadline behavior TBD | target policy defined by ASMT-D01–ASMT-D36 |
| ASMT-QA-082 | F012 | endsAt occurs before per-attempt duration | Global cutoff precedence TBD | target policy defined by ASMT-D01–ASMT-D36 |
| ASMT-QA-083 | F012 | Client clock altered | Server time authoritative if approved | Recommendation |
| ASMT-QA-084 | F012 | Timezone/DST boundary | Timezone policy TBD | target policy defined by ASMT-D01–ASMT-D36 |
| ASMT-QA-085 | F012 | Browser closes at expiry | No additional accepted submission | Lock attempt at server deadline; preserve last accepted submission; do not auto-submit unsent code | ASMT-F012; ASMT-D16 |
| ASMT-QA-086 | F012 | Expiry job fails | Submission endpoint still checks deadline | Recommendation |
| ASMT-QA-087 | F013 | Module 7 sends completed evaluation for linked submission | Associate correct result/context | Cross-module |
| ASMT-QA-088 | F013 | Result belongs to wrong Assessment/Student | Reject mismatched relation | Integrity |
| ASMT-QA-089 | F013 | Evaluation remains pending | Do not mark final score | Cross-module |
| ASMT-QA-090 | F013 | Judge0 infrastructure error | Error distinct from wrong answer | Module 6/7 |
| ASMT-QA-091 | F013 | Stale rejudge result arrives last | Newer run already completed | Ignore stale run for current aggregate; retain both run histories | ASMT-F013/F017; ASMT-D27/D35 |
| ASMT-QA-092 | F013 | Old rejudge result arrives after newer result | Stale event policy/version required | target policy defined by ASMT-D01–ASMT-D36 |
| ASMT-QA-093 | F014 | Assessment has one fully evaluated Problem | One required Problem has 80 percent | Attempt score is 80.00; pass/fail compares to threshold | ASMT-F014; ASMT-D21/D22 |
| ASMT-QA-094 | F014 | Assessment has multiple equal-score Problems | Two Problems return 80 and 60 | Attempt score is 70.00 | ASMT-F014; ASMT-D05/D21 |
| ASMT-QA-095 | F014 | AssessmentProblem weights supplied by client | Weights not supported in v1 | Reject/ignore weight field; all Problems equally weighted | ASMT-F014; ASMT-D05 |
| ASMT-QA-096 | F014 | One Problem unanswered at Finish | Other Problems are complete | Unanswered Problem contributes 0 to denominator and score | ASMT-F014; ASMT-D17/D21 |
| ASMT-QA-097 | F014 | One Problem evaluation is pending | Do not calculate final grade unless approved | target policy defined by ASMT-D01–ASMT-D36 |
| ASMT-QA-098 | F014 | Infrastructure error for one Problem | Other submitted Problems are complete | Do not finalize attempt as if failed Problem scored 0; await recovery/reconciliation | ASMT-F014; ASMT-D17/D35 |
| ASMT-QA-099 | F014 | Score equals passingPercentage | Attempt score=50.00; threshold=50.00 | Pass; comparison is inclusive | ASMT-F014; ASMT-D22 |
| ASMT-QA-100 | F014 | Assessment score decimal rounding | Mean produces repeating decimal | Round half-up once to 2 decimal places | ASMT-F014; ASMT-D21 |
| ASMT-QA-101 | F014 | Multiple attempts have different scores | Attempt scores 60 and 80 | Official score is 80; preserve both attempts | ASMT-F014/F022; ASMT-D23 |
| ASMT-QA-102 | F014 | passingPercentage outside valid range | -1 or >100 | Reject configuration; range is 0.00–100.00 | ASMT-F014; ASMT-D22 |
| ASMT-QA-103 | F014 | Rejudge changes Problem score | SUPER_ADMIN rejudge completed for same CodeSubmission | Recompute pinned attempt and official result; preserve old run/score history | ASMT-F014/F017; ASMT-D27 |
| ASMT-QA-104 | F015 | IMMEDIATE_SAFE result policy | Attempt fully finalized | Student sees only own safe score/pass and visible-case details | ASMT-F015; ASMT-D24/D25 |
| ASMT-QA-105 | F015 | AFTER_CLOSE result policy | Assessment still open or not staff-released | Hide score/pass via API; receipt/pending remains visible | ASMT-F015; ASMT-D24 |
| ASMT-QA-106 | F015 | Student-A accesses Student-B result | Deny | Ownership |
| ASMT-QA-107 | F015 | Student views hidden testcase details | No hidden input/expected output returned | Security |
| ASMT-QA-108 | F015 | Staff releases result without permission | Manager or Student calls release | Deny; HOD/SUPER_ADMIN release only | ASMT-F015; ASMT-D01/D24 |
| ASMT-QA-109 | F015 | Evaluation pending when close event fires | Accepted CodeSubmission still processing | Keep result unreleased/finalizing until all official evaluations complete/recovered | ASMT-F015; ASMT-D17/D24 |
| ASMT-QA-110 | F015 | Legacy immediate-result behavior proposed as default | Legacy old-code audit shows immediate WebSocket delivery | Do not copy legacy default; AFTER_CLOSE is target default | ASMT-F015; ASMT-D24 |
| ASMT-QA-111 | F016 | HOD-A reviews Department-A roster | Only permitted Students/data | RBAC |
| ASMT-QA-112 | F016 | HOD-A requests Department-B Student | Deny | RBAC |
| ASMT-QA-113 | F016 | Manager-A requests Section-B record | Deny | RBAC |
| ASMT-QA-114 | F016 | Staff with View tries to edit score | HOD/Manager has scoped view only | Deny; manual score override disabled in v1 | ASMT-F016/F018; ASMT-D28 |
| ASMT-QA-115 | F016 | Staff requests source code | HOD/Manager scoped summary permission | Deny raw source; SUPER_ADMIN privileged/audited access only | ASMT-F016; ASMT-D25 |
| ASMT-QA-116 | F016 | Staff requests hidden testcase outputs | Deny/mask | Security |
| ASMT-QA-117 | F017 | SUPER_ADMIN re-evaluates submission | Reason supplied; same CodeSubmission/ProblemVersion | Module 6/7 creates new SubmissionRun; no new AssessmentAttempt | ASMT-F017; ASMT-D27 |
| ASMT-QA-118 | F017 | Student requests re-evaluation | Deny unless explicitly approved | RBAC |
| ASMT-QA-119 | F017 | Re-evaluation creates extra Assessment attempt | Existing AssessmentAttempt | Attempt count unchanged; rejudge is not another attempt | ASMT-F017/F022; ASMT-D27 |
| ASMT-QA-120 | F017 | Two re-evaluations run concurrently | Serialize/reject policy TBD | target policy defined by ASMT-D01–ASMT-D36 |
| ASMT-QA-121 | F017 | Re-evaluation infrastructure fails | No fabricated final score | Reliability |
| ASMT-QA-122 | F017 | Older result arrives last | Prevent stale overwrite after version policy approved | target policy defined by ASMT-D01–ASMT-D36 |
| ASMT-QA-123 | F018 | Score adjustment feature not approved | Manual override disabled in v1 | No score-adjustment endpoint/control available | ASMT-F018; ASMT-D28 |
| ASMT-QA-124 | F018 | Unauthorized Manager adjusts out-of-scope score | Deny | RBAC |
| ASMT-QA-125 | F018 | Authorized adjustment lacks required reason | Manual adjustment disabled | Deny feature regardless; no v1 adjustment path | ASMT-F018; ASMT-D28 |
| ASMT-QA-126 | F018 | Adjusted score exceeds permitted range | Reject per approved range | target policy defined by ASMT-D01–ASMT-D36 |
| ASMT-QA-127 | F018 | Adjustment changes pass/fail | Manual adjustment disabled | No override path; automated/rejudged score only | ASMT-F018; ASMT-D28 |
| ASMT-QA-128 | F018 | Concurrent score adjustments | Conflict/history policy TBD | target policy defined by ASMT-D01–ASMT-D36 |
| ASMT-QA-129 | F018 | Adjustment history/audit write fails | Do not report adjustment complete | Reliability |
| ASMT-QA-130 | F019 | Integrity telemetry disabled | Default revision settings | Collect no tab/window/paste signals | ASMT-F019; ASMT-D29 |
| ASMT-QA-131 | F019 | Telemetry enabled without Student notice | Revision telemetry enabled | Block Start until notice is shown/acknowledged | ASMT-F019; ASMT-D29 |
| ASMT-QA-132 | F019 | Tab/window blur event occurs | Telemetry enabled and disclosed | Store review signal only; never auto-fail/terminate | ASMT-F019; ASMT-D29 |
| ASMT-QA-133 | F019 | Paste event occurs | Telemetry enabled | Record count/timestamp only; no automatic score penalty | ASMT-F019; ASMT-D29 |
| ASMT-QA-134 | F019 | Student attempts to view another Student integrity event | Deny | Ownership/privacy |
| ASMT-QA-135 | F019 | Staff reviewer outside scope opens event | Deny | RBAC |
| ASMT-QA-136 | F019 | Legacy flagging-path behavior is used as target requirement | Must not; legacy evidence only | Evidence rule |
| ASMT-QA-137 | F020 | HOD-A report requested for Department-A | Scoped data only | RBAC |
| ASMT-QA-138 | F020 | Manager-A report requested for Section-B | Deny | RBAC |
| ASMT-QA-139 | F020 | View-only actor attempts CSV export | No export permission in v1 | Deny export; report view is read-only | ASMT-F020; ASMT-D26 |
| ASMT-QA-140 | F020 | Export includes hidden testcase input/output | Must exclude | Security |
| ASMT-QA-141 | F020 | Large report generation | Scoped data is large | Use paginated/background-safe read; failure does not expose partial/foreign data | ASMT-F020; ASMT-D26/D35 |
| ASMT-QA-142 | F020 | Client uses foreign scope filter | Reject or safely scope-filter | Security |
| ASMT-QA-143 | F021 | Assessment published and event is approved | Emit event for Module 13 | Cross-module |
| ASMT-QA-144 | F021 | Notification recipient belongs to foreign College | Reject recipient | Tenant scope |
| ASMT-QA-145 | F021 | Duplicate lifecycle event | Same outbox event delivered twice | Consumer deduplicates eventId/source revision; no duplicate notification side effect | ASMT-F021; ASMT-D31/D35 |
| ASMT-QA-146 | F021 | Notification delivery fails | Assessment state remains consistent; Module 13/18 recovery | Cross-module |
| ASMT-QA-147 | F021 | Notification payload includes hidden test data/code | Exclude sensitive payload | Security |
| ASMT-QA-148 | F021 | Assessment event type not mapped to NotificationType | Event contract not configured | Map through Module 13 event catalog before enabling event delivery | ASMT-F021; ASMT-D31 |
| ASMT-QA-149 | F022 | Student has attempts remaining | Eligible Student; no active attempt | Permit explicit Start and create one AssessmentAttempt transactionally | ASMT-F022; ASMT-D18/D19 |
| ASMT-QA-150 | F022 | Student reaches maxAttempts | Consumed attempts equal maxAttempts | Block new Start; existing attempt resumes only while active | ASMT-F022; ASMT-D19 |
| ASMT-QA-151 | F022 | Client changes attempt counter | Server ignores/rejects manipulation | Security |
| ASMT-QA-152 | F022 | Staff grants extra attempt without permission | Deny | RBAC |
| ASMT-QA-153 | F022 | Extra attempt granted | No grant permission/source policy | Reject; extra-attempt override is not supported in v1 | ASMT-F022; ASMT-D19 |
| ASMT-QA-154 | F022 | Infrastructure failure occurs | Active attempt has operational recovery state | Do not fabricate score or automatically refund; SUPER_ADMIN recovery follows audited Module 18 process | ASMT-F022; ASMT-D17/D35 |
| ASMT-QA-155 | F022 | Rejudge occurs | Existing attempt and CodeSubmission | No new attempt or attempt number | ASMT-F022; ASMT-D27 |
| ASMT-QA-156 | F022 | Workspace opened but no code submitted | Assessment not explicitly started | No attempt consumed; only accepted Start creates AssessmentAttempt | ASMT-F022; ASMT-D18 |
| ASMT-QA-157 | F022 | Duplicate submit request | Same idempotency key/payload | Return accepted submission; do not increment attempt or create duplicate CodeSubmission | ASMT-F022; ASMT-D20/D35 |
| ASMT-QA-158 | F022 | Attempt record creation fails | No phantom attempt/count | Reliability |

| ASMT-QA-159 | F004 | Revision Problem order | Three revision Problems configured | Publish revision | orderIndex values are 0,1,2 and immutable |
| ASMT-QA-160 | F005 | AssessmentSection explicit audience | AssessmentSection lists Section-A only | Student in Section-B requests access | Deny; audience rows narrow the Department scope |
| ASMT-QA-161 | F005 | Single Group assignment | Assessment.groupId=Group-A | Student in Group-B but same Department requests access | Deny; only Group-A members qualify |
| ASMT-QA-162 | F005 | Manager creates own Section draft | Manager-A assigned Section-A | Create Assessment with Section-A | Allow DRAFT only; server derives College/Department/creator |
| ASMT-QA-163 | F006 | Publish revision with archived Problem | ProblemVersion is not PUBLISHED | Publish Assessment | Reject; every revision Problem must pin a published same-College ProblemVersion |
| ASMT-QA-164 | F007 | Restore archived Assessment | Assessment ARCHIVED | HOD restores | Return to DRAFT; republish creates a new immutable revision |
| ASMT-QA-165 | F008 | Student views unpublished Assessment | Assessment DRAFT | Open Student Assessments API | Omit/deny; no draft disclosure |
| ASMT-QA-166 | F009 | Start at startsAt boundary | Server now equals startsAt | Start | Accept; start window is inclusive at startsAt |
| ASMT-QA-167 | F009 | Start at endsAt boundary | Server now equals endsAt | Start | Reject new start; endsAt is exclusive for Start |
| ASMT-QA-168 | F010 | Attempt deadline not reset on reconnect | Active attempt deadline is fixed | Refresh and resume | Return existing deadlineAt; never extend duration |
| ASMT-QA-169 | F011 | Assessment CodeSubmission context | AssessmentAttempt exists; Problem is in pinned revision | Submit via Module 6 | context=ASSESSMENT, contextId=AssessmentAttempt.id, ProblemVersion pinned |
| ASMT-QA-170 | F011 | Multiple Problem submissions within one attempt | Active attempt contains Problem A and B | Submit both Problems | Each CodeSubmission is linked to the same AssessmentAttempt and correct revision Problem |
| ASMT-QA-171 | F011 | Assessment duration not Judge0 CPU limit | Assessment duration 90 minutes; ProblemVersion CPU limit 2 seconds | Submit code | Judge0 receives ProblemVersion 2-second CPU limit; Assessment timer remains separate |
| ASMT-QA-172 | F013 | Result event mismatched to attempt | CodeSubmission context points to another AssessmentAttempt | Consume SubmissionGraded | Reject correlation; no Assessment score update |
| ASMT-QA-173 | F014 | Assessment with three Problems | Scores 100, 50 and 0 | Finalize attempt | Score is 50.00; each Problem equal weight |
| ASMT-QA-174 | F014 | Zero passing threshold | passingPercentage=0 and finalized score=0 | Finalize pass status | Pass; threshold range includes zero and comparison is >=  |
| ASMT-QA-175 | F014 | Unanswered Problem after Finish | One Problem unanswered; all submitted Problems finalized | Finish aggregation | Unanswered Problem contributes zero; aggregate denominator includes all revision Problems |
| ASMT-QA-176 | F014 | One accepted Problem evaluation is infrastructure-failed | Other Problem scores complete | Finalize attempt | No final attempt score until recovery; infrastructure failure is not zero |
| ASMT-QA-177 | F014 | Best submission within Problem | One attempt has Problem A submissions 40%, 80% | Finalize attempt | Use 80% for Problem A; keep both CodeSubmission rows |
| ASMT-QA-178 | F014 | Tie between submissions for same Problem | Two submissions have equal percentage | Select Problem result | Latest accepted submission is selected as tie-breaker |
| ASMT-QA-179 | F014 | Official attempt tie | Attempt 1 and 2 both score 75.00 | Read official result | Latest started attempt is official; both attempts retained |
| ASMT-QA-180 | F015 | AFTER_CLOSE with no endsAt | No global close timestamp | Student requests score | Keep score/pass hidden until HOD/SUPER_ADMIN explicitly releases |
| ASMT-QA-181 | F015 | Student result ownership | Student-B requests Student-A attempt | GET result | Deny/not-found-safe; no foreign result details |
| ASMT-QA-182 | F016 | Manager review foreign Section | Manager-A views Section-B roster | GET roster | Deny or safely filter; no foreign Student data |
| ASMT-QA-183 | F017 | Student attempts rejudge | Student owns CodeSubmission | Request rejudge | Deny; SUPER_ADMIN only and reason required |
| ASMT-QA-184 | F018 | Score override by HOD | HOD has Assessment CRUD | Request manual score adjustment | Deny; no manual override endpoint/control in v1 |
| ASMT-QA-185 | F019 | Telemetry auto-fail attempt | Student has multiple tab blur events | Integrity review | Do not automatically fail or terminate; review signal only |
| ASMT-QA-186 | F020 | Course completion side effect | Assessment attempt finalized | Process result event | Do not update CourseEnrollment.completedAt or seat usage |
| ASMT-QA-187 | F021 | Notification event sensitive payload | Assessment published or results released | Emit outbox event | Payload contains IDs/status/time only; no source, Judge0 token, hidden testcase or raw output |
| ASMT-QA-188 | F022 | Concurrent Start requests | Two Start requests race for last attempt | POST Start twice | Unique/transaction guard creates one attempt and returns same active attempt or conflict |

## QA review notes
- ASMT-QA-001–188 trace to ASMT-F001–ASMT-F022 and ASMT-D01–ASMT-D36.
- All expected results are adopted target requirements; cases are defined, not executed.
- The legacy Assessment audit remains factual evidence about the older implementation only.
- Release validation must include schema migration, cross-tenant authorization, timer boundary, attempt concurrency, event idempotency, rejudge and result-release tests.
