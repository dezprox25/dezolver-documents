# DEZOLVER — Module 10: Groups & Membership

**Document ID:** DEZ-GRP-FS-001  
**Version:** 0.2 — expanded function-level spec + QA companion  
**Document status:** Detailed draft — aligned to Functional Specification Standard v1.0; recommendations and product/security decisions remain open  
**Approval status:** Not approved for implementation  
**Prepared:** 2026-11-04

> Committing this document versions the specification. It does not approve the target behavior. Source-confirmed facts, derived requirements, and recommendations are distinguished throughout.

## 1. Purpose and outcome

Groups organize students and associate collaborative academic activities. Business Flow v2 describes Groups as a mediator for collaborative Labs and Contests. The schema supports Group membership, Group-linked Labs, Contests, Course Problems, and an Assessment reference.

The module covers Group lifecycle, membership, invite-code join, activity associations, visibility, authorization, and failure handling. It does **not** own Lab/Contest/Assessment creation, individual submissions, Judge0 execution, scoring, ranking, or enrollment.

## 2. Scope

### In scope
- List/view Groups by role and scope.
- Create and edit Group metadata.
- Activate/deactivate Group.
- View/regenerate invite code.
- Student join via invite code.
- Staff member management and proposed student self-leave.
- Group-Lab, Group-Contest, Group-CourseProblem associations.
- Visibility of Group-linked activities and Assessment.groupId.
- Backend scope enforcement, validation, audit/privacy, and failure handling.

### Out of scope
- Creating/editing Labs, Contests, Courses, Problems, Assessments.
- Individual submission, scoring, ranking, or assessment attempt behavior.
- College/department/section lifecycle.
- Subscription/seat rules.
- Implementation of schema changes without separate approval.

## 3. Source documents and evidence

| Source | Relevant content |
|---|---|
| [RBAC.md](../../docs/source-text/RBAC.md), original PDF pages 3, 5–6, 8 | Group permission matrices, role scope, backend authorization, tenant isolation, audit guidance, testing checklist. |
| [Business Flow v2](../../docs/source-text/Business%20Flow%20v2.md), original pages 3–5 | HOD/Manager/Student Group workflows; Groups mediate Labs and Contests. |
| [Page-to-DB Mapping v2](../../docs/source-text/Page%20-%20DB%20Mapping%20v2.md), original pages 2–3 | HOD Groups, Manager Groups/Classes, Student My Groups page mappings. |
| [Schema](../../docs/source-text/scheema.md) | Group, GroupMember, GroupLab, GroupContest, GroupCourseProblem, Assessment.groupId. |
| [Functional Specification Standard](../FUNCTIONAL-SPECIFICATION-STANDARD.md) | Evidence labels, function-level requirements, decision log, review and approval process. |

**Evidence limitations**
- Business Flow v2 does not define detailed Group CRUD, membership lifecycle, invite-code lifecycle, or deletion behavior.
- Page-to-DB Mapping identifies data entities; it does not establish exact UI behavior.
- Conceptual schema is not proof of deployed implementation or database constraints.
- RBAC Group permission rows conflict for MANAGER and differ in Student wording.
- No legacy implementation behavior is verified here.

Evidence labels:
- **Source-confirmed:** directly stated in supplied sources.
- **Derived:** logically follows from multiple source facts.
- **Recommendation:** proposed target behavior; not approved.
- **TBD / Decision Required:** requires product/security/technical decision.

## 4. Terminology

| Term | Definition |
|---|---|
| Group | Academic grouping represented by the Group schema entity. |
| Group member | Student linked through GroupMember. |
| Invite code | Group.inviteCode, schema type varchar(20); lifecycle is not specified by source. |
| Group activity | Lab, Contest, Course Problem association, or Assessment reference. |
| Permitted Group | Group actor may access after role, college, department, section/group, and ownership checks. |
| Active Group | Group.isActive=true; behavior beyond the field is recommended below. |

## 5. Actors, permissions, and scope

### 5.1 Source-confirmed role scope

| Role | Scope |
|---|---|
| SUPER_ADMIN | Global |
| FINANCE_TEAM | Assigned College |
| HOD | Assigned College + Department |
| MANAGER | Assigned College + Department + Section / permitted Group |
| STUDENT | Own records + permitted academic resources |

### 5.2 Source permission matrix comparison

| Role | Platform matrix | Page matrix | API matrix | Treatment |
|---|---|---|---|---|
| SUPER_ADMIN | Full CRUD | V/C/U/D | Full CRUD | Global CRUD. |
| FINANCE_TEAM | View | V | V* | View only, assigned college. |
| HOD | Full CRUD | V/C/U/D | Full CRUD* | CRUD within assigned college + department. |
| MANAGER | Full CRUD* | V/C/U* | V/C/U* | Conflict: Delete appears only in platform matrix. |
| STUDENT | Join/View | V OWN | V OWN/JOIN | Own/permitted Group view and join; details TBD. |

**Source-confirmed RBAC rule:** Page access does not grant access to all records on the page. Backend/API authorization and tenant/scope checks are mandatory. Frontend visibility is not security.

**Conservative recommendation pending approval:** MANAGER may View/Create/Update within explicit permitted scope; deny Group Delete until the RBAC discrepancy is resolved.

### 5.3 Permission matrix for this draft

