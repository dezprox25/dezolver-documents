# Module 16 — Student Analytics QA Test Cases

**Document ID:** STUDENT-ANALYTICS-QA-001 | **Version:** 0.1 | **Status:** QA draft; decision-dependent tests blocked  
**Related:** [Overview](./README.md) · [Function specifications](./FUNCTION-SPECIFICATIONS.md)

> Tests depending on open decisions are policy-dependent until decisions are approved. Security/scope isolation tests remain mandatory. The prior forensic reconstruction is legacy evidence, not target behavior.

## Test fixtures
- Student A/B with separate progress and tenant records.
- SUPER_ADMIN, HOD-A/HOD-B (different departments), MANAGER-A/MANAGER-B (different sections/groups), FINANCE_TEAM, and any faculty role only if approved.
- Paths with room, optional, non-room, skipped, completed, zero-step, reordered, deleted, and missing-content steps.
- Room progress states and duplicate/out-of-order outbox events.
- Assessment attempts, score thresholds, event registrations/withdrawals/disqualifications/cancellations/ties, contest results, and rating histories.
- Dashboard fixtures with zero, partial, stale, delayed, and failed metrics.
- Analytics data across tenant/department/section and cohort sizes.
- Rating leaderboard pagination, tie, no-rating, and correction fixtures.
- Export fixtures with sensitive fields, large volumes, and retention/legal-hold states.

## Cases

