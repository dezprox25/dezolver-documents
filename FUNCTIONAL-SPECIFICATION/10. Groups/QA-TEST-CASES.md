# DEZOLVER — Module 10 Groups & Membership QA Test Cases

**Version:** 0.2 — companion to Functional Specification Standard v1.0  
**Status:** Detailed QA draft; product/security decisions remain open  
**Related:** [Module 10 README](./README.md) · [Function-Level Specifications](./FUNCTION-SPECIFICATIONS.md)

> QA cases define verification coverage. Where policy is unresolved, the expected result is explicitly decision-blocked; this document does not approve behavior.

## Test actors and fixtures
- SUPER_ADMIN: global.
- FINANCE-A: College A.
- HOD-A: College A / Department A.
- HOD-B: College A / Department B.
- Manager-A: assigned Section A; Group authorization mapping absent unless test setup explicitly grants it.
- Manager-B: different Section/Group scope.
- Student-A: eligible Student in College A / Department A.
- Student-B: another Student in Department A.
- Student-C: Student from another Department/College.
- Group-A: active, College A / Department A.
- Group-B: active, Department B or foreign College.
- Group-Inactive: isActive=false.
- Lab-A / Contest-A: activity within Group-A scope.
- Lab-B / Contest-B: activity outside Group-A scope.
- Course-A / Problem-A: permitted relation.
- Problem-B: unrelated/out-of-scope Problem.

## QA cases