| Action | SUPER_ADMIN | FINANCE_TEAM | HOD | MANAGER | STUDENT |
|---|---|---|---|---|---|
| List/view Group | Global | College view | College + department | Explicit permitted Group | Own/permitted Groups |
| Create Group | Allow | Deny | Allow | Conditional; source allows create but scope unresolved | Deny |
| Update Group | Allow | Deny | Allow | Conditional, permitted Group | Deny |
| Delete | Allow per source; recommend soft deactivate | Deny | Allow per source; recommend soft deactivate | Deny pending decision | Deny |
| Activate/deactivate | Allow | Deny | Allow | Conditional | Deny |
| View/regenerate code | Recommended allow | Deny | Recommended allow | Conditional | Deny |
| Join Group | Admin function | Deny | Staff-managed membership recommended | Staff-managed membership recommended | Allow with valid code recommended |
| Add/remove member | Allow | Deny | Recommended within department | Recommended within permitted Group | Deny other members |
| Leave own Group | N/A | Deny | N/A | N/A | Recommended allow |
| Assign Lab/Contest/Course Problem | Allow | View only | Allow | Conditional, permitted Group | Deny |

### 5.4 Scope rules
- SUPER_ADMIN: global.
- FINANCE_TEAM: assigned college only.
- HOD: collegeId + departmentId must match assigned scope.
- MANAGER: assigned college/department plus explicit permitted Group/section scope.
- STUDENT: own membership and permitted academic scope; student-owned records require authenticated ownership checks.
- Every read/write must enforce scope server-side.
- A Group outside scope must be denied even if the actor knows its ID or invite code.
- Client-supplied tenant, department, section, role, or student IDs must not bypass scope.

## 6. Navigation and entry points

### Source-confirmed page mappings
- HOD: Groups page — Group, GroupMember, User, Department, GroupLab, GroupContest.
- MANAGER: Groups / Classes — Group, GroupMember, User, Department, GroupLab, GroupContest.
- STUDENT: My Groups — Group, GroupMember, GroupLab, GroupContest, User, Department.
- GroupCourseProblem is mapped in Courses & Labs.
- Assessment.groupId exists in schema; dedicated Group Assessment UI is not mapped.

Recommended entry points: HOD Groups; permitted Manager Groups/Classes; Student My Groups; SUPER_ADMIN Group administration. Finance route is TBD. Exact routes, labels, filters, empty states, and UI layout are TBD.

## 7. Functional inventory

| ID | Function | Actors | Type | Evidence/status |
|---|---|---|---|---|
| GRP-F001 | List/filter Groups | Admin/staff/student | Read | Page mapping + recommended filters |
| GRP-F002 | View Group detail | Authorized actors | Read | Derived from mapping |
| GRP-F003 | Create Group | SUPER_ADMIN/HOD/conditional Manager | Create | CRUD source-confirmed; fields recommended |
| GRP-F004 | Update Group metadata | SUPER_ADMIN/HOD/conditional Manager | Update | CRUD source-confirmed; fields recommended |
| GRP-F005 | Activate/deactivate | SUPER_ADMIN/HOD/conditional Manager | Lifecycle | isActive source-confirmed; lifecycle recommended |
| GRP-F006 | View/regenerate invite code | SUPER_ADMIN/HOD/conditional Manager | Workflow | inviteCode source-confirmed; lifecycle recommended |
| GRP-F007 | Join via invite code | Student | Workflow | Student Join source-confirmed; workflow recommended |
| GRP-F008 | Manage members / leave | Staff/student self | Workflow | GroupMember source-confirmed; action rules recommended |
| GRP-F009 | Link Labs/Contests | SUPER_ADMIN/HOD/conditional Manager | Association | Schema-confirmed links; operations recommended |
| GRP-F010 | Link Course Problems | SUPER_ADMIN/HOD/conditional Manager | Association | Schema-confirmed; operations recommended |
| GRP-F011 | View linked activities/Assessment | Staff/member student | Read | Schema/mapping-supported; semantics TBD |

## 8. Function-level specifications

> Main flows below are recommended target behavior wherever source details are absent. They are proposals, not approved requirements.

### GRP-F001 — List and filter Groups

**Status:** Source-supported page mapping; scope is source-confirmed; filters are recommended.  
**Actors:** SUPER_ADMIN, FINANCE_TEAM, HOD, MANAGER, STUDENT.

**Purpose:** Show only Groups the actor is authorized to discover.

**Preconditions:** Authenticated actor; role has Group view permission; scope resolved.

**Entry:** HOD Groups, Manager Groups/Classes, Student My Groups, SUPER_ADMIN admin view.

**UI (recommendation):** list permitted Groups. Staff may see name, department, status, member count, creator, created date; student view shows own memberships. Recommended filters: name, status, department where permitted. Pagination/sort TBD. Do not expose inviteCode.

**Inputs:** optional search text; optional ACTIVE/INACTIVE filter; optional department filter (staff only). All filters must be scope constrained.

**Main flow**
1. Actor opens Group page.
2. Backend resolves role and scope.
3. Backend queries Groups with college/department/section/group/ownership filters.
4. Backend returns authorized fields only.
5. UI renders rows and permission-based actions.

**Rules**
- Source-confirmed: role and record scope both apply.
- Recommendation: Student My Groups lists own memberships, not an unrestricted Group directory.
- Recommendation: inactive Groups are excluded from Student join/default list; staff may see inactive Groups.
- Finance Group detail fields remain TBD.

**Validation/backend:** Ignore/reject forged scope filters; never trust client role or tenant fields; return no unauthorized rows.

**Data changes:** None.

**Success/failure:** Show permitted records; unauthorized page denied; query errors use shared failure policy; empty list must not leak inaccessible record counts.

