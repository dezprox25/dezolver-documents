# DEZOLVER — Module 11 Contests QA Test Cases

**Version:** 0.2 | **Status:** Detailed QA draft; unresolved policy cases remain decision-blocked  
**Standard:** Functional Specification Standard v1.0  
**Related:** [Module 11 README](./README.md) · [Function Specs](./FUNCTION-SPECIFICATIONS.md)

> QA cases do not approve unresolved contest rules. Where a policy is undecided, expected behavior is explicitly “apply approved decision; currently TBD.”

## Test fixtures
- SUPER_ADMIN global.
- Finance-A assigned College A.
- HOD-A College A / Department A; HOD-B Department B.
- Manager-A Section A; Manager-B other section; Contest access mapping absent unless explicitly provided.
- Student-A eligible College/Department A; Student-B another Student; Student-C outside College/Department.
- Contest-A College A; Contest-B foreign College/Department; Contest-UPCOMING; Contest-LIVE; Contest-ENDED; Contest inactive.
- Problem-A authorized; Problem-B foreign/hidden; CP-A ContestProblem link.
- Participant-A Student-A; Participant-B Student-B.
- Score-A has approved evaluation data; score/penalty formula fixtures only when product policy is approved.

## QA cases

| ID | Function | Scenario | Expected result | Trace |
|---|---|---|---|---|
| CON-QA-001 | CON-F001 | HOD-A lists Contests | Only authorized scope; Department scope decision must be applied | RBAC/CON-D01 |
| CON-QA-002 | CON-F001 | Manager-A lists Contest without explicit Group/Section mapping | No inferred access | CON-D01 |
| CON-QA-003 | CON-F001 | Finance-A requests Contest list | College A only; allowed fields/page per decision | CON-D03 |
| CON-QA-004 | CON-F001 | Student-A lists Contests | Permitted academic Contests only | RBAC |
| CON-QA-005 | CON-F001 | Client supplies foreign collegeId | Cannot widen scope | RBAC |
| CON-QA-006 | CON-F001 | Invalid status/page/sort filter | Reject or safe behavior per approved API contract | Module 18 |
| CON-QA-007 | CON-F002 | HOD-A opens in-scope Contest | Permitted detail shown | RBAC |
| CON-QA-008 | CON-F002 | Student-A opens foreign Contest-B | Denied/safe not-found | RBAC |
| CON-QA-009 | CON-F002 | Contest detail contains hidden Problem | Hidden data not exposed | Module 5 |
| CON-QA-010 | CON-F002 | GroupContest link is visible | No automatic team mode implied | CON-D09 |
| CON-QA-011 | CON-F003 | HOD-A creates Contest in College A | Created in authorized scope | RBAC/schema |
| CON-QA-012 | CON-F003 | Student-A attempts create | Denied | RBAC |
| CON-QA-013 | CON-F003 | HOD/Manager submits foreign College ID | Denied; no record | RBAC |
| CON-QA-014 | CON-F003 | title blank/over 255 | Requiredness TBD; max length enforced | Schema/decision |
| CON-QA-015 | CON-F003 | end time earlier than start | Apply approved date rule; recommendation reject | CON-D05 |
| CON-QA-016 | CON-F003 | client sets createdById/role | Server auth identity used | Security |
| CON-QA-017 | CON-F004 | HOD edits allowed title/description | Approved fields persist | RBAC |
| CON-QA-018 | CON-F004 | Manager edits foreign Contest | Denied | RBAC |
| CON-QA-019 | CON-F004 | Client edits collegeId/creator | Reject/ignore; scope unchanged | Security |
| CON-QA-020 | CON-F004 | Live Contest schedule edit | Apply approved policy; no silent change | CON-D05/20 |
| CON-QA-021 | CON-F004 | Concurrent/stale update | Shared conflict behavior; no unapproved overwrite | Module 18 |
| CON-QA-022 | CON-F005 | HOD adds Problem-A to Contest-A | ContestProblem link created | Schema |
| CON-QA-023 | CON-F005 | Foreign Problem-B added | Denied | RBAC |
| CON-QA-024 | CON-F005 | Duplicate ContestProblem | Apply uniqueness policy; decision pending | CON-D23 |
| CON-QA-025 | CON-F005 | Invalid pointsWeight | Range/precision policy TBD; no silent coercion | CON-D15 |
| CON-QA-026 | CON-F005 | Reorder linked Problems | Approved order persists | Schema/orderIndex |
| CON-QA-027 | CON-F005 | Remove Problem after submissions exist | Preserve submission/history; removal policy TBD | CON-D11 |
| CON-QA-028 | CON-F006 | Contest reaches start time | Status transition per approved state machine | CON-D04/05 |
| CON-QA-029 | CON-F006 | Contest reaches end time | ENDED transition per approved state machine | CON-D04/05 |
| CON-QA-030 | CON-F006 | Unauthorized Manager/Student status change | Denied | RBAC |
| CON-QA-031 | CON-F006 | isActive false while status LIVE | Apply explicit precedence; currently TBD | CON-D04 |
| CON-QA-032 | CON-F006 | Scheduler and staff transition race | Consistent/idempotent state | Module 17/18 |
| CON-QA-033 | CON-F006 | Repeated end event | No duplicate side effects; idempotency policy | Module 18 |
| CON-QA-034 | CON-F007 | Authorized staff links Group-A/Contest-A | GroupContest created | Schema |
| CON-QA-035 | CON-F007 | Foreign Group or Contest selected | Denied | RBAC |
| CON-QA-036 | CON-F007 | Duplicate GroupContest link | Prevent per approved constraint | CON-D23 |
| CON-QA-037 | CON-F007 | Group link created | Does not auto-create ContestParticipant | CON-D09 |
| CON-QA-038 | CON-F007 | Unlink Group during/after Contest | Preserve participation/submission/score history | CON-D21 |
| CON-QA-039 | CPART-F001 | Student-A browses Contest catalog | Only permitted contests | RBAC |
| CON-QA-040 | CPART-F001 | Student search matches foreign Contest | Foreign row omitted | Tenant isolation |
| CON-QA-041 | CPART-F001 | Student opens inactive/unpublished Contest | Visibility per approved policy | CON-D04/06 |
| CON-QA-042 | CPART-F001 | Student account suspended | Deny according to Module 1 | Auth |
| CON-QA-043 | CPART-F001 | status/isActive inconsistent | No silent normalization | CON-D04 |
| CON-QA-044 | CPART-F002 | Eligible Student validates join | Eligibility result true | CON-D08 |
| CON-QA-045 | CPART-F002 | Student outside College scope | Deny unless explicit cross-scope rule | CON-D08 |
| CON-QA-046 | CPART-F002 | Student not enrolled in course | Apply policy; source does not require enrollment | CON-D08 |
| CON-QA-047 | CPART-F002 | Student not member of linked Group | Apply explicit Group eligibility policy | CON-D09 |
| CON-QA-048 | CPART-F002 | Contest outside registration/time window | Apply approved boundary policy | CON-D05/10 |
| CON-QA-049 | CPART-F003 | Student joins eligible Contest | One participant record created if self-registration approved | CON-D07 |
| CON-QA-050 | CPART-F003 | Student joins twice | Duplicate behavior/uniqueness policy | CON-D10/23 |
| CON-QA-051 | CPART-F003 | Student submits other Student ID | Auth identity used | Security |
| CON-QA-052 | CPART-F003 | Student joins after cutoff | Reject if cutoff policy approved | CON-D10 |
| CON-QA-053 | CPART-F003 | GroupContest association exists | Members not auto-registered absent decision | CON-D09 |
| CON-QA-054 | CPART-F003 | Join request retried after timeout | No duplicate participant if idempotency approved | Module 18 |
| CON-QA-055 | CPART-F004 | Student views own participation | Own row shown | RBAC |
| CON-QA-056 | CPART-F004 | Student requests full roster | Visibility follows approved policy | CON-D24 |
| CON-QA-057 | CPART-F004 | Staff views roster in scope | Permitted fields only | RBAC |
| CON-QA-058 | CPART-F004 | Foreign Contest roster requested | Denied | RBAC |
| CON-QA-059 | CSUB-F001 | Student opens linked permitted Problem | Statement allowed by policy | Module 5 |
| CON-QA-060 | CSUB-F001 | Problem not linked to Contest | Denied | ContestProblem relation |
| CON-QA-061 | CSUB-F001 | Nonparticipant opens Problem workspace | Apply participation policy | CON-D07/08 |
| CON-QA-062 | CSUB-F001 | Student views hidden testcase | Denied | Module 5 |
| CON-QA-063 | CSUB-F001 | Problem edited after Contest publish | Snapshot/version policy TBD | CON-D11 |
| CON-QA-064 | CSUB-F002 | Eligible Student submits code | Accepted/queued via Module 6 | Business Flow |
| CON-QA-065 | CSUB-F002 | Nonparticipant submits | Denied | RBAC/participation |
| CON-QA-066 | CSUB-F002 | Submit after Contest deadline | Apply end boundary/grace rule | CON-D05/13 |
| CON-QA-067 | CSUB-F002 | Contest context missing from submission | Require approved context linkage; do not guess | CON-D12 |
| CON-QA-068 | CSUB-F002 | Unsupported language | Module 5/6 validation | Modules 5/6 |
| CON-QA-069 | CSUB-F002 | Submission limit exceeded | Apply approved limit | CON-D13 |
| CON-QA-070 | CSUB-F002 | Judge0 service unavailable | Operational error; not automatic wrong answer/zero | Module 6/18 |
| CON-QA-071 | CSUB-F002 | Client retry submits same payload | Idempotency/duplicate policy | Module 18 |
| CON-QA-072 | CSUB-F003 | Student views own submission result | Permitted result shown | RBAC |
| CON-QA-073 | CSUB-F003 | Student views another Student code | Denied | Ownership |
| CON-QA-074 | CSUB-F003 | Evaluation pending | Pending status shown, not final verdict | Module 7 |
| CON-QA-075 | CSUB-F003 | Hidden testcase failed | Do not reveal hidden input/output | Module 7 |
| CON-QA-076 | CSUB-F003 | Staff views submission in foreign scope | Denied | RBAC |
| CON-QA-077 | CSCORE-F001 | Accepted/failed verdict score calculated | Approved formula applied; currently TBD | CON-D15 |
| CON-QA-078 | CSCORE-F001 | pointsWeight changes | Score effect follows approved rule | CON-D15 |
| CON-QA-079 | CSCORE-F001 | Partial-credit evaluation | Apply approved scoring policy | CON-D15 |
| CON-QA-080 | CSCORE-F001 | Duplicate evaluation callback | No double scoring | Module 18 |
| CON-QA-081 | CSCORE-F001 | Judge operational error | Do not assign zero unless approved | Module 7/18 |
| CON-QA-082 | CSCORE-F001 | Submission references another Contest Problem | Reject cross-context result | CON-D12 |
| CON-QA-083 | CSCORE-F002 | Aggregate multiple Problem scores | Approved total formula | CON-D15 |
| CON-QA-084 | CSCORE-F002 | Multiple submissions per Problem | Best/latest/first rule TBD | CON-D15 |
| CON-QA-085 | CSCORE-F002 | Unattempted Problem | Score treatment TBD | CON-D15 |
| CON-QA-086 | CSCORE-F002 | Decimal precision/rounding | Approved precision applied | CON-D15 |
| CON-QA-087 | CSCORE-F002 | Score update triggers ranking | Rank refresh follows Module 11.5 | Cross-module |
| CON-QA-088 | CSCORE-F003 | Contest end with all evaluations complete | Finalization per approved policy | CON-D17 |
| CON-QA-089 | CSCORE-F003 | Contest end with Judge queue pending | Wait/finalize policy TBD | CON-D17 |
| CON-QA-090 | CSCORE-F003 | Rejudge after final score | Recompute/lock policy TBD | CON-D17 |
| CON-QA-091 | CSCORE-F003 | Staff attempts manual score change | Deny unless permission approved | RBAC |
| CON-QA-092 | CSCORE-F003 | Re-finalize score | Idempotent/auditable behavior | Module 18 |
| CON-QA-093 | CPEN-F001 | Accepted solution penalty computed | Approved formula and unit; currently TBD | CON-D16 |
| CON-QA-094 | CPEN-F001 | Wrong-answer penalty | Policy TBD; no standard assumed | CON-D16 |
| CON-QA-095 | CPEN-F001 | Compile/runtime error penalty | Policy TBD | CON-D16 |
| CON-QA-096 | CPEN-F001 | Timestamp/timezone boundary | Approved Contest time policy | CON-D05/16 |
| CON-QA-097 | CPEN-F001 | Duplicate submission event | No double penalty | Module 18 |
| CON-QA-098 | CPEN-F001 | Judge failure | No guessed penalty | Module 7/18 |
| CON-QA-099 | CPEN-F002 | Rejudge changes verdict | Recompute penalty per approved policy | CON-D16/17 |
| CON-QA-100 | CPEN-F002 | Event arrives out of order | Consistent recompute | Module 18 |
| CON-QA-101 | CPEN-F002 | Contest reopens | Penalty recalculation policy TBD | CON-D20 |
| CON-QA-102 | CPEN-F002 | Penalty changes | Ranking refresh triggered | Module 11.5 |
| CON-QA-103 | CRANK-F001 | Score rows rank normally | Approved rank order | CON-D18 |
| CON-QA-104 | CRANK-F001 | Equal scores | Tie policy applied; currently TBD | CON-D19 |
| CON-QA-105 | CRANK-F001 | Equal score and penalty | Tie-break decision needed | CON-D19 |
| CON-QA-106 | CRANK-F001 | Participant has pending evaluation | Provisional/final policy applied | CON-D18 |
| CON-QA-107 | CRANK-F001 | Score/penalty changes after rank | Recalculate ranks | CON-D17/18 |
| CON-QA-108 | CRANK-F001 | Duplicate score row | Integrity/uniqueness policy | CON-D23 |
| CON-QA-109 | CRANK-F002 | Student opens Leaderboard | Permitted ranking fields shown | RBAC |
| CON-QA-110 | CRANK-F002 | Student tries another Student private rank/score | Apply own/permitted restriction | RBAC |
| CON-QA-111 | CRANK-F002 | Finance opens leaderboard | College scope and fields per decision | CON-D03 |
| CON-QA-112 | CRANK-F002 | Results not released | Follow release policy; TBD | CON-D18/24 |
| CON-QA-113 | CRANK-F002 | Foreign Contest leaderboard requested | Denied | RBAC |
| CON-QA-114 | CRANK-F003 | All scoring/penalty finalized | Standings finalize under approved criteria | CON-D17/18 |
| CON-QA-115 | CRANK-F003 | Evaluation still pending | Do not claim final unless policy permits | CON-D17 |
| CON-QA-116 | CRANK-F003 | Rejudge after leaderboard publication | Update/release policy TBD | CON-D17/22 |
| CON-QA-117 | CRANK-F003 | Notification delivery fails | Result state consistent; Module 13/18 retry | Cross-module |
| CON-QA-118 | CTIE-F001 | Equal scores detected | Tie identified according to approved precision | CON-D19 |
| CON-QA-119 | CTIE-F001 | Equal score but different penalty | Use penalty only if approved tie criterion | CON-D19 |
| CON-QA-120 | CTIE-F001 | Equal score and penalty | Tie remains unresolved unless approved rule | CON-D19 |
| CON-QA-121 | CTIE-F001 | Missing completedAt | Do not use as implicit tie-break | CON-D19 |
| CON-QA-122 | CTIE-F002 | Tie rule applies | Rank follows approved sequence | CON-D19 |
| CON-QA-123 | CTIE-F002 | Tie rule does not resolve tie | Shared rank/manual policy TBD | CON-D19 |
| CON-QA-124 | CTIE-F002 | Staff manually resolves tie | Deny unless permission approved; audit if allowed | RBAC/Module 15 |
| CON-QA-125 | CTIE-F002 | Rejudge after tie resolution | Re-run rank/tie process | CON-D17/19 |
| CON-QA-126 | CTIE-F002 | Tie-break data missing or conflicting | Defer final rank; do not arbitrarily choose | CON-D19 |