| Test ID | Function | Scenario | Preconditions / steps | Expected result | Trace |
| ANA-QA-001 | ANA-FF001 | Student opens personal progress list | Authenticated student with paths → Open My Progress | Only own authorized paths shown | F001 |
| ANA-QA-002 | ANA-FF001 | Attempt to query another user’s progress | Student A authenticated → Change user ID in request | Server ignores client identity or denies; Student B data never returned | F001 |
| ANA-QA-003 | ANA-FF001 | Student has no paths | Authenticated student with no progress → Open My Progress | Approved empty state; not a server error | F001 |
| ANA-QA-004 | ANA-FF001 | Progress API unavailable | API failure fixture → Open My Progress | Error state; mock values not presented as live | F001 |
| ANA-QA-005 | ANA-FF001 | Path inclusion rules | Completed/active/archived paths seeded → Load list | Returned path set/order matches approved inclusion policy | F001 |
| ANA-QA-006 | ANA-FF001 | Tenant context mismatch | User scoped to tenant A → Request tenant B data | No cross-tenant progress returned | F001 |
| ANA-QA-007 | ANA-FF002 | View own path detail | Student with path progress → Open path detail | Correct progress values returned | F002 |
| ANA-QA-008 | ANA-FF002 | Guess another learner’s path ID | Student A; path progress belongs to B → Request detail | Denied/safe not-found; no disclosure | F002 |
| ANA-QA-009 | ANA-FF002 | Path does not exist | Authenticated student → Request nonexistent path | Defined not-found response | F002 |
| ANA-QA-010 | ANA-FF002 | Progress record absent | Accessible path without progress row → Open detail | Approved missing/initialization behavior | F002 |
| ANA-QA-011 | ANA-FF002 | Percentage calculation boundary | Progress 0, partial, complete → Load detail | Percent matches approved formula/rounding | F002 |
| ANA-QA-012 | ANA-FF002 | Aggregation stale | Progress event pending → Open path detail | Freshness/consistency contract respected | F002 |
| ANA-QA-013 | ANA-FF003 | Resolve next room | Path has ordered incomplete rooms → Continue | Correct next eligible room returned | F003 |
| ANA-QA-014 | ANA-FF003 | Path fully completed | All required steps complete → Continue | Terminal/no-next-step state; not generic error | F003 |
| ANA-QA-015 | ANA-FF003 | Skipped/optional earlier step | Path includes skip/optional → Continue | Resolver follows approved eligibility semantics | F003 |
| ANA-QA-016 | ANA-FF003 | Path steps reordered | Progress exists; step ordering changed → Resolve next step | No index drift; canonical order applied | F003 |
| ANA-QA-017 | ANA-FF003 | Inaccessible path | Student not enrolled/authorized → Resolve next step | Denied/not-found per policy | F003 |
| ANA-QA-018 | ANA-FF003 | Step content missing | Next step references deleted content → Continue | Defined recovery/error; no invalid navigation | F003 |
| ANA-QA-019 | ANA-FF004 | Start room | Student authorized for room → Start | State becomes in_progress per approved transition | F004 |
| ANA-QA-020 | ANA-FF004 | Complete room | Student authorized; state eligible → Complete | State/timestamp stored; progress event emitted as required | F004 |
| ANA-QA-021 | ANA-FF004 | Skip room | Student authorized → Skip | Skipped state recorded; certificate completion not granted by default | F004 |
| ANA-QA-022 | ANA-FF004 | Room belongs to another tenant | Student in tenant A; room tenant B → Attempt complete | Denied; no progress mutation | F004 |
| ANA-QA-023 | ANA-FF004 | Room not in enrolled path | Room exists but no eligible path membership → Attempt skip/complete | Denied; no progress mutation | F004 |
| ANA-QA-024 | ANA-FF004 | Client forges user ID | Authenticated student A → Submit payload userId=B | Actor remains A; request cannot mutate B’s progress | F004 |
| ANA-QA-025 | ANA-FF004 | Repeat complete request | Same action submitted twice → Repeat completion | Idempotent or defined duplicate response; no corrupt state | F004 |
| ANA-QA-026 | ANA-FF004 | Invalid room transition | State disallows requested action → Submit transition | Validation/state conflict response; unchanged data | F004 |
| ANA-QA-027 | ANA-FF004 | Persistence failure | DB write fails → Complete room | No false success; failure policy applied | F004 |
| ANA-QA-028 | ANA-FF004 | Progress event/outbox failure | Room state write succeeds; event insert fails → Complete room | Atomicity/recovery follows approved outbox policy; no silent loss | F004 |
| ANA-QA-029 | ANA-FF005 | Aggregate ordinary progress | Room states include complete and in-progress → Trigger aggregator | Path aggregate matches approved counting rules | F005 |
| ANA-QA-030 | ANA-FF005 | Duplicate event replay | Same RoomProgressUpdated delivered twice → Consume twice | Idempotent aggregate; no double counting | F005 |
| ANA-QA-031 | ANA-FF005 | Out-of-order progress events | Old/new room states/events → Consume in reverse order | Final aggregate reconciles with canonical room states | F005 |
| ANA-QA-032 | ANA-FF005 | Skipped room in aggregation | Room state skipped → Aggregate | Skip semantics follow approved policy; no unauthorized certificate eligibility | F005 |
| ANA-QA-033 | ANA-FF005 | Optional/non-room path step | Path contains optional/non-room steps → Aggregate | Denominator/count rules follow approved decision | F005 |
| ANA-QA-034 | ANA-FF005 | Zero-step path | Path has zero eligible steps → Aggregate | Defined zero-step behavior; no stale completion silently retained | F005 |
| ANA-QA-035 | ANA-FF005 | Path changes after learner progress | Step added/removed/reordered → Reaggregate | Approved recalculation and historical semantics applied | F005 |
| ANA-QA-036 | ANA-FF005 | Room row missing | Step has no room-progress row → Aggregate | Missing room state handled as defined incomplete/unknown | F005 |
| ANA-QA-037 | ANA-FF005 | Consumer database failure | DB temporarily unavailable → Consume event | Retry/backoff and error visibility follow Module 18 | F005 |
| ANA-QA-038 | ANA-FF005 | Outbox retry exhaustion | Consumer repeatedly fails → Observe job | Dead-letter/alert/recovery follows approved policy | F005 |
| ANA-QA-039 | ANA-FF005 | Path completion signal | Progress meets approved completion rule → Aggregate | Completion emitted once and certificate eligibility separately checked | F005 |
| ANA-QA-040 | ANA-FF006 | Dashboard with populated metrics | Student has learning data → Open dashboard | Metrics reflect approved authoritative data | F006 |
| ANA-QA-041 | ANA-FF006 | Dashboard with zero activity | Student has no activity → Open dashboard | Zero state distinguished from missing data | F006 |
| ANA-QA-042 | ANA-FF006 | Dashboard service loading | API slow → Open dashboard | Loading state shown; no misleading values | F006 |
| ANA-QA-043 | ANA-FF006 | Dashboard service failure | API unavailable → Open dashboard | Error/retry state; stale/mock data not represented as current | F006 |
| ANA-QA-044 | ANA-FF006 | Dashboard data privacy | Two students with different activity → Student A loads dashboard | Only A’s data shown | F006 |
| ANA-QA-045 | ANA-FF006 | Dashboard freshness | Metrics are delayed → Load dashboard | As-of/freshness behavior follows approved contract | F006 |
| ANA-QA-046 | ANA-FF006 | Dashboard KPI formula | Seed values with known totals → Load KPI | Formula, period, timezone and rounding match approved metric definition | F006 |
| ANA-QA-047 | ANA-FF007 | Assessment analytics for authorized viewer | Assessment results seeded → Open analytics | Metrics match canonical score/grade data | F007 |
| ANA-QA-048 | ANA-FF007 | Student accesses own assessment analytics | Student with own attempts → Open view | Only own attempts/results exposed | F007 |
| ANA-QA-049 | ANA-FF007 | Staff queries out-of-scope assessment | HOD/Manager scope mismatch → Request analytics | Denied or no unauthorized data | F007 |
| ANA-QA-050 | ANA-FF007 | Multiple attempts per learner | Learners have retakes → Aggregate assessment | Attempt inclusion follows approved attempt policy | F007 |
| ANA-QA-051 | ANA-FF007 | Pass-rate boundary | Scores at pass threshold → Aggregate | Pass/fail calculation matches approved threshold semantics | F007 |
| ANA-QA-052 | ANA-FF007 | Small cohort data | Cohort below privacy threshold → Open analytics | Mask/aggregate according to privacy decision | F007 |
| ANA-QA-053 | ANA-FF007 | Assessment analytics source failure | Data service fails → Open analytics | Defined error; no client fabricated metric | F007 |
| ANA-QA-054 | ANA-FF008 | Event analytics authorized | Event has registrations and results → Open analytics | Counts and summaries match canonical event records | F008 |
| ANA-QA-055 | ANA-FF008 | Cross-tenant event analytics | Viewer outside event tenant → Request analytics | Access denied/no disclosure | F008 |
| ANA-QA-056 | ANA-FF008 | Withdrawn participant | Event includes withdrawal → Aggregate | Withdrawn status handled according to Module 11/decision | F008 |
| ANA-QA-057 | ANA-FF008 | Disqualified participant | Event includes disqualified participant → Aggregate ranks/counts | Disqualification treatment follows approved policy | F008 |
| ANA-QA-058 | ANA-FF008 | Cancelled event | Event cancelled → Open analytics | Cancelled event state is reflected correctly | F008 |
| ANA-QA-059 | ANA-FF008 | Tied contest ranks | Participants tied → Aggregate rank metrics | Tie behavior follows Module 11 and rating policy | F008 |
| ANA-QA-060 | ANA-FF008 | Late submission | Submission after deadline → Aggregate event | Late result inclusion follows event rules | F008 |
| ANA-QA-061 | ANA-FF009 | Global analytics admin access | SUPER_ADMIN authenticated → Open platform analytics | Authorized global aggregate returned | F009 |
| ANA-QA-062 | ANA-FF009 | Non-admin attempts global analytics | Student/Manager authenticated → Request global analytics | Denied unless explicitly approved role | F009 |
| ANA-QA-063 | ANA-FF009 | Date range boundary | Events on start/end dates → Query period | Timezone and inclusivity match approved contract | F009 |
| ANA-QA-064 | ANA-FF009 | Growth bucket calculation | Known date-indexed records → Query growth chart | Bucket values match approved definition | F009 |
| ANA-QA-065 | ANA-FF009 | Large platform data volume | Large fixture dataset → Load analytics | Server-side aggregation/limits meet agreed performance target | F009 |
| ANA-QA-066 | ANA-FF009 | Global metric contains PII | Underlying records contain user details → Request aggregate | Only approved aggregate fields returned | F009 |
| ANA-QA-067 | ANA-FF009 | Global analytics query fails | DB/API error → Load analytics | Distinct error state; no misleading partial values | F009 |
| ANA-QA-068 | ANA-FF010 | HOD cohort analytics | HOD assigned department → Open cohort dashboard | Only department-authorized cohorts/data returned | F010 |
| ANA-QA-069 | ANA-FF010 | Manager cohort analytics | Manager assigned section → Open cohort dashboard | Only permitted section/group data returned | F010 |
| ANA-QA-070 | ANA-FF010 | HOD attempts another department | Department B cohort → Query | No cross-department data | F010 |
| ANA-QA-071 | ANA-FF010 | Manager attempts another section | Section B cohort outside scope → Query | No cross-section data | F010 |
| ANA-QA-072 | ANA-FF010 | Faculty role not mapped | User with undefined faculty role → Request analytics | Denied until explicit role permission exists | F010 |
| ANA-QA-073 | ANA-FF010 | Student drill-down access | Staff opens learner detail → View student | Detail exposed only if explicit permission approved | F010 |
| ANA-QA-074 | ANA-FF010 | Small cohort privacy | Cohort below threshold → Open cohort metrics | Mask/suppress per approved threshold | F010 |
| ANA-QA-075 | ANA-FF010 | Cohort membership changes | Learner moved between sections → Query historical/current cohort | Membership time semantics follow approved definition | F010 |
| ANA-QA-076 | ANA-FF011 | Leaderboard with ratings | Eligible participants have ratings → Open leaderboard | Ordered by approved rating/rank policy | F011 |
| ANA-QA-077 | ANA-FF011 | Leaderboard empty | No eligible participants → Open leaderboard | Empty state, not API error | F011 |
| ANA-QA-078 | ANA-FF011 | Leaderboard query failure | Service unavailable → Open leaderboard | Error state not shown as empty leaderboard | F011 |
| ANA-QA-079 | ANA-FF011 | Tie rating values | Several equal ratings → Load leaderboard | Tie rank/order follows approved deterministic rule | F011 |
| ANA-QA-080 | ANA-FF011 | Pagination stable | Many eligible records → Page through leaderboard | No duplicates/omissions under agreed cursor consistency | F011 |
| ANA-QA-081 | ANA-FF011 | Leaderboard privacy | Participant profile contains private fields → Load leaderboard | Only approved display fields shown | F011 |
| ANA-QA-082 | ANA-FF011 | Tenant/global scope | Tenant-scoped rating exists → Query global or tenant leaderboard | Scope matches permission and policy | F011 |
| ANA-QA-083 | ANA-FF012 | View own standing | Student has rating → Open My Standing | Correct rating/rank in approved scope | F012 |
| ANA-QA-084 | ANA-FF012 | No rating yet | Student has no rating → Open My Standing | No-rating state, not misleading 404 or error | F012 |
| ANA-QA-085 | ANA-FF012 | Guess another user standing | Student A requests B → Query standing | Denied or self-only behavior enforced | F012 |
| ANA-QA-086 | ANA-FF012 | Unknown person | Invalid person ID → Query standing | Not-found distinct from no-rating/forbidden | F012 |
| ANA-QA-087 | ANA-FF012 | Standing differs from leaderboard | Rating/eligibility changes → Compare endpoints | Same scope/eligibility/ranking logic used | F012 |
| ANA-QA-088 | ANA-FF012 | Standing privacy | User profile includes private attributes → View standing | Only allowed fields returned | F012 |
| ANA-QA-089 | ANA-FF013 | Event completion rating update | Eligible event finalized → Run rating consumer | Rating updated using approved algorithm/config | F013 |
| ANA-QA-090 | ANA-FF013 | Duplicate completion event | EventCompleted replayed → Process twice | Rating not double-applied | F013 |
| ANA-QA-091 | ANA-FF013 | Tie rank event | Final ranks include tie → Update ratings | Tie scoring follows approved policy | F013 |
| ANA-QA-092 | ANA-FF013 | Event not rating-eligible | Practice/non-platform event → Process completion | Rating unchanged or adjusted per eligibility policy | F013 |
| ANA-QA-093 | ANA-FF013 | Event result correction | Final rank corrected → Reprocess/reconcile | Correction policy prevents inconsistent rating history | F013 |
| ANA-QA-094 | ANA-FF013 | Cancelled event | Event cancelled after rating job → Process cancellation | Rating rollback/reversal follows policy | F013 |
| ANA-QA-095 | ANA-FF013 | Redis/cache update failure | Rating DB write succeeds; cache fails → Update rating | Cache rebuild/invalidation behavior works; DB remains source of truth | F013 |
| ANA-QA-096 | ANA-FF013 | Rating config/version change | Algorithm config updated → Process event | Algorithm version and historical behavior are traceable | F013 |
| ANA-QA-097 | ANA-FF014 | Authorized export | Role has export permission → Request scoped export | Only permitted rows/fields exported | F014 |
| ANA-QA-098 | ANA-FF014 | Export without permission | User lacks export permission → Request export | Denied; no artifact generated | F014 |
| ANA-QA-099 | ANA-FF014 | Export cross-tenant attempt | Scoped user supplies foreign tenant filter → Export | Server scope prevents foreign records | F014 |
| ANA-QA-100 | ANA-FF014 | Sensitive metadata in export | Fixture includes PII/secrets → Export | Fields masked/excluded per approved privacy policy | F014 |
| ANA-QA-101 | ANA-FF014 | Export exceeds row/file limit | Large data set → Request export | Limit/async job behavior applied | F014 |
| ANA-QA-102 | ANA-FF014 | Export artifact expires | Expired export link/file → Attempt download | Expired artifact inaccessible | F014 |
| ANA-QA-103 | ANA-FF014 | Retention expiry | Records exceed retention period → Run retention process | Archive/purge according to approved retention/legal hold policy | F014 |
| ANA-QA-104 | ANA-FF014 | Export audit trail | Audit of export enabled → Generate/download export | Module 15 audit event recorded with safe metadata | F014 |

## Coverage and gating
- **Total cases:** 104 (ANA-QA-001 through ANA-QA-104; sequential).
- Covers personal progress, path detail/next step, room mutations, aggregation, dashboards, assessment/event/platform/cohort analytics, leaderboard/standing, ELO rating, export/privacy/retention.
- ANA-D01–D22 and dependent Module 11/12/17/18 decisions must be reflected in expected outcomes before implementation acceptance.
- Tests for cross-tenant, cross-department, cross-section, student self-only access, and secret/PII protection are mandatory regardless of unresolved metric definitions.