**Edge cases:** no Groups; only inactive Groups; membership changed after list loaded.

**Audit/privacy:** Read audit policy belongs to Module 15; do not expose codes.

**Acceptance criteria**
- HOD sees only assigned college + department Groups.
- Manager sees only explicitly permitted Groups.
- Student sees own membership data only.
- Forged college/department filters do not widen scope.

**QA**
| Test | Scenario | Expected |
|---|---|---|
| GRP-QA-001 | HOD lists Groups | Only department scope. |
| GRP-QA-002 | HOD uses foreign department filter | No foreign data. |
| GRP-QA-003 | Manager has no Group assignment | Group not listed. |
| GRP-QA-004 | Student opens My Groups | Own membership only. |
| GRP-QA-005 | Client submits foreign collegeId | Backend scope remains enforced. |

**Open:** GRP-D03, D04, D12, list/filter/pagination behavior.

### GRP-F002 — View Group details

**Status:** Derived from page mapping; display rules recommended.  
**Actors:** SUPER_ADMIN, FINANCE_TEAM, HOD, MANAGER, member/permitted Student.

**Purpose:** Display Group metadata, permitted membership, and associated activities.

**Preconditions:** Group exists; actor is authorized for that Group.

**Entry:** Select Group from a permitted list/link.

**UI (recommendation):** Group identity, description, department/status, permitted members, linked Labs/Contests/Course Problems; Assessment linkage only if owning module supports it. Hide inviteCode from Student/general views.

**Input:** groupId; must be an in-scope Group.

**Flow:** authorize Group → load Group and independently authorized relations → render permitted detail/action controls.

**Rules:** Mapping source confirms Group, GroupMember, User, Department, GroupLab, GroupContest. Group access does not automatically grant access to every linked activity. Student-safe fields/member contact visibility is TBD.

**Backend/data:** Read Group and allowed relations; independently scope each associated resource. No changes.

**Failures/edge cases:** missing/foreign Group ID must not disclose existence; related activity out of scope omitted/denied; inactive Group student visibility TBD; no members/activity.

**Privacy:** inviteCode must not be in generic detail payload.

**Acceptance:** HOD views own Group; foreign Group denied; Student cannot inspect another student's private records; activities checked individually.

**QA**
| Test | Scenario | Expected |
|---|---|---|
| GRP-QA-006 | HOD views own Group | Permitted detail. |
| GRP-QA-007 | Manager views unassigned Group | Denied/no disclosure. |
| GRP-QA-008 | Student views own Group | Student-safe detail. |
| GRP-QA-009 | Student detail response includes code | Must not include inviteCode. |

**Open:** D03, D12, member field visibility, inactive detail policy.

### GRP-F003 — Create Group

**Status:** CRUD permissions source-confirmed; form/defaults recommended.  
**Actors:** SUPER_ADMIN, HOD, conditional Manager.

**Purpose:** Create Group in authorized scope.

**Preconditions:** Authenticated/authorized actor; target department in scope.

**Entry:** Create Group action.

**UI recommendation:** Name required, Description optional; college/department derived from authorization or selected from permitted values; invite code generated by system. Initial active status recommended.

**Inputs**
| Field | Schema | Proposed validation |
|---|---|---|
| name | varchar(255) | Required, trim, nonblank, max 255 |
| description | text | Optional; max length TBD |
| collegeId | varchar | System-derived; never trust arbitrary client value |
| departmentId | varchar | Scoped selection/system-derived |
| createdById | varchar | Authenticated actor |
| inviteCode | varchar(20) | System-generated |
| isActive | boolean | Default true recommended |
| id/timestamps | schema | System-generated |

**Flow:** open form → enter fields → client checks → backend rechecks role/scope → validates → generates ID/code → creates Group → displays result.

**Rules:** Group schema confirms fields. Recommendation: active name unique within department (needs approval); create transaction must not leave partial data; no GroupMember is created automatically unless approved. Manager creation scope unresolved.

**Validation:** missing/blank name; >255 chars; foreign department; duplicate active name if approved; invite-code collision retry.

**Backend/data:** create Group only; no implicit membership/activity links.

**Failure:** no record on authorization/validation error; duplicate conflict handled without overwriting; idempotency/retry policy TBD.

**Edge:** concurrent duplicate names; department deactivated; missing HOD department assignment.

**Audit/privacy:** recommended GROUP_CREATED with actor/group/scope; never log code.

**Acceptance:** HOD creates in assigned dept; Student/Finance denied; client cannot override collegeId/creator; invalid name creates no record.

**QA**
| Test | Scenario | Expected |
|---|---|---|
| GRP-QA-010 | HOD creates valid Group | Scoped Group created. |
| GRP-QA-011 | Student creates Group | Denied. |
| GRP-QA-012 | HOD submits foreign department | Denied. |
| GRP-QA-013 | Blank/oversized name | Validation error/no record. |
| GRP-QA-014 | Duplicate name | Follows approved uniqueness rule. |

**Open:** D03, D05, default status, name uniqueness, idempotency.

### GRP-F004 — Update Group metadata

**Status:** CRUD source-confirmed; mutable fields recommended.  
**Actors:** SUPER_ADMIN, HOD, conditional Manager.

**Purpose:** Update metadata without accidentally changing tenant ownership or relationships.

**Preconditions/entry:** Group exists and is in scope; Edit action.

**UI recommendation:** Name and Description editable; college/department read-only; invite-code and active state separate actions.