| CON-QA-127 | CON-F003/F004 | Target Contest payload is evaluated against legacy Event-only fields (kind, visibility, maxParticipants, paid, registrationFeePaise) | Do not require or persist legacy-only fields unless a target decision and schema change approve them | LEGACY-SPEC-REPORTED / target schema |
| CON-QA-128 | CON-F006 | Legacy Event state sequence/30-second watcher is assumed for target Contest | Target transitions use only the approved Module 17 Contest state machine; historical Event states/cadence are not copied | LEGACY-SPEC-REPORTED / FSM-D04 |
| CON-QA-129 | CPART-F002/F003 | Registration guard only permits legacy literal `published` after Event watcher moves to `registration_open` | Do not reproduce reported lockout; enforce approved target eligibility/window contract | LEGACY-SPEC-REPORTED / CON-D04/D05/D10 |
| CON-QA-130 | CPART-F003 | Legacy Registration.teamName is interpreted as proof of target team mode | Target team/shared submission behavior remains decision-bound; GroupContest alone does not imply team registration | LEGACY-SPEC-REPORTED / CON-D09 |
| CON-QA-131 | CON-F005/CSUB-F001 | Problem is linked only in client-side config/unpersisted EventConfig | Use target ContestProblem records; legacy Event storage gap is not copied | Target schema / legacy report |
| CON-QA-132 | CSUB-F002 | Client submits via legacy `/events/:id/submissions` stub or Assessment proxy with `metadata.eventId` | Use Module 6 `contextType=CONTEST`, `contextId=Contest.id`; no Assessment proxy | LEGACY-SPEC-REPORTED / Module 6 |
| CON-QA-133 | CSUB-F002 | Submission body Contest ID differs from persisted context | Module 6 context and Contest authorization must agree; mismatch denied and not scored | Module 6 / CON-D12 |
| CON-QA-134 | CSCORE-F001 | Legacy `partial` verdict/EventResult.score is copied as target evaluation contract | Consume Module 7 finalized score/verdict; target ContestScoreRank is not legacy EventResult | Module 7 / legacy report |
| CON-QA-135 | CSCORE-F001/F002 | pointsWeight assumed to be raw multiplier/percentage without an approved formula | Apply no guessed weighting formula; calculation blocked by CON-D15 | CON-D15 |
| CON-QA-136 | CON-F005/CSUB-F001 | Problem content changes without applying approved ProblemVersion snapshot rule | Follow approved CON-D11 version/snapshot policy; no silent current-Problem substitution | Module 5 / CON-D11 |
| CON-QA-137 | CPEN-F001 | Default 20-minute ICPC penalty applied from legacy/frontend configuration label | No default penalty; apply only approved CON-D16 unit/formula | LEGACY-SPEC-REPORTED / CON-D16 |
| CON-QA-138 | CRANK-F001 | Rank computed with legacy ascending Redis `zrank` behavior | Do not reproduce inverse-rank defect; use approved target CON-D18 policy | LEGACY-SPEC-REPORTED / CON-D18 |
| CON-QA-139 | CTIE-F001/F002 | Equal-score participants receive arbitrary sequential ranks from database row order | Do not invent tie ordering; apply approved CON-D19 or keep tie decision-blocked | LEGACY-SPEC-REPORTED / CON-D19 |
| CON-QA-140 | CRANK-F001/CRANK-F003 | Event completion applies legacy ELO K=32, initial 1200, floor 100 | Do not apply legacy ELO values; Module 16 owns approved rating policy | LEGACY-SPEC-REPORTED / CON-D27 / ANA-D13 |
| CON-QA-141 | CRANK-F003 | Top-three participants automatically become certificateEligible because legacy `rank < 3` | Certificate criteria follow Module 12; no top-three target rule assumed | LEGACY-SPEC-REPORTED / Module 12 |
| CON-QA-142 | CPART-F003 | Paid Contest order created automatically via legacy event-registration orders | No paid entry/gateway contract assumed without CON-D26 and Module 3 approval | LEGACY-SPEC-REPORTED / CON-D26 |
| CON-QA-143 | CON-F006/CPART-F003 | Contest cancellation automatically issues Razorpay refunds from legacy EventCancelled behavior | Refund behavior requires approved target paid-entry/cancellation contract | LEGACY-SPEC-REPORTED / CON-D20/D26 / Module 3 |
| CON-QA-144 | CON-F001/CPART-F002 | Target eligibility skips College/Department/Section/Group scope because legacy registration reportedly lacked audience enforcement | Enforce target RBAC/resource scope; legacy omission is a reported defect, not target policy | RBAC / CON-D01/D06/D08 |
| CON-QA-145 | CRANK-F002/CRANK-F003 | Live leaderboard assumes legacy Redis/WebSocket room without approved authorization contract | Real-time transport remains CON-D28; any push/read access independently enforces target scope | LEGACY-SPEC-REPORTED / CON-D28 |
| CON-QA-146 | CPART-F001/CPART-F003 | 24h/1h reminders emitted because legacy Event subsystem scheduled them | Target reminders/recipients not assumed; Module 13 and CON-D22 own decision | LEGACY-SPEC-REPORTED / Module 13 |
| CON-QA-147 | CON-F006/CRANK-F003 | Legacy watcher/final-grading delay used as target timing | Use approved Module 17 transitions and CON-D04/D05/D17; recovery belongs to Module 18 | LEGACY-SPEC-REPORTED / Modules 17–18 |
| CON-QA-148 | CRANK-F002/CRANK-F003 | Target leaderboard reads legacy global query or assumes legacy EventResult/Person rating persistence | Use target ContestScoreRank and authorized projections; global rating remains CON-D27/Module 16 | LEGACY-SPEC-REPORTED / target schema |

## Coverage
148 cases span Contest management, participation, submission, scoring, penalty, ranking, tie-breaking, RBAC, privacy, lifecycle, data integrity, cross-module failures and legacy-to-target regression boundaries. CON-QA-127–148 prevent Event-era fields, lifecycle timing, Assessment-proxy submission, score/rank defects, ELO, certificate, payment/refund and notification behavior from being silently imported. Test cases are defined only; they have not been executed. Policy-dependent expected results remain open pending product decisions.
