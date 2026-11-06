# Module 4 — Course & Enrollment QA Test Cases

**Document ID:** DEZ-COURSE-QA-004  
**Version:** 0.1 — Standard v1.0 QA companion  
**Status:** Draft; policy-dependent cases are blocked until the linked decision is approved  
**Specification:** [Course & Enrollment](./README.md)

## Execution conventions

- All role/scope tests use isolated College A and College B, Departments A1/B1, Sections A1-1/B1-1, and users HOD-A, Manager-A1-1, Student-A1-1, Finance-A, SUPER_ADMIN.
- Test data includes Course C-A (College A/Department A1), Course C-B (College B/Department B1), CourseSection links, CourseEnrollment records, CourseProblem links, and separate GroupCourseProblem.
- Product behavior is governed by README Sections 24–26 and ADOPTED-DECISIONS.md. These are target expectations, not evidence that implementation passed.
- Every test enforces server-side identity/College/Department/Section scope. Module 3 entitlement is authoritative for subscription and seat checks.

## Test cases

| Test ID | Scenario | Preconditions / data | Steps | Expected result | Trace / status |
|---|---|---|---|---|---|
| COURSE-QA-001 | HOD lists Courses in own Department | HOD-A; C-A/C-B exist | Open Courses & Labs | C-A in permitted results; C-B excluded | F001; source scope |
| COURSE-QA-002 | HOD attempts foreign Course detail | HOD-A; C-B | Request C-B by ID | Denied/not-found; no C-B fields disclosed | F001; RBAC |
| COURSE-QA-003 | Manager sees Course assigned to own Section | Manager-A1-1; CourseSection C-A→A1-1 | Open Course list | Course appears only if approved visibility rule permits | F001; COURSE-D07/COURSE-D21 |
| COURSE-QA-004 | Manager requests unrelated Section Course | Manager-A1-1; Course linked B1-1 | Request list/detail | No foreign Section Course data returned | F001; RBAC |
| COURSE-QA-005 | Student views own enrolled Course | Student-A1-1 enrolled in C-A | Open My Courses | C-A shown under approved enrollment rule | F001/F013 |
| COURSE-QA-006 | Student requests another Student enrollment | Student-A1-1; enrollment belongs to Student-B | Direct API/URL | Denied; no private enrollment data | F001/F015; RBAC |
| COURSE-QA-007 | Finance Team views Course | Finance-A; RBAC View exists | Open Course route/API | Backend view permission honored; route behavior requires product decision | F001; COURSE-D04 |
| COURSE-QA-008 | SUPER_ADMIN views global Course | SUPER_ADMIN; C-A/C-B | Search Courses | Global Course data visible per RBAC | F001 |
| COURSE-QA-009 | Client attempts to widen College filter | HOD-A | Query with collegeId=B | Server scope remains College A; no College B records | F001; RBAC |
| COURSE-QA-010 | Invalid list query parameter | Course list API | Submit unsupported sort/filter | Rejected or handled per approved API contract; no scope bypass | F001/F022; adopted API contract |
| COURSE-QA-011 | HOD creates Course in own Department | HOD-A; valid form | Create C-A | Course saved with correct College/Department and creator | F002 |
| COURSE-QA-012 | HOD creates Course under foreign Department | HOD-A; Department B1 | Submit create | Denied/rejected; no Course record | F002; derived |
| COURSE-QA-013 | SUPER_ADMIN creates Course in College B | SUPER_ADMIN | Create C-B | Global permission allows creation if required fields valid | F002; RBAC |
| COURSE-QA-014 | Manager creates Course in assigned scope | Manager-A1-1; Manager Create* matrix | Submit Course | Allowed only if approved Manager create policy permits | F002; COURSE-D01/COURSE-D30 |
| COURSE-QA-015 | Manager creates Course in other Department | Manager-A1-1 | Submit Department B1 | Denied | F002; RBAC |
| COURSE-QA-016 | Client forges createdById | HOD-A; request sets creator to other user | Create Course | Backend uses authenticated actor or rejects forged field | F002; derived |
| COURSE-QA-017 | Course title exceeds varchar limit | Title >255 chars | Submit create | Server rejects; no Course saved | F002; schema |
| COURSE-QA-018 | Course code exceeds varchar limit | Code >50 chars | Submit create | Server rejects | F002; schema |
| COURSE-QA-019 | Invalid scope enum | scope=unsupported | Submit create | Server rejects | F002/F006 |
| COURSE-QA-020 | Invalid publication status | status=unsupported | Submit create | Server rejects | F002/F005 |
| COURSE-QA-021 | Update Course title in permitted scope | HOD-A; C-A exists | Edit title/save | Title and updatedAt change | F003 |
| COURSE-QA-022 | Update foreign Course | HOD-A; C-B | PATCH C-B | Denied; no mutation | F003 |
| COURSE-QA-023 | Manager updates Course metadata | Manager assigned Section | Update Course | Allowed only within approved permission/scope policy | F003; COURSE-D03/COURSE-D30 |
| COURSE-QA-024 | Update protected College/Department/creator field | HOD update payload | Attempt reparent/creator change | Rejected unless transfer feature explicitly approved | F003; COURSE-D32 |
| COURSE-QA-025 | Update code to duplicate | Duplicate code exists in relevant scope | Save duplicate | Outcome follows approved uniqueness/case policy | F003; COURSE-D06 |
| COURSE-QA-026 | Concurrent Course edits | Two authorized users edit same Course | Save conflicting edits | Approved conflict/version policy; no silent corruption | F003; COURSE-D40 |
| COURSE-QA-027 | Edit archived Course | Course ARCHIVED | Submit metadata update | Allow/deny per approved archive-edit policy | F003/F005; COURSE-D22 |
| COURSE-QA-028 | HOD archives Course with no history | Course exists, no linked records | Archive | Status/archive behavior follows approved rule | F004; COURSE-D23 |
| COURSE-QA-029 | Archive Course with enrollment/history | Course has enrollment/submissions | Archive | No linked history deleted; archive policy applied | F004/F025; COURSE-D23 |
| COURSE-QA-030 | Manager attempts Course delete | Manager role | Delete Course | Denied unless explicit permission decision grants it | F004; RBAC |
| COURSE-QA-031 | Hard delete Course with linked records | Course has CourseProblem/enrollments | Delete | No unapproved cascade/hard delete | F004/F025; COURSE-D33 |
| COURSE-QA-032 | Restore archived Course | Archived Course | Restore action | Restore only if approved; otherwise action unavailable/denied | F004; COURSE-D23 |
| COURSE-QA-033 | Archive already archived Course | Course ARCHIVED | Archive again | Idempotent/error behavior follows approved lifecycle | F004; COURSE-D28 |
| COURSE-QA-034 | Publish valid Course | Publish criteria approved; Course DRAFT | Publish | Status changes per approved transition | F005; COURSE-D28 |
| COURSE-QA-035 | Publish Course with missing required metadata | Publish checklist defined | Publish incomplete Course | Reject with validation intent; no status change | F005; COURSE-D28 |
| COURSE-QA-036 | Student opens DRAFT Course | Student not otherwise granted staff view | Direct URL | Access follows approved DRAFT visibility; no assumption | F005/F014; COURSE-D28 |
| COURSE-QA-037 | Student opens PUBLISHED Course | Student meets eligibility | Open Course | Access matches approved enrollment/scope rule | F005/F014; COURSE-D21/COURSE-D28 |
| COURSE-QA-038 | Student opens ARCHIVED Course | Existing enrollment | Open Course | Access/read/submission policy follows approved archive behavior | F005/F014; COURSE-D28 |
| COURSE-QA-039 | Invalid status transition | Current/target state forbidden by approved state machine | Request transition | Rejected; state unchanged | F005; Module 17 |
| COURSE-QA-040 | Set DEPT_WIDE scope | Course admin; scope policy approved | Save scope | Stored value and access follow approved DEPT_WIDE definition | F006; COURSE-D31 |
| COURSE-QA-041 | Set SECTION_ONLY without Section link | SECTION_ONLY policy approved | Save/publish | Behavior follows approved required-assignment rule | F006/F007; COURSE-D09/COURSE-D31 |
| COURSE-QA-042 | GLOBAL scope attempted across College | College-bound Course | Set GLOBAL | Cross-College exposure prohibited unless explicit decision permits | F006; COURSE-D31 |
| COURSE-QA-043 | Manager sets broader scope than assigned | Manager assigned one Section | Request DEPT_WIDE/GLOBAL | Denied or constrained by approved scope policy | F006; RBAC |
| COURSE-QA-044 | Change scope after Student enrollment | Course has enrolled Students | Update scope | Existing access follows approved impact policy; no silent history loss | F006/F025; COURSE-D32 |
| COURSE-QA-045 | Invalid scope string via API | Course update | Submit unsupported enum | Reject | F006 |
| COURSE-QA-046 | Course scope differs from Problem scope | Problem has restrictive visibility | Open Course problem | Problem’s own policy remains enforced; Course does not bypass it | F006/F012; Module 5 |
| COURSE-QA-047 | HOD assigns valid Section | HOD-A; Section A1-1 | Add CourseSection | Association created | F007 |
| COURSE-QA-048 | Assign Course to Section in foreign College | Course C-A; Section B1-1 | Add association | Reject | F007; derived |
| COURSE-QA-049 | Manager assigns outside Section scope | Manager-A1-1 | Assign Section B1-1 | Denied | F007; RBAC |
| COURSE-QA-050 | Duplicate CourseSection assignment | Link already exists | Add same link | Duplicate behavior follows approved uniqueness rule; no unintended duplicate | F007; COURSE-D29 |
| COURSE-QA-051 | SECTION_ONLY Course assigned multiple Sections | Scope policy unresolved | Add second Section | Behavior follows approved scope definition | F007; COURSE-D09/COURSE-D31 |
| COURSE-QA-052 | Section assignment auto-enrolls eligible Students | PUBLISHED Course; active Student roster in A1-1 | Create CourseSection C-A→A1-1 | One CourseEnrollment per eligible active Student after Module 3 entitlement check | COURSE-F007/F016/F017; COURSE-D11/COURSE-D13/COURSE-D25/COURSE-D29 |
| COURSE-QA-053 | Assign inactive/missing Section | Section unavailable | Submit link | Reject/eligibility response per Module 2 policy | F007; Module 2 |
| COURSE-QA-054 | Remove CourseSection link | Existing association | Remove | Association removed according to approved policy | F008; COURSE-D32 |
| COURSE-QA-055 | Remove Section link with active enrollments | Existing CourseEnrollment | Remove association | Enrollment/history not silently deleted | F008/F025 |
| COURSE-QA-056 | Remove foreign Section link | Actor out of scope | Request unlink | Denied | F008; RBAC |
| COURSE-QA-057 | Concurrent Section assignment/removal | Two staff actions | Perform conflicting actions | Approved concurrency behavior; no corrupt link state | F008; COURSE-D40 |
| COURSE-QA-058 | HOD views assigned Sections | CourseSection links valid | Open Course detail | All permitted Section associations shown | F009 |
| COURSE-QA-059 | Manager views Sections | Manager assigned Section A1-1 | Open Course details | Only permitted Section data shown | F009; RBAC |
| COURSE-QA-060 | Student views Section assignment | Student permitted Course | Open course detail | Only approved public Section metadata; no roster leak | F009; privacy |
| COURSE-QA-061 | HOD attaches valid Problem | Course and Problem permitted | Add CourseProblem | Association created | F010 |
| COURSE-QA-062 | Attach Problem from foreign scope | HOD-A; Problem B | Add link | Denied unless explicit cross-scope policy allows | F010 |
| COURSE-QA-063 | Manager attaches Problem | Manager within Section | Add Problem | Allowed only if scoped permission and policy approved | F010; RBAC |
| COURSE-QA-064 | Attach duplicate CourseProblem | Existing link | Add same Problem | Duplicate policy enforced; no unintended duplicate | F010; COURSE-D34 |
| COURSE-QA-065 | Attach unpublished/archived Problem | Problem status restrictive | Add link | Outcome follows Module 5 eligibility policy | F010; Module 5 |
| COURSE-QA-066 | CourseProblem ordering requested | Schema has no order field | Reorder items | Not supported unless approved schema/behavior exists | F010; COURSE-D34 |
| COURSE-QA-067 | CourseProblem add audit | Module 15 event approved | Add link | Audit event records actor/course/problem/scope per policy | F010; Module 15 |
| COURSE-QA-068 | Remove CourseProblem | Valid link | Remove | Association removed only | F011 |
| COURSE-QA-069 | Remove Problem from Course with history | Student submissions exist | Unlink | Problem/submissions remain; history policy applied | F011/F025 |
| COURSE-QA-070 | Manager removes CourseProblem | Manager scope | Remove link | Allowed only under approved Update* policy | F011; RBAC |
| COURSE-QA-071 | Remove missing association | Link absent | Remove | No unintended mutation; result follows API policy | F011 |
| COURSE-QA-072 | Student views permitted Course Problems | Course and Problems visible | Open list | Only permitted Problems returned | F012 |
| COURSE-QA-073 | CourseProblem points to inaccessible Problem | Problem restricted | Student list | Restricted Problem is not exposed | F012; Module 5 |
| COURSE-QA-074 | GroupCourseProblem vs CourseProblem | Both relations exist | Open Course content | Relations remain distinct; group-only link is not treated as course-wide | F012/F024 |
| COURSE-QA-075 | Course view bypasses Problem status | Problem hidden/archived | Request Problem directly | Module 5 access restriction still enforced | F012 |
| COURSE-QA-076 | Orphan CourseProblem | Link references missing Problem | Read Course | Safe error/filter; no crash or disclosure | F012 |
| COURSE-QA-077 | Student My Courses uses own identity | Student has enrollments | Open My Courses | Records derived from authenticated Student | F013 |
| COURSE-QA-078 | Client sends another studentId | Student request includes foreign ID | Query My Courses | Ignored/rejected; no other Student data | F013 |
| COURSE-QA-079 | Student visibility source conflict | Section-assigned but not enrolled Course | Open My Courses | Result follows approved discoverability rule | F013; COURSE-D21 |
| COURSE-QA-080 | Student Course list contains foreign College Course | Foreign Course ID | Load My Courses | No foreign Course appears absent explicit approved cross-scope permission | F013 |
| COURSE-QA-081 | Student has no Courses | No eligible records | Open My Courses | Empty state; no false enrollment | F013 |
| COURSE-QA-082 | Student course is archived | Enrollment exists | My Courses | Visibility follows approved archive rule | F013; COURSE-D28 |
| COURSE-QA-083 | Student subscription expired | Module 3 state known | My Courses | Access follows approved Module 3 entitlement | F013/F023; Module 3 |
| COURSE-QA-084 | My Courses includes GroupCourseProblem | Student group membership exists | Load Course | Group-associated content shown only under Module 10 permission | F013/F024 |
| COURSE-QA-085 | Student opens eligible Course | Enrollment/eligibility approved | Click Course | Course detail loads within allowed scope | F014 |
| COURSE-QA-086 | Student opens Course by guessed URL | No permission/enrollment | Direct URL | Denied/not-found; no data leak | F014 |
| COURSE-QA-087 | Manager opens Course outside Section | Manager scope mismatch | Direct URL | Denied | F014 |
| COURSE-QA-088 | Course access grants Lab/Assessment permission | Student has Course view only | Attempt Lab/Assessment action | Module 8/9 permission independently checked | F014; cross-module |
| COURSE-QA-089 | Course is DRAFT | Student access request | Open Course | Outcome follows approved DRAFT policy | F014; COURSE-D28 |
| COURSE-QA-090 | Course status changes during access | Request races with archive | Open/access | State consistency follows approved lifecycle policy | F014; COURSE-D40 |
| COURSE-QA-091 | Course access denied by subscription | Subscription policy defined | Open Course | Module 3 entitlement response enforced | F014/F023 |
| COURSE-QA-092 | HOD views enrollments in Department | Enrollment data exists | Open course enrollment panel | Only assigned Department data | F015 |
| COURSE-QA-093 | Manager views enrollment in assigned Section | Student in assigned Section | Open enrollments | Permitted student enrollment details shown | F015 |
| COURSE-QA-094 | Manager queries Student from another Section | Student foreign to Manager | Direct ID request | Denied | F015; RBAC |
| COURSE-QA-095 | Student views own enrollment | Own CourseEnrollment | Open enrollment | Own enrolledAt/completedAt returned | F015/F020 |
| COURSE-QA-096 | Student views another enrollment | Foreign CourseEnrollment | Request ID | Denied | F015; RBAC |
| COURSE-QA-097 | Finance Team queries enrollment | Finance role | Request CourseEnrollment | Access follows explicit permission; no Course View inference grants enrollment data | F015; COURSE-D36 |
| COURSE-QA-098 | HOD enrolls eligible Student | Course/Student in Department; policy approved | Create enrollment | CourseEnrollment created with correct refs and enrolledAt | F016 |
| COURSE-QA-099 | Enrollment for foreign Student | HOD-A; Student B | Create enrollment | Denied | F016 |
| COURSE-QA-100 | Enrollment for invalid Course | Missing courseId | Create | Reject; no row | F016 |
| COURSE-QA-101 | Enrollment for invalid Student | Missing/non-Student user | Create | Reject | F016; Module 1 |
| COURSE-QA-102 | Duplicate enrollment | Student already enrolled | Repeat create | Behavior follows approved uniqueness/repeat rule | F016; COURSE-D14 |
| COURSE-QA-103 | Enrollment at seat capacity | Module 3 reports no seat | Create | Denied/held according to Module 3 rule; no seat bypass | F016/F023 |
| COURSE-QA-104 | Manager assigns enrollment | Manager within Section | Create enrollment | Allowed only if Manager enrollment action approved | F016; COURSE-D30 |
| COURSE-QA-105 | Enrollment Course status not eligible | Course status restrictive | Create enrollment | Apply approved status eligibility; no assumption | F016; COURSE-D28 |
| COURSE-QA-106 | Client supplies enrolledAt/completedAt | Create request for Student | Submit forged timestamps | Server applies authoritative timestamps/state; rejects unauthorized fields | F016 |
| COURSE-QA-107 | Section assignment triggers automatic enrollment | PUBLISHED Course; eligible active roster | Add CourseSection link | Each eligible Student receives one CourseEnrollment; Module 3 entitlement checked; seat counters unchanged | COURSE-F017; COURSE-D11/COURSE-D13/COURSE-D25/COURSE-D29 |
| COURSE-QA-108 | Automatic enrollment partial failure | Bulk roster run; worker fails after some Student rows | Retry same outbox event | Existing rows remain; missing pairs retry without duplicates; event not marked complete prematurely | COURSE-F017; COURSE-D14/COURSE-D29/COURSE-D40 |
| COURSE-QA-109 | Duplicate Section event | Course/Section event already processed | Replay event | No duplicate CourseSection or CourseEnrollment; idempotent outcome | COURSE-F017; COURSE-D14/COURSE-D29/COURSE-D40 |
| COURSE-QA-110 | Student joins Section after Course assigned | PUBLISHED Course audience exists; Student newly enters Section | Activate Student membership | Student automatically enrolled after scope and Module 3 entitlement checks | COURSE-F017; COURSE-D11/COURSE-D13/COURSE-D29 |
| COURSE-QA-111 | Student self-enrollment disabled | Student has no CourseEnrollment; Course PUBLISHED | Request self-enrollment endpoint | Denied; no self-enrollment endpoint/control and no enrollment write | COURSE-F018; COURSE-D12/COURSE-D35 |
| COURSE-QA-112 | Student self-enrollment denied | Student has no CourseEnrollment | Request enroll action from UI/API | Denied; Student View-only RBAC applies | COURSE-F018; COURSE-D35 |
| COURSE-QA-113 | Student self-enrolls foreign Course | Student College A; Course B | Request enroll | Denied | F018; RBAC |
| COURSE-QA-114 | Student submits another studentId | Student self-enrollment route | Attempt enroll as other Student | Backend uses auth identity; reject/ignore forged ID | F018 |
| COURSE-QA-115 | Self-enroll duplicate/seat-full | Student already enrolled or no seat | Request enrollment | Approved duplicate/capacity policy enforced | F018; COURSE-D14/COURSE-D25 |
| COURSE-QA-116 | Student self-withdrawal | Student owns active CourseEnrollment | Attempt self-withdrawal | Denied in v1; Student cannot withdraw/delete enrollment | COURSE-F019; COURSE-D15 |
| COURSE-QA-117 | Student withdraws another enrollment | Foreign enrollment ID | Request withdrawal | Denied | F019; RBAC |
| COURSE-QA-118 | HOD withdraws enrollment | HOD-A; Student enrollment in assigned Department | Submit withdrawal with reason | Status WITHDRAWN; server withdrawnAt set; history retained and audit emitted | COURSE-F019; COURSE-D15/COURSE-D16/COURSE-D27 |
| COURSE-QA-119 | Repeat after completion | Student has COMPLETED CourseEnrollment | Replay Section roster event | No new row and no status reset; repeat attempts disabled in v1 | COURSE-F019; COURSE-D14/COURSE-D18 |
| COURSE-QA-120 | Withdraw enrollment with activity history | Code submissions/Labs exist | Withdraw | History preserved; access effect follows approved rule | F019/F025 |
| COURSE-QA-121 | Duplicate withdrawal | Already withdrawn | Repeat action | Idempotent/error behavior follows approved lifecycle | F019; COURSE-D15 |
| COURSE-QA-122 | Concurrent withdrawal and submission | Student action races with submission | Execute both | Module 6/18 transaction and timing policy enforced | F019; COURSE-D40 |
| COURSE-QA-123 | Student views enrolledAt | Enrollment exists | Open course progress | Timestamp displayed accurately | F020 |
| COURSE-QA-124 | completedAt null | Incomplete enrollment | View | No false completion label/timestamp | F020 |
| COURSE-QA-125 | HOD views completion within Department | Student in scope | Open enrollment report | Permitted completion data visible | F020 |
| COURSE-QA-126 | Manager views completion outside Section | Student foreign | Query | Denied | F020; RBAC |
| COURSE-QA-127 | UI shows progress percent without source | No progress schema | Open My Courses | No unsupported percentage invented | F020; COURSE-D37 |
| COURSE-QA-128 | HOD verifies Course completion | HOD reviewed eligible ENROLLED record | Confirm completion with reason | Status COMPLETED; server completedAt set; audit/notification events emitted | COURSE-F021; COURSE-D17/COURSE-D27/COURSE-D38 |
| COURSE-QA-129 | Course completion criterion unmet | Incomplete required activity | Process completion | completedAt remains unset | F021 |
| COURSE-QA-130 | Completion event replayed | Already completed | Replay | No duplicate side effects; timestamp policy applied | F021; COURSE-D38 |
| COURSE-QA-131 | Course completion by CourseProblem count | CourseProblem links exist; no formula | Process Problem solve event | No automatic completion from CourseProblem count | COURSE-F021; COURSE-D19/COURSE-D38 |
| COURSE-QA-132 | Lab/Assessment completion contribution | Student completes Lab or Assessment | Process activity completion event | No automatic Course completion; HOD verification required in v1 | COURSE-F021; COURSE-D38 |
| COURSE-QA-133 | Staff overrides completion | Override permission approved | Set completion manually | Only authorized override; audit required | F021; COURSE-D38 |
| COURSE-QA-134 | Completion reversal/repeat | Reversal policy approved | Reset/reopen Course | Approved lifecycle applied; historical record retained | F021; COURSE-D18/COURSE-D38 |
| COURSE-QA-135 | Search by Course title/code | Search supported | Query | Matching permitted Courses returned | F022; COURSE-D24 |
| COURSE-QA-136 | Filter by status/scope | Filters approved | Apply filter | Results match filter and scope | F022 |
| COURSE-QA-137 | Sort/paginate list | Sort/page supported | Request page | Stable ordering and page result per contract | F022 |
| COURSE-QA-138 | Search hidden Course | Student not eligible | Search title | Hidden Course not disclosed | F022; RBAC |
| COURSE-QA-139 | Filter with foreign College ID | HOD/Manager | Query | Server scope not widened | F022; RBAC |
| COURSE-QA-140 | Invalid page/sort value | Query API | Submit invalid param | Reject/normalize per API contract | F022; adopted API contract |
| COURSE-QA-141 | Seat eligibility positive | Module 3 confirms active College entitlement | Run auto-enrollment/open Course | Allowed; CourseEnrollment does not increment/reserve seat count | COURSE-F023; COURSE-D25 |
| COURSE-QA-142 | Seat eligibility negative | Module 3 denies entitlement | Run auto-enrollment | No new CourseEnrollment or active access; no Module 4 capacity calculation | COURSE-F023; COURSE-D25 |
| COURSE-QA-143 | Subscription state changes mid-enrollment | Race condition | Start then alter subscription | Result follows Module 3/18 consistency policy | F023 |
| COURSE-QA-144 | Seat counters disagree | College/Department/subscription values differ | Enroll | Course does not choose its own calculation; Module 3 policy governs | F023 |
| COURSE-QA-145 | Entitlement service unavailable | Module 3 entitlement check fails | Run automatic enrollment | Fail closed; retry reconciliation; no bypass or seat counter change | COURSE-F023/F017; COURSE-D25/COURSE-D29 |
| COURSE-QA-146 | College suspended but subscription active | Conflicting states | Course access | Precedence follows Module 2/3 approved policy | F023 |
| COURSE-QA-147 | Group member reads GroupCourseProblem | Group member permitted | Open group/course | Allowed association data returned | F024; Module 10 |
| COURSE-QA-148 | Non-member reads private GroupCourseProblem | Student not group member | Direct request | Denied/filtered | F024; Module 10 |
| COURSE-QA-149 | GroupCourseProblem mistaken for CourseProblem | Group-only relation exists | Student opens general Course | Group-only problem not automatically course-wide | F024 |
| COURSE-QA-150 | Group and Course scope mismatch | Group/ Course org differ | Request relation | Reject unless approved association rule permits | F024; Module 10 |
| COURSE-QA-151 | GroupCourseProblem update via Course module | User attempts group relation mutation | Course endpoint | Module 10 write permission/contract applies; Course endpoint cannot bypass it | F024 |
| COURSE-QA-152 | Course update with enrollment history | Existing enrollment/submissions | Change metadata | History references preserved | F025 |
| COURSE-QA-153 | Delete Course with CourseProblem links | Linked Problems | Delete | No unapproved cascade into Problem records | F025 |
| COURSE-QA-154 | Delete Course with Lab/Assessment links | Activities exist | Delete/archive | Retention/cascade decision enforced | F025; Modules 8/9 |
| COURSE-QA-155 | Partial failure updating Course and relationships | Multi-record change proposed | Inject failure | Transaction/recovery behavior per approved architecture; no inconsistent partial state | F025; COURSE-D40 |
| COURSE-QA-156 | Concurrent Course/Problem unlink and submission | Student activity in progress | Perform both | Module 6/18 consistency policy applied; historical submission remains traceable | F025 |
| COURSE-QA-157 | Audit Course/enrollment mutation | Audit catalog approved | Create/update/archive/enroll | Module 15 receives actor, target, action, scope per policy | F025; Module 15 |