**Inputs:** groupId; name required/nonblank/max 255; description optional.

**Flow:** open edit → authorize → load current values → edit → backend reauthorize/revalidate → update permitted fields and updatedAt → display updated record.

**Rules:** Recommend only name/description editable in general form. Keep collegeId, departmentId, createdById, inviteCode, isActive outside generic edit. Department transfer requires separate workflow/approval.

**Backend/data:** update only approved fields; stale-write/concurrency policy TBD.

**Failure/edge:** foreign Group, duplicate name, concurrent update, inactive Group edit, no-op save.

**Audit:** recommended GROUP_UPDATED with changed field names; avoid full description in logs.

**Acceptance:** HOD updates metadata in scope; no tenant transfer through edit; invalid changes do not persist.

**QA**
| Test | Scenario | Expected |
|---|---|---|
| GRP-QA-015 | HOD edits name/description | Changes persist. |
| GRP-QA-016 | Manager edits foreign Group | Denied. |
| GRP-QA-017 | Client changes collegeId | Rejected/ignored; scope unchanged. |
| GRP-QA-018 | Concurrent edit | Shared conflict policy applied. |

**Open:** D03, D05, transfer and concurrency policy.

### GRP-F005 — Activate/deactivate Group

**Status:** Group.isActive source-confirmed; lifecycle is recommendation.  
**Actors:** SUPER_ADMIN, HOD, conditional Manager.

**Purpose:** Stop/restore operational use without destroying relationships.

**Preconditions/entry:** Group exists; actor can update lifecycle; status action.

**UI recommendation:** Show current status; confirmation on deactivation.

**Flow:** request status change → backend authorizes/checks current state → update isActive and updatedAt → show new status.

**Rules (recommendations):** deactivation is logical removal; preserve GroupMember, GroupLab, GroupContest, GroupCourseProblem, and historical references; no new joins or associations for inactive Group; reactivation does not rejoin removed students.

**Validation/backend:** only boolean; repeated operation idempotent; race with join/assignment must respect final inactive state.

**Data:** update Group.isActive; preserve relations.

**Failures/edge:** unauthorized, already in state, deactivation during active activity, concurrent join/assignment.

**Audit:** GROUP_ACTIVATED / GROUP_DEACTIVATED recommended.

**Acceptance:** deactivation preserves records; inactive Group cannot be joined; repeated deactivate has no duplicate side effects.

**QA**
| Test | Scenario | Expected |
|---|---|---|
| GRP-QA-019 | HOD deactivates | isActive=false; links remain. |
| GRP-QA-020 | Student joins inactive Group | Rejected. |
| GRP-QA-021 | Repeat deactivate | Idempotent. |
| GRP-QA-022 | Deactivate during contest | Approved active-activity policy applies. |

**Open:** D02, D07, concurrency.

### GRP-F006 — View/regenerate invite code

**Status:** inviteCode field source-confirmed; lifecycle recommendation.  
**Actors:** SUPER_ADMIN, HOD, conditional Manager.

**Purpose:** Manage code used by students to request membership.

**Preconditions:** Group active; actor has Group-management permission.

**UI recommendation:** Code shown only in dedicated staff management UI; regenerate confirmation; code hidden from list and Student/general detail.

**Input:** groupId; code system-generated.

**Flow:** authorize → retrieve current code in narrow response → regenerate request/confirmation → backend creates new code → atomically invalidates old code → display new code.

**Rules:** source confirms varchar(20). Recommend opaque unpredictable code ≤20 characters, one current code per Group, immediate invalidation on regeneration, active Group only. Expiry, use limits, rate limit, and revoke policy TBD.

**Backend/data:** update inviteCode and updatedAt; avoid race; retry collisions.

**Failure/edge:** unauthorized view, inactive Group, concurrent regenerate, stale code, compromised code.

**Audit/privacy:** GROUP_INVITE_CODE_REGENERATED recommended; never log code or raw join code.

**Acceptance:** Student/general API cannot read code; authorized staff can regenerate; old code invalid; code ≤20 chars.

**QA**
| Test | Scenario | Expected |
|---|---|---|
| GRP-QA-023 | HOD regenerates code | New code; old invalid. |
| GRP-QA-024 | Student requests code | Denied/no disclosure. |
| GRP-QA-025 | Code length | ≤20 chars. |
| GRP-QA-026 | Code collision | Retry; unique usable code. |

**Open:** D08, expiry/revocation/rate limit, Manager rights.

### GRP-F007 — Join Group by invite code

**Status:** Student Join source-confirmed; code workflow recommended.  
**Actor:** STUDENT.

**Purpose:** Join eligible Group by code and create GroupMember.

**Preconditions:** authenticated eligible student; active Group; valid code; scope eligibility.

**Entry/UI recommendation:** My Groups → Join Group; code input. Do not disclose Group identity/details before code validation.

**Inputs:** inviteCode (required, ≤20 chars); studentId supplied only by authenticated identity.

**Flow:** student submits code → backend validates actor/code/Group status/scope → checks existing membership → creates GroupMember(groupId, studentId, joinedAt) → confirms and displays Group.

**Rules:** source confirms Student Join and GroupMember fields. Recommend one membership per student/Group; allow multiple Groups per student; require same college/department by default unless cross-department membership approved. Group must be active. Academic batch/section/enrollment/subscription eligibility TBD.

**Validation:** invalid/stale code; inactive Group; foreign college/department; duplicate membership; inactive student account. Do not leak protected Group details.