| Test ID | Function | Scenario / steps | Expected result | Trace/status |
|---|---|---|---|---|
| GRP-QA-001 | F001 | HOD-A lists Groups | Only College A / Department A records | RBAC/source |
| GRP-QA-002 | F001 | HOD-A submits Department B filter | Foreign Groups not returned | RBAC/source |
| GRP-QA-003 | F001 | Manager-A without Group authorization mapping lists Groups | No guessed/inferred Group access | GRP-D03 |
| GRP-QA-004 | F001 | Student-A opens My Groups | Own/permitted membership only | Mapping/RBAC |
| GRP-QA-005 | F001 | Client sends foreign collegeId/departmentId | Cannot widen results | RBAC |
| GRP-QA-006 | F002 | HOD-A opens Group-A | Permitted detail returned | RBAC |
| GRP-QA-007 | F002 | Manager-A opens unassigned Group-B | Denied/safe not-found | GRP-D03 |
| GRP-QA-008 | F002 | Student-A opens own Group-A | Student-safe detail | Membership |
| GRP-QA-009 | F002 | Group detail payload includes inviteCode | Code must not appear in generic detail | Security recommendation |
| GRP-QA-010 | F003 | HOD-A creates valid Group in Department A | Group created in authorized scope | RBAC/schema |
| GRP-QA-011 | F003 | Student-A attempts create | Denied | RBAC |
| GRP-QA-012 | F003 | HOD-A submits Department B ID | Denied; no Group created | RBAC |
| GRP-QA-013 | F003 | Name blank or >255 chars | Reject per approved required/length validation; no record | Schema/recommendation |
| GRP-QA-014 | F003 | Create duplicate Group name in same Department | Apply GRP-D05; decision-blocked until approved | Decision |
| GRP-QA-015 | F004 | HOD-A updates Group-A name/description | Approved fields persist | CRUD |
| GRP-QA-016 | F004 | Manager-A edits Group-B outside permitted scope | Denied | RBAC |
| GRP-QA-017 | F004 | Client attempts to update collegeId/createdById | Reject/ignore; scope and creator remain server-controlled | Security |
| GRP-QA-018 | F004 | Concurrent metadata update | Shared conflict policy applied; no silent overwrite unless approved | Module 18 |
| GRP-QA-019 | F005 | HOD-A deactivates Group-A | isActive becomes false if transition permitted | Schema/recommendation |
| GRP-QA-020 | F005 | Student joins inactive Group | Reject if inactive Group is non-joinable; decision pending | GRP-D02/D08 |
| GRP-QA-021 | F005 | Deactivate already inactive Group | Idempotent/no-op behavior per approved policy | Decision |
| GRP-QA-022 | F005 | Deactivate Group during active Contest/Assessment | Apply owning-module lifecycle decision; unresolved | GRP-D07/D09/D11 |
| GRP-QA-023 | F006 | HOD-A regenerates Group invite code | New code returned only to authorized staff; old-code invalidation per decision | GRP-D08 |
| GRP-QA-024 | F006 | Student requests Group invite code | Denied/no code disclosure | Security |
| GRP-QA-025 | F006 | Generated invite code length checked | ≤20 characters | Schema |
| GRP-QA-026 | F006 | Code collision occurs | Generation retry/uniqueness policy applied; unresolved mechanism | Decision |
| GRP-QA-027 | F007 | Student-A joins Group-A using valid code | One GroupMember created | Join workflow proposal |
| GRP-QA-028 | F007 | Student submits invalid invite code | No membership; no Group detail leakage | Security |
| GRP-QA-029 | F007 | Student-A sends Student-B ID with valid code | Authenticated Student-A identity used; no impersonation | RBAC |
| GRP-QA-030 | F007 | Student-A joins Group-A twice | No duplicate membership | GRP-D06 |
| GRP-QA-031 | F007 | Student submits old code after regeneration | Reject if rotation invalidates old code | GRP-D08 |
| GRP-QA-032 | F008 | HOD-A adds eligible Student-A to Group-A | Membership created | Scope |
| GRP-QA-033 | F008 | Manager-A removes member from unassigned Group | Denied | GRP-D03 |
| GRP-QA-034 | F008 | Student-A attempts to remove Student-B | Denied | RBAC |
| GRP-QA-035 | F008 | Student-A leaves own Group | Applies approved leave/removal model; currently decision-blocked | GRP-D07 |
| GRP-QA-036 | F008 | Member removed with historical submission/results | Student-owned history preserved under owning-module rules | Cross-module |
| GRP-QA-037 | F008 | Staff adds same member concurrently twice | Only one active membership under approved uniqueness policy | GRP-D06 |
| GRP-QA-038 | F009 | HOD-A links Lab-A to Group-A | GroupLab relation created | Schema |
| GRP-QA-039 | F009 | Manager-A links foreign Contest-B | Denied | RBAC |
| GRP-QA-040 | F009 | Duplicate Lab association | No duplicate relation if uniqueness policy approved | Decision |
| GRP-QA-041 | F009 | Group–Contest link is created | Must not automatically create ContestParticipant, score, or rank | Schema/behavior boundary |
| GRP-QA-042 | F010 | HOD-A associates permitted Problem-A/Course-A with Group-A | GroupCourseProblem created | Schema |
| GRP-QA-043 | F010 | Manager-A associates foreign Problem-B | Denied | RBAC |
| GRP-QA-044 | F010 | Problem not attached/available to Course | Expected behavior decision-blocked | GRP-D10 |
| GRP-QA-045 | F010 | Duplicate GroupCourseProblem relation | Prevent duplicate if approved | Decision |
| GRP-QA-046 | F011 | Student-A opens permitted Group-linked Lab | Show only if Lab module permits access | Module 8 |
| GRP-QA-047 | F011 | Student-A requests hidden Contest/Problem resource | Owning module denies/filters | RBAC |
| GRP-QA-048 | F011 | Assessment has groupId=Group-A | Display/eligibility follows Module 9; no inferred group attempts | GRP-D11 |
| GRP-QA-049 | F011 | Student-A requests Student-B attempt/result from Group context | Denied | Ownership |
| GRP-QA-050 | F001 | Finance-A requests Group list | Only approved read-only College A view; fields TBD | GRP-D12 |
| GRP-QA-051 | F001 | Student-A searches all Group names | Directory behavior follows approved discovery rule | GRP-D04 |
| GRP-QA-052 | F001 | Student-A has no memberships | Empty state without exposing Group counts | Privacy |
| GRP-QA-053 | F001 | Client supplies Manager role/scope in request body | Authenticated role/scope is authoritative | RBAC |
| GRP-QA-054 | F001 | HOD-A submits invalid status/sort/page filter | Reject or safe default per approved contract | UI/API TBD |
| GRP-QA-055 | F001 | Group membership changes after list fetch | Subsequent detail/action rechecks authorization | Security |
| GRP-QA-056 | F001 | Search term matches both local and foreign Group | Only local authorized results returned | Tenant isolation |
| GRP-QA-057 | F002 | Group has no members | Empty member section | UX TBD |
| GRP-QA-058 | F002 | Group has no activities | Empty activities section | UX TBD |
| GRP-QA-059 | F002 | Group inactive | Visibility follows approved staff/Student policy | GRP-D02 |
| GRP-QA-060 | F002 | Linked activity is archived | Follow owning module access | Cross-module |
| GRP-QA-061 | F002 | Student is former member | Detail access follows approved retention/visibility rule | GRP-D07 |
| GRP-QA-062 | F003 | HOD has no Department assignment | Create denied until scope is valid | Module 2 |
| GRP-QA-063 | F003 | Manager has section but no Group assignment | Do not infer create scope | GRP-D03/D16 |
| GRP-QA-064 | F003 | Description exceeds proposed limit | Enforce only if limit is approved; otherwise TBD | Decision |
| GRP-QA-065 | F003 | Client supplies custom inviteCode | Server-generated code used; reject/ignore override | Security |
| GRP-QA-066 | F003 | Group create retry after network timeout | Duplicate prevention/idempotency policy applied | Module 18 |
| GRP-QA-067 | F003 | Department deactivated while form open | Revalidate on submit; deny if no longer eligible | Module 2 |
| GRP-QA-068 | F003 | Concurrent same-name Group create | Apply GRP-D05/unique policy; no accidental overwrite | Decision |
| GRP-QA-069 | F004 | Attempt Department transfer via edit form | Not allowed through general metadata edit | Recommendation |
| GRP-QA-070 | F004 | Update inactive Group metadata | Allow/deny per approved lifecycle policy | Decision |
| GRP-QA-071 | F004 | No-op update | No unintended side effects; timestamp policy TBD | Module 18 |
| GRP-QA-072 | F004 | Description contains HTML/script-like text | Safe handling/rendering; exact policy TBD | Security |
| GRP-QA-073 | F004 | Update fails after validation | No partial metadata change | Reliability |
| GRP-QA-074 | F005 | Activate inactive Group | isActive=true if authorized and permitted | Lifecycle |
| GRP-QA-075 | F005 | Deactivate Group with members | Membership rows preserved under recommended rule | GRP-D02 |
| GRP-QA-076 | F005 | Deactivate Group with linked Lab/Contest | Links/history preserved; activity effect decision-blocked | GRP-D09 |
| GRP-QA-077 | F005 | Join and deactivate race | Final state must not permit unauthorized join after inactive transition | Concurrency |
| GRP-QA-078 | F005 | Two staff change state concurrently | Shared conflict/idempotency policy | Module 18 |
| GRP-QA-079 | F005 | Unauthorized Student activates Group | Denied | RBAC |
| GRP-QA-080 | F006 | Generic Group list/detail queried for inviteCode | Code omitted | Security |
| GRP-QA-081 | F006 | Finance requests code | Denied | GRP-D12 |
| GRP-QA-082 | F006 | Manager requests code without Group mapping | Denied | GRP-D03 |
| GRP-QA-083 | F006 | Code generation returns >20 chars | Reject/regenerate before persist | Schema |
| GRP-QA-084 | F006 | Concurrent regenerate requests | One coherent current code; concurrency policy TBD | Module 18 |
| GRP-QA-085 | F006 | Audit event emitted for regeneration | Raw code must not appear in logs/metadata | Privacy |
| GRP-QA-086 | F007 | Student submits blank/whitespace code | Reject | Validation |
| GRP-QA-087 | F007 | Student submits code >20 chars | Reject | Schema |
| GRP-QA-088 | F007 | Group inactive at join commit | Reject if inactive join policy approved | GRP-D02 |
| GRP-QA-089 | F007 | Student account suspended/deactivated | Deny per Module 1 policy | Auth |
| GRP-QA-090 | F007 | Student from foreign College attempts code join | Deny unless cross-college rule explicitly approved | GRP-D06 |
| GRP-QA-091 | F007 | Student from another Department attempts join | Apply approved cross-department rule; currently TBD | GRP-D06 |
| GRP-QA-092 | F007 | Code expires/revokes | Behavior decision-blocked; no expiry/revoke fields in schema | GRP-D08 |
| GRP-QA-093 | F007 | Student submits code while rate limit threshold reached | Apply approved anti-abuse controls; threshold TBD | Security |
| GRP-QA-094 | F007 | Group is deactivated/regenerated between lookup and membership write | Recheck state/code atomically; no stale join | Concurrency |
| GRP-QA-095 | F008 | HOD adds Student from another Department | Deny unless cross-department membership approved | GRP-D06 |
| GRP-QA-096 | F008 | HOD removes non-member | No mutation; safe feedback | Membership |
| GRP-QA-097 | F008 | Manager adds Student outside Group scope | Deny | RBAC |
| GRP-QA-098 | F008 | Staff removes member during active Contest | Follow Contest access/attempt policy; unresolved | Module 11 |
| GRP-QA-099 | F008 | Staff removes member during Assessment attempt | Follow Module 9 active-attempt policy; unresolved | Module 9 |
| GRP-QA-100 | F008 | Student leaves while active submission/attempt exists | Preserve history; access transition decision required | GRP-D07 |
| GRP-QA-101 | F008 | Student leaves then rejoins | Membership history/rejoin policy TBD | GRP-D17 |
| GRP-QA-102 | F008 | Student requests another student's leave action | Deny | Ownership |
| GRP-QA-103 | F008 | GroupMember record missing during removal | Safe no-op/not-found; no unrelated deletion | Reliability |
| GRP-QA-104 | F008 | Removal persistence succeeds but audit write fails | Completion/audit atomicity policy Module 15 | Reliability |
| GRP-QA-105 | F008 | Removed member's previous Lab/Contest result requested | Access follows owning module and historical record policy | Cross-module |
| GRP-QA-106 | F009 | HOD assigns Lab from foreign Department | Deny | RBAC |
| GRP-QA-107 | F009 | HOD assigns Contest from foreign College | Deny | Tenant scope |
| GRP-QA-108 | F009 | Group inactive at assignment | Deny if inactive association policy approved | GRP-D02 |
| GRP-QA-109 | F009 | Lab due date has passed | Assignment policy TBD | Module 8 |
| GRP-QA-110 | F009 | Contest already live | Assignment policy TBD; no inferred access changes | Module 11 |
| GRP-QA-111 | F009 | Contest ended | Assignment policy TBD | Module 11 |
| GRP-QA-112 | F009 | Unassign Lab after Student submission exists | Preserve submission; association removal policy TBD | Module 8 |
| GRP-QA-113 | F009 | Unassign Contest after participation exists | Preserve participant/score/rank; policy TBD | Module 11 |
| GRP-QA-114 | F009 | Activity assigned to multiple Groups | Supported by relationship model if approved; semantics TBD | Decision |
| GRP-QA-115 | F009 | Concurrent assign/unassign request | Prevent inconsistent association state | Module 18 |
| GRP-QA-116 | F009 | Association creation request retried | No duplicate relation if idempotency/uniqueness approved | Module 18 |
| GRP-QA-117 | F010 | Course and Problem in different scopes | Deny if incompatible under approved relation rules | GRP-D10 |
| GRP-QA-118 | F010 | CourseProblem association does not exist | Apply approved GroupCourseProblem semantics; decision-blocked | GRP-D10 |
| GRP-QA-119 | F010 | Problem hidden/private | Do not expose through Group association alone | Module 5 |
| GRP-QA-120 | F010 | Group inactive | Association behavior follows approved inactive rule | GRP-D02 |
| GRP-QA-121 | F010 | Student not enrolled in Course | Group link must not bypass Course enrollment rules | Module 4 |
| GRP-QA-122 | F010 | Duplicate GroupCourseProblem relation | Prevent duplicates if approved | Decision |
| GRP-QA-123 | F010 | Problem removed from Course after Group link | Visibility/cleanup follows Module 4 decision | Cross-module |
| GRP-QA-124 | F010 | Unassociate Course Problem after submission | Preserve submission/result; policy TBD | Module 4/6 |
| GRP-QA-125 | F010 | Client supplies foreign courseId/problemId | Backend scope and relationship checks deny | Security |
| GRP-QA-126 | F011 | Group has Assessment.groupId | Assessment visibility/eligibility follows Module 9 | GRP-D11 |
| GRP-QA-127 | F011 | Student joins after Assessment start | Eligibility timing follows Module 9 | GRP-D11 |
| GRP-QA-128 | F011 | Student removed during Assessment attempt | Active attempt behavior follows Module 9 | GRP-D07 |
| GRP-QA-129 | F011 | Group-linked activity includes hidden testcases | Hidden data not returned | Module 7 |
| GRP-QA-130 | F011 | Student views another member's source code | Deny absent explicit source-code permission | RBAC |
| GRP-QA-131 | F011 | Group membership used to request another Student's score | Deny unless staff permission/owner module allows | Ownership |
| GRP-QA-132 | F011 | Owning module unavailable | Apply shared partial-load/error behavior; TBD | Module 18 |

## QA coverage summary
- 132 cases cover GRP-F001–GRP-F011.
- Coverage includes role/scope, CRUD, membership, invite code, activity associations, privacy, cross-module access, concurrency, retries, and schema gaps.
- Decision-blocked expected results remain intentionally open for later product review.