| COURSE-QA-158 | Published assignment enrolls full active roster | PUBLISHED SECTION_ONLY Course; active eligible Students S1/S2 | Assign to Section A1-1 | Exactly one ENROLLED row per eligible Student with server timestamp/source | COURSE-F007/F016/F017; COURSE-D11/COURSE-D13/COURSE-D14/COURSE-D29 |
| COURSE-QA-159 | Student joins assigned Section later | PUBLISHED Course audience includes A1-1; Student S3 newly enters | Activate Student Section membership | S3 automatically receives one CourseEnrollment after Module 3 entitlement check | COURSE-F017; COURSE-D11/COURSE-D13/COURSE-D29 |
| COURSE-QA-160 | DRAFT assignment waits for publication | DRAFT Course with audience link and active roster | Assign Section then publish | No enrollment while DRAFT; publish triggers roster reconciliation | COURSE-F007/F017; COURSE-D11/COURSE-D22/COURSE-D28/COURSE-D29 |
| COURSE-QA-161 | Overlapping audience trigger | Student covered by DEPT_WIDE and another trigger | Reconcile all audience links | Unique Student-Course row prevents duplicate enrollment | COURSE-F017; COURSE-D07/COURSE-D14/COURSE-D29 |
| COURSE-QA-162 | Section activation updates audience | Published DEPT_WIDE/GLOBAL Course; new Section activates | Process Module 2 Section activation | CourseSection audience materializes and eligible roster is reconciled | COURSE-F017; COURSE-D07/COURSE-D09/COURSE-D29 |
| COURSE-QA-163 | Section unlink preserves enrollment history | Existing CourseSection and CourseEnrollment | Remove CourseSection link | No enrollment/activity history is deleted; Student retains history-only Course if outside audience | COURSE-F008/F019/F025; COURSE-D20/COURSE-D32/COURSE-D33 |
| COURSE-QA-164 | Student transfer out of Course audience | Student enrolled then transfers to out-of-scope Section | Process Section transfer | Enrollment retained, not reset; Course is history-only; no new audience access | COURSE-F017/F019; COURSE-D18/COURSE-D20/COURSE-D32 |
| COURSE-QA-165 | Archived Course roster reconciliation | ARCHIVED Course; new Student/Section event | Run reconciliation | No new CourseEnrollment; existing records remain historical | COURSE-F017; COURSE-D22/COURSE-D23/COURSE-D29 |
| COURSE-QA-166 | Restore requires separate publish | ARCHIVED Course restored to DRAFT; Section audience remains | Restore then publish separately | Restore creates no enrollment; later publication triggers reconciliation | COURSE-F004/F005/F017; COURSE-D23/COURSE-D28/COURSE-D29 |
| COURSE-QA-167 | Archived Course is read-only | Student enrolled before archive | Open Course and attempt new activity | Historical metadata readable; new activity denied | COURSE-F014; COURSE-D22/COURSE-D23 |
| COURSE-QA-168 | Student cannot self-enroll from My Courses | Student not enrolled; Course PUBLISHED | Attempt UI and API enrollment | No self-enrollment control/endpoint and no CourseEnrollment row | COURSE-F018; COURSE-D35 |
| COURSE-QA-169 | Forged enrollment identity | Student request supplies another studentId | Attempt enrollment request | Denied; identity is server-derived; no manual/self-enrollment path | COURSE-F018/F016; COURSE-D12/COURSE-D35/COURSE-D36 |
| COURSE-QA-170 | CourseEnrollment does not change seats | Record usedSeats before/after enrollment | Process automatic roster enrollment | CourseEnrollment may be created; usedSeats/purchasedSeats unchanged | COURSE-F016/F023; COURSE-D25 |
| COURSE-QA-171 | Module 3 entitlement denial | Eligible Student but Module 3 denies entitlement | Run auto-enrollment | No enrollment; failure is retryable; no seat mutation | COURSE-F017/F023; COURSE-D13/COURSE-D25/COURSE-D29 |
| COURSE-QA-172 | Partial roster worker failure and replay | Ten Students; worker commits four then fails | Replay same outbox event | Four existing rows remain; six missing rows created once; event not complete until reconciliation ends | COURSE-F017; COURSE-D14/COURSE-D29/COURSE-D40 |
| COURSE-QA-173 | Enrollment event audit | CourseSection assignment triggers auto-enrollment | Process event | Module 15 records service actor, Course, Student, Section, scope, event key and timestamp | COURSE-F017; COURSE-D27/COURSE-D29 |
| COURSE-QA-174 | HOD verifies completion with reason | HOD-A; Student has ENROLLED record | Confirm completion | Status COMPLETED; server completedAt; audit written | COURSE-F021; COURSE-D16/COURSE-D27/COURSE-D38 |
| COURSE-QA-175 | Duplicate completion preserves timestamp | CourseEnrollment already COMPLETED | Repeat HOD completion | No timestamp reset or duplicate side effect | COURSE-F021; COURSE-D18/COURSE-D38/COURSE-D40 |
| COURSE-QA-176 | Manager cannot mark completion | Manager-A1-1; Student assigned Section | Attempt completion override | Denied; HOD verifies and SUPER_ADMIN correction requires reason | COURSE-F021; COURSE-D03/COURSE-D38 |
| COURSE-QA-177 | Completed Course cannot be re-enrolled | Student has COMPLETED CourseEnrollment | Replay roster/repeat request | Existing row remains COMPLETED; no second row or reset | COURSE-F019; COURSE-D14/COURSE-D18 |
| COURSE-QA-178 | Finance cannot inspect enrollment | Finance-A has metadata View | Request enrollment list/export | Denied; metadata View does not include academic records | COURSE-F015; COURSE-D04/COURSE-D30/COURSE-D36 |
| COURSE-QA-179 | Foreign College scope isolation | GLOBAL Course in College A; College B data exists | Query foreign College/Section IDs | No foreign Course, Section, Student, or enrollment data returned | COURSE-F001/F006/F014; COURSE-D08/COURSE-D31 |
| COURSE-QA-180 | Active Student membership is the only seat trigger | Record Module 3 usage before/after Course enrollment | Process CourseEnrollment automation | usedSeats unchanged; membership activation consumed seat, not CourseEnrollment | COURSE-F023; COURSE-D25 |

## QA review note

This suite contains 180 unique target cases. Product decisions are adopted; implementation execution is still pending. Do not mark a case passed until it has been executed against the implementation with evidence recorded. The suite covers access, data integrity, Option A enrollment, Module 3 seat isolation, retry, and concurrency.