**Backend/data:** transaction/concurrency-safe membership create; never accept studentId from client.

**Failures/edge:** duplicate join idempotent; code regenerated between check and join; Group deactivated during join; concurrent join requests.

**Audit/privacy:** GROUP_MEMBER_JOINED recommended; never log raw code.

**Acceptance:** valid code creates one membership; foreign scope rejected; duplicate does not duplicate; invalid code reveals no Group; no impersonation.

**QA**
| Test | Scenario | Expected |
|---|---|---|
| GRP-QA-027 | Valid code | Membership created; Group visible. |
| GRP-QA-028 | Invalid code | No membership. |
| GRP-QA-029 | Client submits another studentId | Authenticated student identity used. |
| GRP-QA-030 | Join same Group twice | One membership only. |
| GRP-QA-031 | Old code after regeneration | Rejected. |

**Open:** D06, D08, account eligibility.

### GRP-F008 — Manage membership / leave Group

**Status:** GroupMember schema/page mapping source-confirmed; operation rules recommended.  
**Actors:** SUPER_ADMIN, HOD, conditional Manager; Student self-leave recommended.

**Purpose:** Staff manage membership; Student can leave own Group if approved.

**Preconditions:** Group in actor scope; target student eligible; permission for membership operation.

**Entry/UI recommendation:** Group detail → Members; Student My Groups → Leave. Staff add uses eligible scoped student selector; removal/leave confirmation.

**Inputs:** groupId; studentId for staff add/remove only; action Add/Remove/Leave.

**Flow:** staff opens members → backend scopes → selects eligible student → validates → creates membership; or actor confirms remove/leave → backend authorizes → applies approved membership model → refreshes UI.

**Rules:** schema has GroupMember(id, groupId, studentId, joinedAt). Recommend duplicate membership prevention, student may belong to multiple Groups, and membership end must not delete individual submissions/scores/participation. Current schema has no status/leftAt/removedAt; removal lifecycle must be decided. Ongoing activity access/attempt handling is TBD.

**Validation:** duplicate add; non-member removal; foreign student; Manager outside permitted Group; Student attempts another student's membership.

**Backend/data:** GroupMember create/delete or approved status update. If hard delete, preserve required audit/history. Membership status/history needs schema extension if retained.

**Failures/edge:** member removed/re-added; multiple memberships; active contest/lab/assessment; Group deactivation.

**Audit/privacy:** GROUP_MEMBER_ADDED/REMOVED/LEFT recommended; minimize personal data.

**Acceptance:** staff add/remove in scope; Student cannot manage others; self-leave uses authenticated identity; history preserved; server scope enforced.

**QA**
| Test | Scenario | Expected |
|---|---|---|
| GRP-QA-032 | HOD adds eligible student | Membership created. |
| GRP-QA-033 | Manager removes member from unassigned Group | Denied. |
| GRP-QA-034 | Student removes another member | Denied. |
| GRP-QA-035 | Student leaves own Group | Applies approved leave model. |
| GRP-QA-036 | Member with historical submissions removed | History remains. |
| GRP-QA-037 | Duplicate add | No duplicate membership. |

**Open:** D06, D07, membership history schema.

### GRP-F009 — Associate Group with Labs and Contests

**Status:** GroupLab/GroupContest relationships and collaborative flow source-confirmed; association workflow recommended.  
**Actors:** SUPER_ADMIN, HOD, conditional Manager.

**Purpose:** Link existing Lab Assignment/Contest to Group.

**Preconditions:** Group and activity exist; actor may manage both; Group active (recommended).

**Entry/UI recommendation:** Group detail → Activities → Assign Lab/Contest; scoped selector; remove/unassign control subject to lifecycle restrictions.

**Inputs:** groupId + labAssignmentId or contestId.

**Flow:** open association view → load eligible activities → select → backend validates scope/ownership/lifecycle → create GroupLab/GroupContest relation → display link.

**Rules:** schema confirms GroupLab(groupId, labAssignmentId, createdAt) and GroupContest(groupId, contestId, createdAt); Business Flow v2 shows Groups mediating Labs/Contests. Recommend same college/department compatibility, duplicate prevention, and no new associations for inactive Groups. Whether assignment/unassignment allowed after activity start/end is TBD.

**Backend/data:** create/remove relation only. Do not implicitly create LabSubmission, ContestParticipant, CodeSubmission, scores, or ranks.

**Failure/edge:** duplicate association, foreign activity, activity started/ended, Group deactivated after assignment, archived activity, member removed.

**Audit:** GROUP_LAB_ASSIGNED/UNASSIGNED and GROUP_CONTEST_ASSIGNED/UNASSIGNED recommended.

**Acceptance:** permitted association succeeds; cross-scope denied; duplicate prevented; no implicit participation/submission/score.

**QA**
| Test | Scenario | Expected |
|---|---|---|
| GRP-QA-038 | HOD links Lab | GroupLab created. |
| GRP-QA-039 | Manager links foreign Contest | Denied. |
| GRP-QA-040 | Duplicate Lab link | No duplicate relation. |
| GRP-QA-041 | Group association triggers score creation | No score created. |

**Open:** D09, activity lifecycle/unassign rules.

### GRP-F010 — Associate Course Problems with Group

**Status:** GroupCourseProblem schema source-confirmed; workflow recommended.  
**Actors:** SUPER_ADMIN, HOD, conditional Manager.

**Purpose:** Link a Problem to a Group in a Course context.

**Preconditions:** Group, Course, Problem exist; actor can manage them.

**Entry/UI recommendation:** Group detail → Course Problems or mapped Courses & Labs page.

**Inputs:** groupId, courseId, problemId.

**Flow:** load permitted Course/Problem options → select → backend checks role/scope and relationship → create GroupCourseProblem → show link.

**Rules:** schema confirms groupId/courseId/problemId/createdAt. Recommend Problem must be permitted by Course rules; compatible college/department; duplicate prevention. GroupCourseProblem must not override CourseProblem or enrollment rules. Exact grant/narrowing semantics TBD.

**Backend/data:** create/remove association only; do not mutate CourseProblem or create submissions.

**Failures/edge:** out-of-scope Problem, missing Course, duplicate, Problem removed from Course, Group deactivated.

**Audit:** Group Course Problem association events recommended.

**Acceptance:** valid association succeeds; out-of-scope association rejected; duplicates prevented; CourseProblem is not silently changed.

**QA**
| Test | Scenario | Expected |
|---|---|---|
| GRP-QA-042 | HOD associates permitted Problem | Relation created. |
| GRP-QA-043 | Manager uses foreign Problem | Denied. |
| GRP-QA-044 | Problem not available to Course | Rejected under recommended rule. |
| GRP-QA-045 | Duplicate relation | No duplicate. |

**Open:** D10, association/unassignment semantics.

### GRP-F011 — View Group-linked activities and Assessments

**Status:** schema/mapping source-confirmed; activity behavior derived/TBD.  
**Actors:** Staff, member Student, SUPER_ADMIN.

**Purpose:** Show associated Labs, Contests, Course Problems, and Assessment references in Group context.

**Preconditions:** actor may view Group and linked activity.

**Entry/UI recommendation:** Group detail → Activities; sections shown only where owning modules expose them.

**Inputs:** groupId; optional activity filter TBD.

**Flow:** authorize Group → retrieve associations → independently authorize each activity → display permitted metadata and links.

**Rules:** Assessment has groupId in schema. Recommend this means targeting/scope only unless Assessment module explicitly approves group attempt/scoring. Preserve individual student submissions/attempts/scores; do not infer group score or shared submission. Group membership alone does not override owning module access.

**Backend/data:** read associations; do not create Assessment/submission records.

**Failure/edge:** archived activity, revoked access, membership changed, Group inactive, Student joins after assessment starts.

**Acceptance:** only permitted activities shown; Group link does not override activity access; no group-level attempt/score inferred; Student cannot see other student's private attempt.

**QA**
| Test | Scenario | Expected |
|---|---|---|
| GRP-QA-046 | Member views linked Lab | Lab shown only if permitted. |
| GRP-QA-047 | Member requests hidden contest/problem | Owning module denies. |
| GRP-QA-048 | Assessment has groupId | Display follows Assessment rules. |
| GRP-QA-049 | Student views another student's attempt | Denied. |

**Open:** D11, activity visibility and membership freeze policy.

## 9. Business rules summary

| Rule | Rule | Status |
|---|---|---|
| GRP-BR-001 | Role and record scope both apply; page access alone is not authorization. | Source-confirmed RBAC |
| GRP-BR-002 | Backend enforces college/department/section/group/ownership scope. | Source-confirmed RBAC |
| GRP-BR-003 | Student private records are ownership-scoped. | Source-confirmed RBAC |
| GRP-BR-004 | Groups mediate collaborative Labs and Contests. | Source-confirmed Business Flow |
| GRP-BR-005 | Student list defaults to own membership, not open directory. | Recommendation |
| GRP-BR-006 | Inactive Group cannot accept joins or new activity links. | Recommendation |
| GRP-BR-007 | Prevent duplicate student/Group membership; allow multiple Groups per student. | Recommendation |
| GRP-BR-008 | Deactivation preserves linked history. | Recommendation |
| GRP-BR-009 | Group association does not create shared submission/score/rank. | Derived + recommendation |
| GRP-BR-010 | Invite code is not exposed through general APIs/logs. | Security recommendation |
| GRP-BR-011 | Each linked activity independently enforces access rules. | Derived from RBAC |

## 10. Data and state reference

### 10.1 Group entity (source-confirmed schema)

| Field | Type | Notes |
|---|---|---|
| id | varchar | Group identifier |
| collegeId | varchar | College association |
| departmentId | varchar | Department association |
| createdById | varchar | Creator |
| name | varchar(255) | Group name |
| description | text | Description |
| inviteCode | varchar(20) | Invite code |
| isActive | boolean | Active flag |
| createdAt | timestamp | Creation timestamp |
| updatedAt | timestamp | Update timestamp |

Recommended editability: name/description editable; tenant/department/creator system-controlled; invite code and isActive separate actions. This is not source-approved.

### 10.2 Related source entities
- **GroupMember:** id, groupId, studentId, joinedAt.
- **GroupLab:** groupId, labAssignmentId, createdAt.
- **GroupContest:** groupId, contestId, createdAt.
- **GroupCourseProblem:** groupId, courseId, problemId, createdAt.
- **Assessment:** includes groupId.

### 10.3 Schema gaps
- Group has departmentId but no sectionId.
- No Group-to-Manager assignment relation.
- GroupMember has no status/leftAt/removedAt/role.
- No invite-code expiry/revocation/use-limit fields.
- No Group capacity field.
- No group-specific submission/score/ranking entity.
- No source-defined uniqueness constraints for Group name, inviteCode, or membership pair.
- These are schema observations, not proof of deployed constraints.

### 10.4 Proposed state behavior

| State | Current representation | Proposed behavior | Status |
|---|---|---|---|
| Active | isActive=true | Operational and joinable | Recommendation |
| Inactive | isActive=false | No new joins/assignments; preserve history | Recommendation |
| Deleted | No explicit deleted state | Prefer deactivate over physical delete | Recommendation |
| Member joined | GroupMember row | Member access under module rules | Recommendation |
| Member left/removed | No explicit status | Decide hard delete vs history/state model | TBD |

Group/member state transitions require product approval and coordination with Module 17.

## 11. Audit, notifications, privacy

### 11.1 Source facts
RBAC identifies Group membership/access changes as sensitive actions to consider for audit. Schema includes AuditLog fields: action, actorId, targetId, targetType, metadata, ipAddress, userAgent, createdAt.

### 11.2 Recommended audit candidates
GROUP_CREATED, GROUP_UPDATED, GROUP_ACTIVATED, GROUP_DEACTIVATED, GROUP_INVITE_CODE_REGENERATED, GROUP_MEMBER_ADDED, GROUP_MEMBER_REMOVED, GROUP_MEMBER_LEFT, GROUP_LAB_ASSIGNED/UNASSIGNED, GROUP_CONTEST_ASSIGNED/UNASSIGNED, GROUP_COURSE_PROBLEM_ASSIGNED/UNASSIGNED.

Final event naming/metadata/retention belongs to Module 15.

### 11.3 Privacy
- Log actor/target/action/scope IDs as approved.
- Never log invite code or submitted join code.
- Avoid unnecessary student personal data, descriptions, or source code in audit/telemetry.

### 11.4 Notifications
No Group notification triggers/recipients are defined in sources. Recommendation: defer to Module 13; do not send implicit notifications for Group CRUD/membership/activity links until event/recipient policy is approved.

## 12. Cross-module dependencies

| Module | Dependency |
|---|---|
| 1 Authentication & User Lifecycle | Authenticated actor, role, account status |
| 2 College / Department / Section | Scope hierarchy; Manager scope gap |
| 3 Subscription & Seat Management | Potential student/college eligibility; not defined by Group sources |
| 4 Course & Enrollment | Course, enrollment, CourseProblem eligibility |
| 6 Code Submission & Judge0 | Student-owned CodeSubmission |
| 8 Labs & Lab Scoring | LabAssignment/LabSubmission behavior |
| 9 Assessments | Assessment.groupId, attempts, timing, individual scores |
| 11 Contests | Contest participation/scoring/ranking |
| 13 Notifications | Notification triggers/recipients |
| 15 Audit Logging | Canonical event catalog and retention |
| 17 Functional State Machines | Group/member lifecycle |
| 18 Edge Cases & Failure Handling | Shared error, retry, conflict, concurrency policy |

## 13. Edge cases / expected handling

| Case | Treatment | Status |
|---|---|---|
| Foreign college/department Group ID | Deny without data disclosure | Source-confirmed security principle |
| Manager lacks explicit Group scope | Deny until mapping resolved | Recommendation |
| Inactive Group | No new join/association | Recommendation |
| Duplicate membership | Prevent | Recommendation |
| Stale/regenerated invite code | Reject | Recommendation |
| Group deactivated during activity | Preserve history; ongoing access TBD | Recommendation/TBD |
| Member removed after submissions | Preserve student-owned history | Recommendation |
| Cross-scope activity association | Deny | Derived from RBAC |
| Duplicate activity link | Prevent | Recommendation |
| Archived linked activity | Follow owning module | Derived |
| Concurrent join/deactivate/assign | Final state must be authorization-consistent; implementation strategy TBD | TBD |
| Physical Group deletion | Prefer deactivation; destructive deletion needs explicit policy | Recommendation |

## 14. Decision log

**Decision owner:** Product Owner; Security/Engineering/Academic owners as indicated. All recommendations are pending review.

| ID | Question / conflict | Best recommendation | Impact | Owner | Status |
|---|---|---|---|---|---|
| GRP-D01 | MANAGER permission conflict: platform Full CRUD*, Page/API V/C/U*. | V/C/U only; deny Delete until explicit approval. | Permission implementation/tests. | Product + Security | Recommendation pending review |
| GRP-D02 | Meaning of Group Delete. | Map delete to logical deactivate; no hard delete with history. | Retention, references, lifecycle. | Product + Engineering | Recommendation pending review |
| GRP-D03 | Manager-to-Group/Section scope; Group has no sectionId/manager relation. | Add explicit GroupManager authorization relation or equivalent; never infer from creator/member list. Confirm whether section mapping is required. | **Blocking** safe Manager access. | Product + Security + Engineering | Recommendation pending review |
| GRP-D04 | Student Group discovery/list. | My Groups defaults to own memberships; joining uses valid code or explicit approved mechanism. | Privacy/discovery. | Product + Security | Recommendation pending review |
| GRP-D05 | Group name uniqueness. | Unique among active Groups within a department; trim/normalize comparison. | Duplicate names and constraint design. | Product + Engineering | Recommendation pending review |
| GRP-D06 | Multiple Groups and student eligibility. | Allow multiple Groups; prevent duplicate membership per Group; same college/department by default. | Membership/authorization. | Product + Academic | Recommendation pending review |
| GRP-D07 | Leave/removal and active activity effects. | Future Group access changes; preserve submissions/results; define ongoing attempt policy. | Contest/Assessment integrity/history. | Product + Academic | Recommendation pending review |
| GRP-D08 | Invite-code lifecycle. | Opaque ≤20-char code; one current code; regeneration invalidates old; active Group only; no expiry unless schema supports it. | Join security/schema. | Product + Security | Recommendation pending review |
| GRP-D09 | Lab/Contest assignment lifecycle. | Scope-compatible association; no duplicates; restrict unassignment after activity begins/records exist pending owning-module rules. | Activity access. | Product + Academic | Recommendation pending review |
| GRP-D10 | GroupCourseProblem semantics. | Treat as Group association; preserve CourseProblem/enrollment rules. | Student problem visibility. | Product + Academic | Recommendation pending review |
| GRP-D11 | Assessment.groupId semantics. | Group targeting only; attempts/scores remain student-level unless Assessment module approves group attempts/scoring. | Assessment eligibility/scoring. | Product + Academic | Recommendation pending review |
| GRP-D12 | Finance Team visibility. | Read-only Group metadata in assigned college; no code/member private data. | Privacy and UI. | Product + Security | Recommendation pending review |
| GRP-D13 | Group capacity. | No capacity limit unless business requirement exists; add field only if needed. | Enrollment behavior. | Product | Open |
| GRP-D14 | Group notifications. | Defer triggers/recipients to Module 13. | Notification behavior. | Product | Deferred to Module 13 |
| GRP-D15 | Audit event catalog. | Use candidate events in Section 11, finalize Module 15. | Audit completeness/privacy. | Security + Product | Deferred to Module 15 |
| GRP-D16 | Manager Group creation. | Default deny until explicit scope is established; then only assigned scope. | Create permission. | Product + Security | Recommendation pending review |
| GRP-D17 | Membership history model. | Prefer membership status/leftAt if history needed; otherwise explicitly approve hard-delete semantics and audit. | Rejoin/history/active activities. | Product + Engineering | Recommendation pending review |

## 15. Module-level acceptance criteria

1. All Group reads/writes enforce role and scope server-side.
2. SUPER_ADMIN is global; HOD is college+department; Manager is explicitly scoped; Student access is membership/ownership scoped.
3. Finance is read-only and college-scoped unless approved otherwise.
4. Client-supplied tenant/creator IDs cannot bypass authorization.
5. Deactivation preserves membership and activity history under recommended behavior.
6. Invite codes are excluded from general APIs and logs.
7. Valid join creates one membership; invalid/stale/inactive/out-of-scope join creates none.
8. Membership changes do not erase student-owned submissions, participation, or scores.
9. GroupLab/GroupContest/GroupCourseProblem links are independently authorized and create no implicit submissions/participation/scores.
10. Group membership does not override Lab/Contest/Course/Problem/Assessment access.
11. MANAGER Delete remains disabled until GRP-D01 is resolved.
12. Manager-scoped Group operations are not implementation-ready until GRP-D03 is resolved.

## 16. QA coverage summary

Function QA cases GRP-QA-001–049 cover:
- role matrix and scope;
- tenant/department/section/group/ownership checks;
- Manager permission discrepancy;
- CRUD validation and spoofed tenant IDs;
- active/inactive lifecycle and preservation;
- invite-code privacy, regeneration, stale-code rejection, length;
- join eligibility, duplicate membership, impersonation;
- member add/remove/leave and historical records;
- Group-Lab/Contest/CourseProblem association and duplicate prevention;
- Assessment visibility and individual ownership;
- concurrency and logging privacy.

## 17. Review and approval checklist

- [ ] Product Owner reviewed MANAGER permission conflict.
- [ ] Product Owner approved/revised recommendations in GRP-D01–D17.
- [ ] Security reviewed Manager scope, invite codes, and membership checks.
- [ ] Engineering confirmed schema gaps and any proposed migration (e.g. GroupManager/membership history).
- [ ] Academic owner reviewed Group-Lab, Contest, Course Problem, and Assessment behavior.
- [ ] Owners of Modules 4, 8, 9, 11, 13, 15, 17, 18 reviewed dependencies.
- [ ] Acceptance criteria and QA cases reviewed.
- [ ] Target behavior explicitly approved before implementation.

**Readiness:** Detailed draft for later product review. Continue the all-modules-first authoring sequence; do not resolve GRP-D01–D17 during this pass. Not implementation-ready while permission, scope, membership, activity, and schema decisions remain unresolved.

## 18. Function-level specifications and QA companion\n\n- [Function-Level Specifications](./FUNCTION-SPECIFICATIONS.md) — GRP-F001–GRP-F011, detailed function templates, permissions, inputs, validations, flows, backend/data behavior, failure handling, acceptance criteria, and dependencies.\n- [QA Test Cases](./QA-TEST-CASES.md) — GRP-QA-001–GRP-QA-132.\n\n## 19. Source references

1. [RBAC source](../../docs/source-text/RBAC.md) — permission matrices, role scopes, tenant/section/group enforcement, audit/security rules.
2. [Business Flow v2](../../docs/source-text/Business%20Flow%20v2.md) — Groups mediate Labs/Contests and HOD/Manager/Student flows.
3. [Page-to-DB Mapping v2](../../docs/source-text/Page%20-%20DB%20Mapping%20v2.md) — HOD Groups, Manager Groups/Classes, Student My Groups.
4. [Schema](../../docs/source-text/scheema.md) — Group and related entities.
5. [Functional Specification Standard](../FUNCTIONAL-SPECIFICATION-STANDARD.md) — evidence labels and approval workflow.

---
