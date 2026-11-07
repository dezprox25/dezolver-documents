# DEZOLVER — Module 10: Groups & Membership — Function-Level Specifications

**Document ID:** DEZ-GRP-FS-002  
**Version:** 0.3  
**Status:** Detailed draft aligned to Functional Specification Standard v1.0; product/security/technical decisions remain open  
**Approval status:** Not approved for implementation  
**Parent:** [Module 10 README](./README.md)  
**QA companion:** [QA Test Cases](./QA-TEST-CASES.md)

> This document expands the source-grounded Module 10 draft into function-level detail. The schema and supplied documents establish entities, page mappings, role scope, and some high-level workflow context. They do not establish every proposed UI or lifecycle behavior. Recommendations remain proposals, not approved target requirements. A GitHub commit versions this draft only.

## 1. Source and evidence baseline

| Evidence | Source-supported facts | Limits |
|---|---|---|
| Business Flow v2 | Groups act as a mediator for collaborative Labs and Contests; HOD, Manager, and Student flows reference Groups/members. | Does not define detailed Group CRUD, join-code lifecycle, membership termination, or assignment rules. |
| Business Flow (original) | Also describes Group-mediated Lab/Contest flow. | Does not resolve lifecycle or permission conflicts. |
| Page-to-DB Mapping v2 | HOD Groups page; Manager Groups/Classes; Student My Groups; mapping includes Group, GroupMember, User, Department, GroupLab, GroupContest. GroupCourseProblem appears in Courses & Labs mapping. | Entity mapping does not define UI controls, field visibility, filters, or API behavior. |
| Page-to-DB Mapping (original) | Groups page for HOD; Groups/Classes for Manager; My Groups for Student; related entities mapped. | Original mapping does not list GroupCourseProblem on the Groups page. |
| RBAC.md | SUPER_ADMIN global; Finance assigned College; HOD College+Department; Manager College+Department+Section/permitted Group; Student own/permitted academic resources. Backend role+scope+ownership checks required. | Manager Group permission differs between RBAC matrices; exact Group authorization mapping unresolved. |
| Schema | Group, GroupMember, GroupLab, GroupContest, GroupCourseProblem, Assessment.groupId fields/relations. | Conceptual schema is not proof of deployed constraints. No GroupManager/Section assignment, membership status/end-date, capacity, or invite-code lifecycle fields are shown. |
| Functional Specification Standard v1.0 | Requires complete function-level sections, traceable evidence, testable criteria, QA, and decisions. | No unapproved recommendation may be represented as approved target behavior. |
| Historical Module 10 Cohort specification | Reports legacy Cohort list/create, email invitation, roster import, scalar membership, syllabus overlay, WebSocket, and JWT-claim behavior/defects. | The artifact is reviewed as historical secondary evidence; raw cited application files were not independently fetched in this update. It reports no target Group implementation. |

Evidence labels:
- **Source-confirmed:** directly stated by a supplied source.
- **Derived:** follows from source-supported facts, but is not stated verbatim.
- **Recommendation:** proposed behavior; approval is still required.
- **TBD / Decision Required:** unresolved or conflicting evidence.

### 1.1 Legacy functional-specification cross-check — reported Cohort functions

**Evidence status:** LEGACY-SPEC-REPORTED. The historical artifact at [module-10-groups/SPECIFICATION.md](../../functional-spec-from-old%20code/module-10-groups/SPECIFICATION.md) labels its reconstruction as completed/forensically verified. This pass reviewed that artifact, not the raw legacy application files it cites. Target Group functions must not be described as implemented merely because a legacy Cohort analogue exists.

| Target function | Historical details reported by the old specification | Required interpretation for this target spec |
|---|---|---|
| GRP-F001 List/filter | `GET /tenants/:id/cohorts`; `read:cohort`; `id=self` resolves to authenticated tenant; repository query filters by tenant. The frontend exposed a Cohort card list; no member count was populated by the repository. | Preserve the general scoped-list/security lesson. There is no legacy Group list route; Group listing remains a target function based on Group/GroupMember and page mappings. |
| GRP-F002 Detail | No Cohort detail endpoint or student drill-down route was reported; Cohort cards had no detail interaction. | No legacy implementation. Details and field-level visibility are recommendations/TBD. |
| GRP-F003 Create | `POST /tenants/:id/cohorts`; `manage:cohort`; `CreateCohortDto` name min 2/max 80, optional description, optional `academicYear` max 20; 201 response. | Create is a workflow analogue only. Target Group schema has collegeId, departmentId, createdById, name `varchar(255)`, description, inviteCode, isActive and timestamps; no academicYear. Do not import legacy DTO validation without a target decision. |
| GRP-F004 Update | Old Cohort API had no PATCH/PUT endpoint. | No legacy Group/Cohort update analogue. Proposed Group metadata edit remains unapproved. |
| GRP-F005 Lifecycle | Legacy Cohort had no status/`isActive` column and no archive/deactivate API; it was reported as create/read only. | The target Group `isActive` field is a conceptual-schema fact; transitions and effects remain recommendations and decision-bound. |
| GRP-F006 Invite-code administration | Legacy invitation endpoint was `POST /tenants/:id/invitations` with email, role, optional cohortId. It was not a Group inviteCode workflow. | No legacy invite-code display/rotation equivalent. Keep inviteCode protected and define its lifecycle in GRP-D08. |
| GRP-F007 Student join | Old invitation acceptance reportedly marked invitation.acceptedAt but did not update identity.users.cohortId. No Group invite-code join was reported. | Student Join is present in target RBAC, but the mechanism is a recommendation. A successful target join must persist GroupMember, not just invitation status. |
| GRP-F008 Membership | Legacy used scalar nullable User.cohortId (0/1 Cohort); no membership join table or direct member add/remove API. Roster worker accepted cohortId, but controller did not pass it in the reported flow. | Do not map GroupMember to User.cohortId. Add/remove/leave, history, multiple Groups, and import are target questions/recommendations; retain GRP-D06/D07/D17 pending. |
| GRP-F009 Lab/Contest association | The historical audit reported no Group, GroupLab, or GroupContest models/operations. | These are target conceptual relations. Use GroupLab/GroupContest and Module 8/11 contracts; no legacy function to copy. |
| GRP-F010 Course Problem association | Legacy Cohort syllabus overlays used `(syllabusId, cohortId, nodeId)` to tailor syllabus nodes. No GroupCourseProblem was reported. | SyllabusOverlay is not GroupCourseProblem. Keep CourseProblem/enrollment eligibility with Module 4. |
| GRP-F011 Activities/Assessment | Legacy Cohort overlays and WebSocket rooms were reported. Its JWT signer omitted `cohort_id`, causing principal cohortId to be undefined; old Assessment had no Group/Cohort scope. | Do not use cohortId claims/rooms as Group authorization. Target Group activity access comes from target Group relations and independent Module 8/9/11 checks. Assessment.groupId is target schema evidence; semantics stay with Module 9. |

**Additional legacy endpoints and defects reported**
- `POST /tenants/:id/invitations` (`manage:cohort`) records optional cohortId; accepting the invitation reportedly does not bind the accepting user to that cohort.
- `POST /tenants/:id/roster/import` returns 202 and enqueues a background import; the worker supports a cohortId parameter, but the controller reportedly does not forward cohortId (or s3Key).
- Cohort context was also used by curriculum syllabus overlays and `cohort:<cohortId>` WebSocket rooms. The old report identifies missing cohort_id claims in both token creation paths; this is a legacy defect, not a target Group contract.
- The old report states that no member-count join was performed and no Group/Cohort member drill-down endpoint existed.

**Function evidence rule:** GRP-F001/F003 have historical Cohort analogues only. GRP-F002/F004–F011 have no equivalent target Group implementation in the old specification; related Cohort capabilities are explicitly distinguished above. Current detailed function requirements remain target proposals derived from current business-flow, page mapping, RBAC, conceptual schema, and cross-module specifications.
## 2. Actors, permissions, and scope

### 2.1 Source-confirmed role scope

| Role | Source-defined scope |
|---|---|
| SUPER_ADMIN | Global platform scope |
| FINANCE_TEAM | Assigned College |
| HOD | Assigned College + Department |
| MANAGER | Assigned College + Department + assigned Section / permitted Group |
| STUDENT | Own records + permitted academic resources |

### 2.2 Source permission comparison

| Role | RBAC platform matrix | Page/API matrices | Draft handling |
|---|---|---|---|
| SUPER_ADMIN | Full CRUD | V/C/U/D or Full CRUD | Source supports global CRUD. |
| FINANCE_TEAM | View | V; API V* | Read-only; Group field visibility remains TBD. |
| HOD | Full CRUD | V/C/U/D; API Full CRUD* | CRUD within College+Department scope. |
| MANAGER | Full CRUD* | V/C/U* | Conflict: Delete appears only in platform matrix. Recommend V/C/U only pending decision GRP-D01. |
| STUDENT | Join/View | V OWN; API V OWN/JOIN | Own/permitted Group view and Join capability; exact join mechanism is not source-confirmed. |

### 2.3 Permission matrix (not final approval)

| Function/action | SUPER_ADMIN | FINANCE_TEAM | HOD | MANAGER | STUDENT |
|---|---|---|---|---|---|
| List/view Groups | Allow, global | Conditional read-only, College | Allow, Department | Conditional, explicit Group scope | Own/permitted Groups |
| View Group details | Allow | TBD fields | Allow | Conditional | Member/permitted view |
| Create Group | Allow | Deny | Allow | Conditional; GRP-D03/D16 | Deny |
| Update Group metadata | Allow | Deny | Allow | Conditional; GRP-D03 | Deny |
| Activate/deactivate | Allow | Deny | Allow | Conditional | Deny |
| View/regenerate invite code | Allow (proposed) | Deny | Allow (proposed) | Conditional | Deny |
| Join by code | Administrative support TBD | Deny | Staff join support TBD | Staff join support TBD | Allow per RBAC Join; workflow TBD |
| Add/remove member | Allow | Deny | Allow within scope | Conditional | Deny other members |
| Leave own Group | N/A | Deny | N/A | N/A | Recommendation; decision GRP-D07 |
| Associate activities | Allow | Deny | Allow | Conditional | Deny |
| Delete Group | Source says CRUD for some roles; exact delete meaning unresolved | Deny | Source says CRUD | Conflicting Manager matrix | Deny |

**Security requirement from RBAC:** Frontend page visibility is not authorization. Every operation must check authenticated role, permission, College/Department/Section/Group scope, and Student ownership/membership where relevant on the backend. Client-supplied IDs must not widen scope.

### 2.4 Cross-cutting assumptions / guardrails

- Do not infer Manager access from creator identity, membership, or guessed section relation.
- Group has Department but no Section field; Group-to-Manager authorization mapping is not represented in the supplied schema.
- Group membership does not grant access to every related Lab, Contest, Course, Problem, Assessment, submission, or result.
- Student private records are owner-scoped.
- No Group deletion, membership leave/removal, invitation lifecycle, capacity, or Group-scoring behavior is considered approved by this document.

## 3. Functional inventory

| Function ID | Function | Primary actor | Entry point | Type | Status |
|---|---|---|---|---|---|
| GRP-F001 | List and filter Groups | Staff / Student | HOD Groups, Manager Groups/Classes, Student My Groups | Read | Detailed draft; filters/UI TBD |
| GRP-F002 | View Group details | Authorized actor | Group list/detail link | Read | Derived; field visibility TBD |
| GRP-F003 | Create Group | SUPER_ADMIN/HOD/conditional Manager | Create Group action | Create | CRUD source-supported; form rules proposed |
| GRP-F004 | Update Group metadata | SUPER_ADMIN/HOD/conditional Manager | Edit Group | Update | CRUD source-supported; editable fields proposed |
| GRP-F005 | Activate/deactivate Group | SUPER_ADMIN/HOD/conditional Manager | Group status action | Lifecycle | Group.isActive source-supported; transitions proposed |
| GRP-F006 | View/regenerate invite code | SUPER_ADMIN/HOD/conditional Manager | Group invite-code control | Security workflow | inviteCode field source-supported; lifecycle proposed |
| GRP-F007 | Join Group by invite code | Student | My Groups → Join | Workflow | Student Join source-supported; mechanism proposed |
| GRP-F008 | Manage members / leave Group | Staff / Student self | Group members / My Groups | Workflow | GroupMember source-supported; operation semantics TBD |
| GRP-F009 | Associate Group with Lab / Contest | Staff | Group activities / owning module | Association | Relationship source-supported; workflow proposed |
| GRP-F010 | Associate Course Problem with Group | Staff | Courses & Labs / Group context | Association | Schema source-supported; workflow proposed |
| GRP-F011 | View linked activities / Assessment reference | Staff / Student member | Group detail / My Groups | Read | Derived; visibility governed by owning modules |

---

# 4. Detailed function specifications

## GRP-F001 — List and filter Groups

**Requirement status:** Source-confirmed page mappings and RBAC scope; filter behavior is Recommendation/TBD.  
**Specification status:** Detailed draft; decision required.  
**Priority:** TBD.  
**Primary actors:** SUPER_ADMIN, FINANCE_TEAM, HOD, MANAGER, STUDENT.  
**Supporting actors:** Authentication/authorization services; Group data service.

### 4.1 Purpose and outcome
Display Group records the actor is permitted to discover. The result must not disclose the existence, count, names, membership, or details of out-of-scope Groups.

### 4.2 Permissions and data scope
- SUPER_ADMIN: global view.
- FINANCE_TEAM: assigned College only; exact Group fields and page entry are TBD (GRP-D12).
- HOD: assigned College + Department.
- MANAGER: assigned College + Department + explicitly permitted Section/Group. Deny until a Group-scope assignment is resolvable (GRP-D03).
- STUDENT: own memberships / permitted academic resources. “My Groups” mapping supports a student Group page; an unrestricted directory is not source-confirmed.
- Backend must apply role and scope before returning records. Search/filter must not bypass scope.

### 4.3 Preconditions
1. Actor is authenticated and account/session is valid under Module 1.
2. Actor's role and scope are resolved.
3. Group view permission is available for the actor.
4. For Student view, authenticated Student identity is known.

### 4.4 Entry point and trigger
- HOD: Groups page.
- MANAGER: Groups / Classes page.
- STUDENT: My Groups page.
- SUPER_ADMIN: administrative Group view (recommended; exact route TBD).
- FINANCE_TEAM: entry point TBD; page mapping does not explicitly map a Finance Group page.

### 4.5 User interface behavior
**Source-confirmed:** Page mapping identifies the pages/entities; it does not prescribe layout.  
**Recommendation:** Render a scoped list. Staff list fields may include Group name, Department, active state, and created date. Student list may include Group name and permitted academic context. Member count, creator, sorting, pagination, and exact labels require approval. Invite code must not appear in list payloads.  
UI states: loading; populated list; no Groups; no matching filter; access denied; load failure. Exact messages are TBD and Module 18-owned.

### 4.6 Inputs and field definitions

| Input | Source | Type | Required | Allowed/constraint | Validation |
|---|---|---|---|---|---|
| Search term | Proposed UI | String | No | Length/normalization TBD | Search only within authorized rows |
| Status filter | Proposed UI | Enum/boolean | No | Active/inactive mapping to isActive | Validate supported values |
| Department filter | Proposed staff UI | ID | No | Only Departments actor may access | Reject/ignore foreign scope |
| Page / page size | Proposed UI | Integer | No | Limits TBD | Enforce bounded values if implemented |
| Sort field/order | Proposed UI | String/enum | No | Allowed fields TBD | Do not accept arbitrary sort expressions |
| collegeId / departmentId / sectionId | Client or auth context | IDs | No (client filters optional) | Auth scope is authoritative | Never widen scope from client input |

### 4.7 Main flow
1. Actor opens the mapped Groups page.
2. UI requests Group list.
3. Backend identifies actor from authenticated context; it does not trust role/user/tenant supplied by the client.
4. Backend evaluates Group-view permission and scope.
5. Backend queries only permitted Group records and applies allowed filters.
6. Backend returns the authorized projection, excluding inviteCode and unrelated private fields.
7. UI renders permitted Groups and actions according to permissions.
8. If no records match, UI displays an empty state without exposing unauthorized counts.

### 4.8 Business rules
- **Source-confirmed:** role scope and database scope both apply.
- **Source-confirmed:** HOD is Department-scoped; Manager is Section/permitted Group-scoped; Student sees own/permitted academic data.
- **Recommendation:** Student “My Groups” lists memberships, not all Groups.
- **Recommendation:** inactive Groups may be visible to authorized staff but should not be presented as joinable to Students.
- **TBD:** Finance list/detail fields; Student directory/search; pagination/sort.

### 4.9 Validation rules and feedback

| Validation ID | Condition | Validation point | Behavior | Feedback |
|---|---|---|---|---|
| GRP-F001-V01 | Actor lacks Group view permission | Server request | Deny page/API | Shared access-denied behavior TBD |
| GRP-F001-V02 | Foreign Department filter | Server | Reject or safely scope-filter; must not return foreign rows | Message intent: filter unavailable/denied; exact copy TBD |
| GRP-F001-V03 | Forged College/role/Student ID | Server | Ignore/reject client scope; derive identity from auth | Security response policy Module 18 |
| GRP-F001-V04 | Invalid status/sort/page input | Server | Reject unsupported value or apply approved safe default | Exact behavior TBD |

### 4.10 System/backend behavior
- Resolve authenticated actor and permission/scope.
- Query Group and relevant scope fields.
- Apply server-side filters, then permitted user-supplied filters.
- Return a privacy-limited response.
- Do not mutate data.
- Query implementation/indexes/pagination contract are technical decisions, not defined by sources.

### 4.11 Data changes and state transitions
None. Reads Group records and may join Department/GroupMember for permitted UI fields. No membership mutation.

### 4.12 Success result
Authorized Group rows and permitted actions are displayed. The exact returned field set is TBD and must exclude inviteCode from generic list responses.

### 4.13 Alternate flows and failures
- No Group records: empty state.
- No filter matches: filtered empty state.
- Foreign Group/Department filter: no foreign record disclosure.
- Session expired: Module 1 session behavior.
- Backend query error: Module 18 failure behavior; no partial unauthorized data.
- Membership changed during request: current authorization should be re-evaluated on subsequent detail/action requests.

### 4.14 Edge cases
- Actor has access to no Groups.
- Group deactivated after list render.
- Student leaves Group after list render.
- Department/Group scope changed after login.
- Search matches both permitted and foreign Group names; only permitted results returned.

### 4.15 Audit, notification, privacy
- Read audit policy belongs to Module 15; no Group-specific read logging requirement is source-confirmed.
- Do not expose inviteCode, hidden membership details, or foreign Group counts.
- No notification.

### 4.16 Acceptance criteria
1. Given an HOD assigned to Department A, when they list Groups, then only Groups in the approved College A / Department A scope are returned.
2. Given a Manager without a resolvable permitted Group mapping, when they list Groups, then the system does not guess scope or expose unassigned Groups.
3. Given a Student, when they open My Groups, then records are limited to their permitted memberships.
4. Given a client-supplied foreign college/department ID, when a list is requested, then the filter cannot broaden authorized results.
5. Given any list response, then inviteCode is absent.

### 4.17 QA test cases
See [QA cases GRP-QA-001–005, 050–056](./QA-TEST-CASES.md).

### 4.18 Unresolved items
GRP-D03, GRP-D04, GRP-D12; list filters/pagination/sort and Finance visibility.

---

## GRP-F002 — View Group details

**Requirement status:** Derived from page mapping/schema; UI fields and visibility are Recommendation/TBD.  
**Specification status:** Detailed draft; decision required.  
**Priority:** TBD.  
**Primary actors:** SUPER_ADMIN, FINANCE_TEAM, HOD, MANAGER, permitted Student member.  
**Supporting actors:** Group service; owning Lab/Contest/Course/Assessment modules.

### 4.1 Purpose and outcome
Show authorized Group metadata and allowed membership/activity summaries without treating Group access as automatic access to every linked resource.

### 4.2 Permissions and data scope
- Apply role/scope rules in Section 2.
- Student detail requires a permitted Group relationship; whether former members retain detail access is TBD.
- Finance visibility remains limited by GRP-D12.
- Each linked activity must independently authorize its data and actions.
- InviteCode must not be exposed in generic Group detail.

### 4.3 Preconditions
Actor authenticated; Group exists; actor has Group view permission and access to that Group.

### 4.4 Entry point and trigger
Select a Group from an authorized list or follow a Group link from a mapped academic page.

### 4.5 User interface behavior
**Recommendation:** Display Group name, description, Department, active/inactive status, member list/summary where authorized, and links to permitted Labs/Contests/Course Problems. Assessment reference is displayed only when Assessment module provides approved Group visibility. Exact fields, member contact data, and inactive Group view are TBD.

### 4.6 Inputs

| Input | Source | Type | Required | Constraints |
|---|---|---|---|---|
| groupId | Route/API | ID | Yes | Must identify an in-scope Group |
| activity filter | Proposed UI | Enum | No | Types/filters TBD |
| page/limit for members | Proposed UI | Integer | No | Pagination contract TBD |

### 4.7 Main flow
1. Actor selects Group.
2. Backend resolves actor and authorizes Group access.
3. Backend loads Group metadata.
4. Backend loads only permitted membership fields.
5. Backend obtains activity references.
6. Each owning module authorizes its linked activity and fields.
7. UI displays permitted Group details, member information, and activity links.

### 4.8 Business rules
- Page mappings confirm Group/GroupMember/User/Department/GroupLab/GroupContest context.
- Group access does not imply access to Student private submissions, results, or hidden Contest/Assessment information.
- Assessment.groupId is a schema reference; it does not establish Group attempt/scoring semantics.
- Member name/email/phone exposure is TBD; default recommendation is least-privilege disclosure.

### 4.9 Validation
| ID | Condition | Behavior |
|---|---|---|
| GRP-F002-V01 | Missing/invalid Group ID | Safe not-found response |
| GRP-F002-V02 | Group exists but outside scope | Deny without confirming existence |
| GRP-F002-V03 | Linked activity outside actor scope | Omit/deny that resource per owning module; no leakage |
| GRP-F002-V04 | Client requests inviteCode field | Do not return it through generic detail |

### 4.10 Backend/system
Read Group, Department, allowed GroupMember data, and association records. Re-authorize linked entities under their owning module. No mutations.

### 4.11 Data/state
Read-only: Group; GroupMember; Department; GroupLab; GroupContest; GroupCourseProblem if supported by approved UI; Assessment.groupId association if approved. No state transition.

### 4.12 Success
Authorized Group detail is displayed with only fields and linked resources permitted to the actor.

### 4.13 Alternate/failure flows
- Group not found/out of scope: safe denial/not-found.
- No members/activities: empty section.
- Linked resource archived/removed: follow owning module.
- Membership revoked between list and detail: re-check access.
- Related service unavailable: show partial data only if partial display is approved; otherwise fail safely (TBD).

### 4.14 Edge cases
Inactive Group; former member; activity archived; student has membership but not activity eligibility; Group links to Assessment with different assignment rules.

### 4.15 Audit/privacy
No source-defined detail-view audit event. Module 15 decides read audit. Do not expose inviteCode, hidden data, other Student private records.

### 4.16 Acceptance criteria
- HOD in scope can view permitted Group details.
- Foreign Group is not disclosed.
- Student receives only student-safe fields and own/permitted membership context.
- Activity links do not bypass owning module permissions.
- Generic detail never returns inviteCode.

### 4.17 QA
See GRP-QA-006–009 and GRP-QA-057–061.

### 4.18 Decisions
GRP-D03, GRP-D12; member field visibility; inactive/former-member detail access; partial rendering.

---

## GRP-F003 — Create Group

**Requirement status:** RBAC CRUD is source-supported for SUPER_ADMIN/HOD; Manager CRUD appears in platform matrix but scope conflict remains. Field form/defaults are Recommendation.  
**Specification status:** Detailed draft; decision required.  
**Primary actors:** SUPER_ADMIN, HOD; Manager conditional.  
**Supporting actors:** College/Department scope service; ID/code generation.

### 4.1 Purpose and outcome
Create a Group record within the actor’s authorized College/Department/Group scope.

### 4.2 Permissions and scope
- SUPER_ADMIN: global.
- HOD: assigned College + Department.
- Manager: deny/hold until GRP-D03 and GRP-D16 establish Group create scope; do not infer it from Section.
- Finance/Student: deny.
- College/Department/createdBy must be backend-controlled or selected only from authorized scope.

### 4.3 Preconditions
Authenticated actor; create permission; target Department exists and is in scope; Department/College eligibility is confirmed by Module 2. Subscription/capacity checks are not defined by Group sources.

### 4.4 Entry point
Create Group action on authorized Groups page.

### 4.5 UI behavior
**Recommendation:** Form fields: Name (required), Description (optional), Department (only if actor has more than one permitted choice). College and creator derived from auth/scope. Invite code generated by backend. Initial active status is proposed, not approved. Exact labels, confirmation, and success route TBD.

### 4.6 Inputs

| Field | Source/type | Required | Constraint | Validation/status |
|---|---|---|---|---|
| name | Group.name varchar(255) | Recommended required | Trim; nonblank; max 255 chars | Requiredness/trim behavior proposed |
| description | Group.description text | No, proposed | Length limit not specified | TBD |
| collegeId | Group.collegeId varchar | System | Must be authorized College | Never trust arbitrary client value |
| departmentId | Group.departmentId varchar | System/authorized selection | Must be within HOD/Manager scope | Manager scope TBD |
| createdById | Group.createdById varchar | System | Authenticated actor | Client must not set |
| inviteCode | Group.inviteCode varchar(20) | System | ≤20 chars; code generation rules TBD | Do not accept client code |
| isActive | Group.isActive boolean | System default | Default value TBD; active=true recommendation | Decision |
| id, createdAt, updatedAt | Schema fields | System | Generation/clock policy TBD | Technical contract |

### 4.7 Main flow
1. Actor opens Create Group.
2. UI obtains permitted Department choices.
3. Actor enters proposed metadata and submits.
4. Backend authenticates actor and re-checks create permission.
5. Backend derives/validates College and Department scope.
6. Backend validates name/description and approved uniqueness rules.
7. Backend creates Group and generates inviteCode if the approved design requires it at creation.
8. Backend records createdById and timestamps.
9. UI shows success and Group detail/list.
10. No GroupMember/activity association is created implicitly unless a separate approved decision says otherwise.

### 4.8 Business rules
- Group schema has collegeId, departmentId, createdById, name, description, inviteCode, isActive, createdAt, updatedAt.
- Group creation does not imply the creator becomes a member.
- Unique name within Department is only a recommendation (GRP-D05).
- Manager create permission is not implementation-ready pending GRP-D03/D16.
- No capacity field or Group limit is source-confirmed (GRP-D13).

### 4.9 Validation
| ID | Condition | Behavior | Feedback |
|---|---|---|---|
| GRP-F003-V01 | Actor lacks create permission | Deny; no record | Access denied, exact message TBD |
| GRP-F003-V02 | Missing/blank name | Reject if required rule approved | Required field intent; exact copy TBD |
| GRP-F003-V03 | Name exceeds 255 chars | Reject based on schema limit | Field error |
| GRP-F003-V04 | Department foreign/out of scope | Deny | Do not disclose foreign Department |
| GRP-F003-V05 | Client sends collegeId/createdById/inviteCode | Ignore/reject client override | Security-safe response |
| GRP-F003-V06 | Name duplicate | Apply GRP-D05 only after approval | Decision-blocked |
| GRP-F003-V07 | Invite-code collision | Retry generation if generation approach approved | Internal retry; no code leakage |

### 4.10 Backend/system
- Resolve actor, permission, and scope.
- Validate target Department and College relationship.
- Create Group record with server-controlled identifiers/creator/timestamps.
- Generate inviteCode if required by approved join workflow.
- Ensure record creation is atomic; no partial Group record if required creation step fails.
- Idempotency/retry contract is TBD under Module 18.

### 4.11 Data/state
Create one Group row. Related GroupMember/GroupLab/GroupContest/GroupCourseProblem rows are not created by default. Field default values remain open.

### 4.12 Success
Group is created in the permitted scope; the actor receives Group ID and authorized metadata.

### 4.13 Alternate/failure flows
- Validation or authorization fails: no Group created.
- Department unavailable/deactivated: reject per Module 2 policy.
- Duplicate name: behavior depends on GRP-D05.
- Code generation failure: do not create incomplete record if code is mandatory; policy TBD.
- Request retry after timeout: duplicate prevention/idempotency TBD.
- Concurrent same-name creation: uniqueness/locking policy TBD.

### 4.14 Edge cases
HOD missing Department assignment; multi-Department HOD; Manager has Section assignment but no Group mapping; very long description; simultaneous creation; inactive Department.

### 4.15 Audit/privacy
Recommended event GROUP_CREATED; final event name/metadata/retention belongs to Module 15. Do not log inviteCode or full description unless approved.

### 4.16 Acceptance criteria
- Authorized HOD creates Group in assigned Department.
- Student and Finance cannot create Group.
- Foreign Department/College cannot be supplied to bypass scope.
- Name validation follows approved rule and schema limit.
- Create does not implicitly add members or activities.
- Manager create remains blocked until scope decision is approved.

### 4.17 QA
See GRP-QA-010–014 and GRP-QA-062–068.

### 4.18 Decisions
GRP-D03, D05, D13, D16; initial isActive; invite-code creation; idempotency.

---

## GRP-F004 — Update Group metadata

**Requirement status:** CRUD source-supported for relevant roles; mutable field policy is Recommendation.  
**Specification status:** Detailed draft; decision required.  
**Primary actors:** SUPER_ADMIN, HOD; Manager conditional.

### 4.1 Purpose
Edit Group metadata without changing tenant/department ownership or related membership/activity relationships unintentionally.

### 4.2 Permissions/scope
SUPER_ADMIN global; HOD College+Department; Manager only within explicitly assigned Group scope after GRP-D03; Finance/Student deny.

### 4.3 Preconditions
Group exists; actor may update it; Group in actor scope.

### 4.4 Entry point
Group detail/list → Edit.

### 4.5 UI behavior
**Recommendation:** Name and description are editable in general form. College, Department, creator, inviteCode, and isActive are not editable through this form. Department transfer should be a separate approved workflow, not an ordinary metadata edit.

### 4.6 Inputs

| Input | Source/type | Required | Constraints |
|---|---|---|---|
| groupId | Route/API ID | Yes | In-scope Group |
| name | varchar(255) | Recommended required | Nonblank/max 255 |
| description | text | No | Maximum length TBD |
| collegeId, departmentId, createdById | Schema | No user edit | Scope/system-controlled |
| inviteCode/isActive | Schema | No user edit here | Dedicated functions F005/F006 |

### 4.7 Main flow
1. Actor selects Edit.
2. Backend authorizes Group and returns editable fields.
3. UI loads current values.
4. Actor changes allowed metadata and submits.
5. Backend re-authorizes and validates Group scope and values.
6. Backend updates only approved mutable fields and updatedAt.
7. UI displays saved values.

### 4.8 Business rules
- General edit must not change collegeId, departmentId, createdById, inviteCode, or isActive unless separate approved workflow.
- Department transfer requires a dedicated policy covering membership and activities.
- Duplicate-name policy is GRP-D05.
- Concurrency/stale-edit behavior is TBD.

### 4.9 Validation
| ID | Condition | Behavior |
|---|---|---|
| GRP-F004-V01 | Group missing/out of scope | Deny safely |
| GRP-F004-V02 | Name blank/too long | Reject |
| GRP-F004-V03 | Foreign tenant fields supplied | Reject/ignore; scope unchanged |
| GRP-F004-V04 | Duplicate name | Decision-blocked pending GRP-D05 |
| GRP-F004-V05 | Concurrent edit/stale version | Apply Module 18 conflict policy; TBD |

### 4.10 Backend/system
Read current Group, authorize, validate allowed fields, update fields and updatedAt. Do not update association or membership tables.

### 4.11 Data/state
Group.name and Group.description may change after approved field policy; updatedAt changes. No other Group/relationship data changes.

### 4.12 Success
Updated Group metadata is returned/displayed.

### 4.13 Alternate/failure
Permission failure, validation failure, Group deleted/deactivated, stale update, no-op update, service failure. Failed update must not partially mutate Group.

### 4.14 Edge cases
Name normalization/uniqueness; simultaneous HOD/Manager updates; attempted Department transfer; update inactive Group; description containing unsafe content (content policy/escaping TBD).

### 4.15 Audit/privacy
GROUP_UPDATED recommended with actor, Group, scope, changed field names. Avoid storing full before/after descriptions in audit unless Module 15 approves.

### 4.16 Acceptance criteria
- Authorized actor can update approved metadata.
- Foreign Group update denied.
- Tenant/Department/creator cannot be changed through general edit.
- Invalid update leaves stored record unchanged.
- Related memberships and activity associations remain unchanged.

### 4.17 QA
See GRP-QA-015–018 and GRP-QA-069–073.

### 4.18 Decisions
GRP-D03, D05; Department transfer; editable fields; concurrency policy.

---

## GRP-F005 — Activate/deactivate Group

**Requirement status:** Group.isActive exists in schema; lifecycle behavior is Recommendation.  
**Specification status:** Detailed draft; decision required.  
**Primary actors:** SUPER_ADMIN, HOD; Manager conditional.

### 4.1 Purpose
Change Group operational availability without silently destroying membership or activity history.

### 4.2 Permissions/scope
SUPER_ADMIN global; HOD within Department; Manager conditional scope; Finance/Student deny.

### 4.3 Preconditions
Group exists; actor can manage lifecycle; current Group state known.

### 4.4 Entry point
Group detail/list → status action.

### 4.5 UI behavior
**Recommendation:** Show current state and confirm deactivation. Exact confirmation copy, reactivation flow, and warning content TBD. Staff may see inactive Groups; Student joinability behavior is not source-confirmed.

### 4.6 Inputs
- groupId (required).
- desired isActive boolean (server validates allowed transition).
- Optional confirmation token/action acknowledgment if approved.

### 4.7 Main flow
1. Actor requests activate/deactivate.
2. Backend authenticates and authorizes actor.
3. Backend loads current Group state.
4. Backend checks transition and any active Lab/Contest/Assessment restrictions.
5. Backend updates isActive and updatedAt.
6. UI shows updated status.

### 4.8 Business rules
**Recommendation:** Deactivation is logical, not physical deletion; preserve GroupMember and Group activity associations. No new joins/assignments to inactive Group. Reactivation does not automatically restore previously removed membership. Active event restrictions require owner-module decisions. These are not approved until GRP-D02/D07/D09/D11 resolved.

### 4.9 Validation
| ID | Condition | Behavior |
|---|---|---|
| GRP-F005-V01 | Actor lacks lifecycle permission | Deny |
| GRP-F005-V02 | Group out of scope | Deny |
| GRP-F005-V03 | Invalid state value | Reject |
| GRP-F005-V04 | Active activity restriction applies | Block/warn according to approved rule; TBD |
| GRP-F005-V05 | Repeated same-state request | Recommended idempotent no-op; approval TBD |

### 4.10 Backend/system
Re-check scope, current state, and active related activity constraints. Update Group.isActive/updatedAt only. Keep related rows intact under the recommendation.

### 4.11 Data/state
Proposed transition: active=true → inactive=false; inactive=false → active=true. Physical deletion is not part of this function. State machine must be finalized in Module 17.

### 4.12 Success
Group state is updated and visible to authorized staff.

### 4.13 Alternate/failure
Unauthorized actor; Group not found; already in requested state; activity running; concurrent join/assignment; backend error. Failed transition must not leave partially updated relationships.

### 4.14 Edge cases
Group deactivated during active contest/assessment; existing student members; reactivation; code rotation; concurrent Join; inactive Department.

### 4.15 Audit/privacy
GROUP_ACTIVATED/GROUP_DEACTIVATED recommended; event catalog/retention Module 15. No inviteCode in logs.

### 4.16 Acceptance criteria
- Authorized staff can request status change only within scope.
- Deactivation preserves historical relationships under approved preservation rule.
- Inactive Group join/association behavior follows approved policy.
- Unauthorized actor cannot change state.
- Active activity impact is not silently decided.

### 4.17 QA
See GRP-QA-019–022 and GRP-QA-074–079.

### 4.18 Decisions
GRP-D02, D07, D09, D11; inactive Group visibility and transition rules.

---

## GRP-F006 — View/regenerate invite code

**Requirement status:** Group.inviteCode varchar(20) source-confirmed; code lifecycle and security controls are Recommendation/TBD.  
**Specification status:** Detailed draft; security review required.  
**Primary actors:** SUPER_ADMIN, HOD; Manager conditional.  
**Supporting actors:** Secure code generation; authorization; audit.

### 4.1 Purpose
Allow authorized Group managers to retrieve the current invite code and, if approved, rotate it so that a new code can be used to request membership.

### 4.2 Permissions/scope
- Staff must have Group management permission and Group scope.
- Student/Finance/general Group APIs must not expose inviteCode.
- Manager access remains conditional on GRP-D03.
- Exact staff code-view permission is not explicit in RBAC; this is a recommendation requiring GRP-D08/D12 approval.

### 4.3 Preconditions
Group exists and actor has code-management permission. Active-only restriction is recommended.

### 4.4 Entry point
Dedicated invite-code action within Group management view.

### 4.5 UI behavior
**Recommendation:** code appears only in dedicated management view; regenerate action requires confirmation. Generic list/detail responses omit code. Exact reveal/copy controls, code reveal duration, and warning are TBD.

### 4.6 Inputs

| Input | Type | Required | Constraints |
|---|---|---|---|
| groupId | ID | Yes | In-scope Group |
| action | enum | Yes | View / Regenerate (if approved) |
| confirmation | boolean/token | Regenerate only if policy requires | UX/security contract TBD |
| inviteCode | system-generated | No client input | varchar(20); exact alphabet/entropy not specified |

### 4.7 Main flow
**View:** authorize → read current code → return through narrowly scoped response → display to staff.  
**Regenerate:** authorize → verify current Group state → generate candidate code → ensure accepted uniqueness/collision behavior → update inviteCode/updatedAt atomically → invalidate prior code → return new code to authorized staff.

### 4.8 Business rules
- Schema max length is varchar(20).
- Recommend unpredictable opaque code, no human-readable PII.
- Recommend one current code per Group and old code invalid after regeneration.
- Expiry, revocation, usage limits, code entropy, rate limiting, lockout, and code creation timing are not source-confirmed.
- Do not log code in AuditLog or general telemetry.

### 4.9 Validation
| ID | Condition | Behavior |
|---|---|---|
| GRP-F006-V01 | Actor not authorized | Deny |
| GRP-F006-V02 | Group foreign/out of scope | Deny without code disclosure |
| GRP-F006-V03 | Group inactive | View/regenerate policy TBD; recommendation to block new rotation |
| GRP-F006-V04 | Code exceeds 20 characters | Reject generated value; generate valid code |
| GRP-F006-V05 | Collision | Retry or use approved uniqueness policy |
| GRP-F006-V06 | Old code used after rotation | Reject under recommended immediate invalidation |

### 4.10 Backend/system
Use a secure generation mechanism; check uniqueness as required by approved design; atomically update Group.inviteCode and timestamp; avoid returning code except in authorized response. Rate limit/replay controls are security decisions.

### 4.11 Data/state
Group.inviteCode changes on rotation. Group.updatedAt updates. No GroupMember created. The old code's validity ends under recommended behavior.

### 4.12 Success
Authorized staff sees current code or the newly rotated code. Other roles receive no code.

### 4.13 Alternate/failure
- Permission denied: no code returned.
- Collision/generation failure: retry safely; no invalid/partial code stored.
- Concurrent regenerate: final code must be coherent; policy/locking TBD.
- Client timeout after rotation: current code may have changed; recovery/retrieval flow TBD.
- Inactive Group: follow approved state policy.

### 4.14 Edge cases
Code leaked; repeated rotation; multiple staff rotate concurrently; old code already submitted; code accidentally appears in analytics/logging; Group is deactivated during request.

### 4.15 Audit/privacy
GROUP_INVITE_CODE_REGENERATED recommended. Log actor, Group ID, scope, timestamp only; never raw code or code fragment. Event catalog Module 15.

### 4.16 Acceptance criteria
- Unauthorized actors cannot view/regenerate.
- Generic Group list/detail payloads omit inviteCode.
- Generated code length is at most 20 characters.
- Rotation invalidates old code only if that policy is approved.
- Audit event contains no raw code.

### 4.17 QA
See GRP-QA-023–026 and GRP-QA-080–085.

### 4.18 Decisions
GRP-D08, D12; code expiry/revoke/use limits, rate limiting, generation strategy, rotation semantics.

---

## GRP-F007 — Join Group by invite code

**Requirement status:** Student Join permission source-confirmed; invite-code workflow is Recommendation.  
**Specification status:** Detailed draft; eligibility/security decisions open.  
**Primary actor:** STUDENT.  
**Supporting actors:** Group service; User/College/Department scope; secure invite-code validation.

### 4.1 Purpose
Allow an authenticated Student to join a Group through a valid invite code, subject to approved eligibility and scope rules.

### 4.2 Permissions and scope
- RBAC grants Student Group Join/View in its matrices.
- Student identity must come from authenticated session/token, not client-supplied studentId.
- Group must be within approved College/Department/academic eligibility.
- Cross-department/cross-college membership is not approved by the source; default restriction is a recommendation.
- Group outside scope must not be disclosed merely because the code is guessed.

### 4.3 Preconditions
Authenticated active Student; Join action enabled; code supplied; Group exists and is joinable according to approved state policy. Student eligibility based on account status, College, Department, Section, batch, enrollment, and subscription is TBD.

### 4.4 Entry point
Student My Groups → Join Group (recommended UI). Exact route/label is TBD.

### 4.5 UI behavior
**Recommendation:** provide code input; do not reveal Group name/details before server validates the code. After successful join, show the Group in My Groups. Invalid code feedback should not reveal whether a code corresponds to a Group in another College.

### 4.6 Inputs

| Input | Type | Required | Constraint |
|---|---|---|---|
| inviteCode | string | Yes | ≤20 chars (schema length); whitespace/case handling TBD |
| studentId | ID | No client input | Authenticated identity only |
| groupId | ID | No client input for code join | Resolved from code server-side |

### 4.7 Main flow
1. Student opens Join Group.
2. Student submits inviteCode.
3. Backend authenticates Student and validates account status.
4. Backend resolves code to Group without exposing foreign Group details.
5. Backend checks Group status, College/Department/Section eligibility, and join permission.
6. Backend checks existing membership.
7. Backend creates GroupMember with groupId, authenticated studentId, joinedAt if eligible and not already member.
8. Backend returns success and permitted Group information.
9. UI refreshes My Groups.

### 4.8 Business rules
- GroupMember schema supports id, groupId, studentId, joinedAt.
- Recommend duplicate membership prevention and multiple Groups per Student (GRP-D06).
- Recommend inactive Group cannot be joined (GRP-D02/D08).
- Code expiry, batch, section, enrollment, subscription/seat eligibility, and membership cap are TBD.
- A Group join must not automatically enroll Student in Course or Contest or grant access beyond owning module rules.

### 4.9 Validation
| ID | Condition | Validation point | Behavior |
|---|---|---|---|
| GRP-F007-V01 | Missing/blank code | Client/server | Reject |
| GRP-F007-V02 | Code >20 chars | Client/server | Reject |
| GRP-F007-V03 | Invalid/expired/revoked code | Server | No membership; generic safe feedback |
| GRP-F007-V04 | Group inactive | Server | Reject if approved inactive policy |
| GRP-F007-V05 | Student account inactive/suspended | Server | Deny under Module 1 account policy |
| GRP-F007-V06 | Foreign College/Department | Server | Deny unless cross-scope membership explicitly approved |
| GRP-F007-V07 | Already member | Server | No duplicate; response behavior TBD |
| GRP-F007-V08 | Client-supplied studentId | Server | Ignore/reject; auth identity is authoritative |

### 4.10 Backend/system
Resolve code and Group; re-check authorization at time of join; use atomic membership create and uniqueness/idempotency strategy. Do not trust code validation cached on client. No external service is source-confirmed.

### 4.11 Data/state
Create GroupMember(id, groupId, studentId, joinedAt). Does not update Group or create CourseEnrollment, ContestParticipant, LabSubmission, AssessmentAttempt, or scores.

### 4.12 Success
Membership is created once and Group appears in Student My Groups, subject to approved visibility.

### 4.13 Alternate/failure
- Invalid/stale code: no record.
- Group deactivated during request: final authorization check prevents join if inactive policy applies.
- Duplicate concurrent join: only one membership; duplicate result follows approved idempotency.
- Group scope changes: re-evaluate.
- Membership persistence fails: no success message/phantom membership.
- Client retry after timeout: avoid duplicate membership.

### 4.14 Edge cases
Code rotated between submission and commit; code collision; case/whitespace; Student changes College/Department; Group deactivated; concurrent join requests; Student account status changes; Student already belongs to Group.

### 4.15 Audit/privacy
GROUP_MEMBER_JOINED recommended. Do not log raw inviteCode. Record actor/Group/scope only according to Module 15. Generic errors should avoid Group existence leakage.

### 4.16 Acceptance criteria
- Valid code and eligible Student produce one membership.
- Invalid code produces no membership.
- Client cannot impersonate another Student.
- Duplicate join does not create duplicate GroupMember.
- Code-based join does not grant unrelated Course/Contest/Lab/Assessment access.
- Scope/eligibility restrictions are applied server-side.

### 4.17 QA
See GRP-QA-027–031 and GRP-QA-086–094.

### 4.18 Decisions
GRP-D04, D06, D08; eligibility, cross-department, code expiry, duplicate response, rate limit.

---

## GRP-F008 — Manage Group membership / leave Group

**Requirement status:** GroupMember entity and page mappings source-confirmed; add/remove/leave semantics are Recommendation/TBD.  
**Specification status:** Detailed draft; lifecycle/schema decisions open.  
**Primary actors:** SUPER_ADMIN, HOD, conditional Manager; Student self-leave only if approved.  
**Supporting actors:** User/College/Department/Section authorization; Module 15 audit.

### 4.1 Purpose
Allow authorized staff to manage membership and optionally allow a Student to leave their own Group while preserving historical academic records.

### 4.2 Permissions/scope
- SUPER_ADMIN global; HOD College+Department.
- Manager only explicit permitted Group; no inferred access.
- Finance denied.
- Student cannot add/remove another Student; self-leave is proposed and requires decision GRP-D07.
- Staff can only add/remove eligible Students within Group scope.

### 4.3 Preconditions
Group exists; actor can manage membership; target Student exists and is eligible; Group is in permitted scope. Active Group requirement for add is proposed.

### 4.4 Entry point
Staff Group detail → Members → Add/Remove; Student My Groups → Leave (proposed).

### 4.5 UI behavior
**Recommendation:** staff sees scoped Student selector and current members; Student self-leave action only for own membership. Confirmation on remove/leave. Member field visibility, list filtering, and warnings about ongoing activities are TBD.

### 4.6 Inputs

| Input | Source/type | Required | Constraints |
|---|---|---|---|
| groupId | ID | Yes | In scope |
| studentId | ID | Staff add/remove only | Must be in authorized Student scope |
| action | Add/Remove/Leave | Yes | Supported action only |
| actor identity | Auth context | System | Cannot be client overridden |
| reason | Optional | TBD | May be required for staff removal if policy approved |

### 4.7 Main flow
**Add:** staff opens member list → selects Student → backend authorizes Group and Student scope → checks eligibility/duplicate → creates GroupMember → refreshes member list.  
**Remove:** staff selects member → confirms → backend reauthorizes → applies approved removal semantics → records audit/history as required → refreshes list.  
**Leave:** Student selects own Group → confirms → backend derives Student identity → applies approved membership-end model → updates My Groups.

### 4.8 Business rules
- GroupMember schema only has id, groupId, studentId, joinedAt.
- No status/leftAt/removedAt/role is source-confirmed.
- Hard delete vs soft end/history is unresolved.
- Membership removal must not delete Student-owned submissions, scores, Contest participation, Lab submissions, Assessment attempts/results.
- Student may belong to multiple Groups is recommendation, not confirmed.
- Ongoing contest/assessment access after leave/removal is owned by Modules 9/11 and decision GRP-D07.

### 4.9 Validation
| ID | Condition | Behavior |
|---|---|---|
| GRP-F008-V01 | Actor lacks member management permission | Deny |
| GRP-F008-V02 | Group out of scope | Deny |
| GRP-F008-V03 | Student outside staff scope | Deny |
| GRP-F008-V04 | Duplicate add | Prevent duplicate; result UX TBD |
| GRP-F008-V05 | Remove nonmember | No mutation; feedback TBD |
| GRP-F008-V06 | Student requests other Student removal | Deny |
| GRP-F008-V07 | Student leaves foreign Group | Deny |
| GRP-F008-V08 | Group inactive | Add policy TBD; recommend no new membership |

### 4.10 Backend/system
Authorize both Group and Student; apply membership operation using approved data model. If history is required, schema change or retained audit is needed. Use atomic behavior to avoid duplicate membership. Preserve related student records.

### 4.11 Data/state
- Add: create GroupMember.
- Remove/leave: either delete GroupMember or update approved membership state; source schema lacks state fields.
- Existing submissions/results/participation remain separate and must not be deleted by membership operation.
- Module 17 owns canonical state transitions.

### 4.12 Success
Member list/My Groups reflects the approved membership state. Historical academic data remains accessible under owning-module policy.

### 4.13 Alternate/failure
Unauthorized Group/Student; duplicate add; member not found; concurrent remove/leave; Group deactivated; Student changed scope; DB failure; member has active attempt/contest participation. Exact active-work behavior TBD.

### 4.14 Edge cases
Remove then re-add; duplicate concurrent add; self-leave while activity is active; Student graduation/deactivation; historical access after removal; multiple Groups; group membership record needed for audit.

### 4.15 Audit/privacy
GROUP_MEMBER_ADDED, GROUP_MEMBER_REMOVED, GROUP_MEMBER_LEFT recommended. Include actor, Group, target Student, action, scope; minimize PII. Module 15 decides canonical event fields/retention.

### 4.16 Acceptance criteria
- Authorized staff can add/remove only within scope.
- Student cannot manage another Student’s membership.
- Self-leave uses authenticated Student identity.
- Membership operation does not erase academic history.
- Duplicate membership is prevented.
- Active attempt/contest consequences are not silently guessed.

### 4.17 QA
See GRP-QA-032–037 and GRP-QA-095–105.

### 4.18 Decisions
GRP-D06, D07, D17; membership history, leave behavior, add/remove eligibility, active work.

---

## GRP-F009 — Associate Group with Labs and Contests

**Requirement status:** GroupLab/GroupContest schema and Group-mediated collaborative flow source-confirmed; operation details Recommendation/TBD.  
**Specification status:** Detailed draft; cross-module decisions open.  
**Primary actors:** SUPER_ADMIN, HOD, conditional Manager.  
**Supporting actors:** Modules 8 (Labs), 11 (Contests), 2 (scope).

### 4.1 Purpose
Associate an existing Lab Assignment or Contest with an authorized Group. This function links records; it does not create the Lab/Contest or group submission/score.

### 4.2 Permissions/scope
- SUPER_ADMIN global.
- HOD College+Department.
- Manager only if both Group and activity are within explicit permitted scope.
- Finance/Student deny association management.
- Actor must be allowed to manage both Group and activity by owning module.

### 4.3 Preconditions
Group exists; activity exists; actor can manage Group and activity; compatibility checks are approved. Active Group requirement is a recommendation.

### 4.4 Entry point
Group detail → Activities / Assign Lab or Contest; or an owning-module Group assignment workflow (exact UI TBD).

### 4.5 UI behavior
**Recommendation:** scoped activity selector with linked/unlinked state; assignment and removal actions. The Lab/Contest must be selected from permitted records only. Start/end lifecycle warnings and unassignment controls require owning-module approval.

### 4.6 Inputs

| Input | Type | Required | Constraints |
|---|---|---|---|
| groupId | ID | Yes | Authorized Group |
| labAssignmentId | ID | For Lab link | Authorized LabAssignment |
| contestId | ID | For Contest link | Authorized Contest |
| action | Assign/Unassign | Yes | Only approved transitions |

### 4.7 Main flow
1. Actor opens assignment UI.
2. Backend returns only eligible Groups/activities.
3. Actor selects Group and Lab or Contest.
4. Backend reauthorizes actor against both records.
5. Backend checks College/Department compatibility and activity state per approved rules.
6. Backend creates GroupLab or GroupContest association.
7. UI confirms link and refreshes activity list.

### 4.8 Business rules
- GroupLab fields: groupId, labAssignmentId, createdAt.
- GroupContest fields: groupId, contestId, createdAt.
- Business Flow says Groups mediate collaborative Labs and Contests.
- Recommend no duplicate association.
- Recommend scope compatibility.
- Association does not create LabSubmission, ContestParticipant, CodeSubmission, score, rank, or certificate.
- Assignment/unassignment after activity begins/ends is TBD.
- Whether Group association changes participation eligibility is governed by Modules 8/11 and remains unresolved.

### 4.9 Validation
| ID | Condition | Behavior |
|---|---|---|
| GRP-F009-V01 | Actor lacks permission for Group | Deny |
| GRP-F009-V02 | Actor lacks permission for activity | Deny |
| GRP-F009-V03 | Foreign College/Department activity | Deny |
| GRP-F009-V04 | Duplicate link | Prevent duplicate; UX TBD |
| GRP-F009-V05 | Group inactive | Block if approved inactive policy |
| GRP-F009-V06 | Activity started/ended/archived | Apply owning-module lifecycle decision; TBD |
| GRP-F009-V07 | Unassign after records exist | Block/warn/allow according to approved policy; TBD |

### 4.10 Backend/system
Validate both records and scope; create/delete association record only. Use transaction/unique constraints if approved. Do not alter owning activity configuration or generate participation/submissions/scores.

### 4.11 Data/state
Create/remove GroupLab(groupId, labAssignmentId, createdAt) or GroupContest(groupId, contestId, createdAt). No other entity changes.

### 4.12 Success
Association appears in Group and owning activity context as allowed.

### 4.13 Alternate/failure
Foreign activity; activity not found; duplicate association; activity starts during request; Group deactivated; partial failure; remove association while Student has existing submissions/participation. No data should be partially linked.

### 4.14 Edge cases
Contest already live/ended; Lab due date passed; archived activity; Group member removed; multiple Groups associated with same activity; concurrent assign/unassign.

### 4.15 Audit/privacy
GROUP_LAB_ASSIGNED/UNASSIGNED and GROUP_CONTEST_ASSIGNED/UNASSIGNED recommended. Module 15 finalizes. Do not include code/submission source in Group audit.

### 4.16 Acceptance criteria
- Authorized compatible Group/activity association succeeds.
- Foreign-scope associations are denied.
- Duplicate associations are prevented.
- Linking Group does not create individual participation/submission/score/rank.
- Activity start/end unassignment behavior follows approved owner-module rules.

### 4.17 QA
See GRP-QA-038–041 and GRP-QA-106–116.

### 4.18 Decisions
GRP-D03, D09; active/ended activity assignment, unassignment, member eligibility semantics.

---

## GRP-F010 — Associate Course Problems with Group

**Requirement status:** GroupCourseProblem schema source-confirmed; page mapping v2 includes it in Courses & Labs; assignment workflow Recommendation/TBD.  
**Specification status:** Detailed draft; decision required.  
**Primary actors:** SUPER_ADMIN, HOD, conditional Manager.

### 4.1 Purpose
Create a Group-to-Course/Problem association without replacing CourseProblem or CourseEnrollment semantics.

### 4.2 Permissions/scope
Actor must manage Group, Course, and Problem under the relevant College/Department scope. Manager requires explicit scope. Students cannot create/edit association.

### 4.3 Preconditions
Group, Course, and Problem exist; actor can manage each; Course/Problem compatibility rule approved.

### 4.4 Entry point
Courses & Labs page (mapping v2) or Group detail → Course Problems (recommended).

### 4.5 UI behavior
**Recommendation:** show permitted Course and Problem selectors, current Group-associated Problems, and add/remove action. Exact UI, filters, and relation semantics are TBD.

### 4.6 Inputs

| Input | Type | Required | Constraints |
|---|---|---|---|
| groupId | ID | Yes | In scope |
| courseId | ID | Yes | In scope |
| problemId | ID | Yes | In scope and course-compatible |
| action | Associate/Unassociate | Yes | Approved actions only |

### 4.7 Main flow
1. Actor opens Group/Course association UI.
2. Backend loads eligible Course/Problem options.
3. Actor selects Course and Problem.
4. Backend re-checks role, Group, Course, Problem, and relationship scope.
5. Backend creates GroupCourseProblem.
6. UI confirms and displays the association.

### 4.8 Business rules
- GroupCourseProblem schema: groupId, courseId, problemId, createdAt.
- It must not silently modify CourseProblem or CourseEnrollment.
- Recommended compatibility: Problem is permitted within the selected Course and scope.
- Whether the association grants, narrows, or merely records visibility/assignment is TBD.
- Does not create CodeSubmission, LabSubmission, AssessmentAttempt, or score.

### 4.9 Validation
| ID | Condition | Behavior |
|---|---|---|
| GRP-F010-V01 | Group/Course/Problem out of scope | Deny |
| GRP-F010-V02 | Problem missing or not available to Course | Decision-blocked until compatibility rule approved |
| GRP-F010-V03 | Duplicate association | Prevent duplicate if approved |
| GRP-F010-V04 | Group inactive | Apply approved inactive Group policy |
| GRP-F010-V05 | Client changes tenant/scope IDs | Reject/ignore; backend scope authoritative |

### 4.10 Backend/system
Authorize each record; validate course/problem relationship; create/remove GroupCourseProblem only. Association storage does not imply access grant without an approved rule.

### 4.11 Data/state
Create/remove GroupCourseProblem row. No CourseProblem, CourseEnrollment, GroupMember, or submission mutation.

### 4.12 Success
Association is persisted and displayed to authorized staff; Student visibility follows approved Course/Problem access rules.

### 4.13 Alternate/failure
Out-of-scope Course/Problem; invalid relation; duplicate; Group inactive; association fails; concurrent updates; Course/Problem later archived/removed. Owning modules determine continued visibility.

### 4.14 Edge cases
Problem not yet attached to Course; Course scoped to multiple Sections; Student not enrolled; Group membership changes; multiple Groups associated with same Problem.

### 4.15 Audit/privacy
Group Course Problem assignment/unassignment audit events recommended; Module 15 defines event catalog. Do not expose hidden Problem/testcase content.

### 4.16 Acceptance criteria
- Valid association is created only after all three records are authorized.
- Foreign Course/Problem cannot be linked.
- Association does not silently modify CourseProblem or enrollment.
- Student access remains controlled by owning Course/Problem modules.
- No submissions or scores are created by linking.

### 4.17 QA
See GRP-QA-042–045 and GRP-QA-117–125.

### 4.18 Decisions
GRP-D10; relation semantics, compatibility rule, duplicate uniqueness, unassignment behavior.

---

## GRP-F011 — View Group-linked activities and Assessment reference

**Requirement status:** Schema/page mappings source-confirmed; Group-level activity display and Assessment semantics derived/TBD.  
**Specification status:** Detailed draft; cross-module decisions open.  
**Primary actors:** SUPER_ADMIN, HOD, Manager in scope, Student member/permitted.  
**Supporting actors:** Modules 4, 8, 9, 11, 17.

### 4.1 Purpose
Show authorized Group-linked Lab, Contest, Course Problem, and Assessment references in context without granting access beyond the owning module.

### 4.2 Permissions/scope
- Actor must have Group view permission.
- Student must be a current member or otherwise permitted under approved rule.
- Each activity is independently authorized.
- Staff view is limited by Group and owning activity scope.
- Group membership is not sufficient to access another Student’s submissions, attempts, results, or hidden testcases.

### 4.3 Preconditions
Group exists; actor can view it; association records/Assessment.groupId exist; owning modules provide authorized data.

### 4.4 Entry point
Group detail → Activities; Student My Groups → Group detail; owning module may link back to Group.

### 4.5 UI behavior
**Recommendation:** separate activity sections (Labs, Contests, Course Problems, Assessments) only where the owning module exposes the relevant details. Show minimal metadata and link to authorized owning page. Empty state if no association. Assessment UI is not dedicated in Group page mapping.

### 4.6 Inputs
- groupId required.
- activity type/filter optional; supported values TBD.
- studentId must be auth-derived, not client-supplied.
- Assessment reference via Assessment.groupId; no special Group assessment behavior is implied.

### 4.7 Main flow
1. Actor opens Group activity view.
2. Backend authorizes Group.
3. Backend loads association records.
4. Backend asks/uses owning-module authorization for Lab/Contest/Course Problem/Assessment.
5. Backend returns permitted metadata and links only.
6. UI renders sections and empty/loading/error states.
7. Opening activity navigates to owning module, which performs fresh authorization.

### 4.8 Business rules
- GroupLab/GroupContest/GroupCourseProblem and Assessment.groupId exist in schema.
- Group membership does not imply Group attempt, shared submission, Group score, or Group rank.
- Assessment.groupId is only a relationship until Module 9 approves semantics.
- Individual Student submissions, attempts, scores, and results remain protected by owning module and ownership rules.
- Archived/ended activities follow owning-module rules.
- Membership freeze/timing (e.g., after contest begins) is TBD.

### 4.9 Validation
| ID | Condition | Behavior |
|---|---|---|
| GRP-F011-V01 | Group outside scope | Deny |
| GRP-F011-V02 | Linked activity outside scope | Omit/deny according to owner module |
| GRP-F011-V03 | Student requests another Student’s record | Deny |
| GRP-F011-V04 | Assessment has groupId | Display only according to Module 9 assignment/visibility rules |
| GRP-F011-V05 | Client forges activity/student IDs | Re-authorize; auth identity authoritative |

### 4.10 Backend/system
Read association data and permitted activity metadata; use owning module for access policy; no writes. Do not expose hidden test cases, code, private submission details, or score fields unless separately authorized.

### 4.11 Data/state
Read Group, GroupMember, GroupLab, GroupContest, GroupCourseProblem, and Assessment.groupId relationship. No state changes.

### 4.12 Success
Authorized linked activities are displayed and navigable. No unauthorized details are returned.

### 4.13 Alternate/failure
No activities; activity archived; Group inactive; membership revoked; owning module unavailable; Assessment access denied; student activity eligibility differs from Group membership. Fail/partial-display behavior TBD under Module 18.

### 4.14 Edge cases
Student joins after activity starts; member removed while an attempt is active; activity linked to several Groups; Group inactive while assessment still scheduled; former Student has historical results.

### 4.15 Audit/privacy
Read audit policy Module 15. No notification. Hide source code, hidden testcase values, other Student private attempts/results.

### 4.16 Acceptance criteria
- Only Group and activity metadata authorized for actor are displayed.
- Group access does not bypass owning module permissions.
- Assessment.groupId does not imply group scoring or shared attempts.
- Student cannot access another Student’s private records.
- Archived/active activity behavior follows owning module.

### 4.17 QA
See GRP-QA-046–049 and GRP-QA-126–132.

### 4.18 Decisions
GRP-D07, D09–D11; visibility, member timing, Assessment semantics, partial-load behavior.

---

## 5. Shared data and state reference

### 5.1 Group (source-confirmed conceptual schema)

| Field | Type | Meaning | Editability/status |
|---|---|---|---|
| id | varchar | Group identifier | System-managed; generation TBD |
| collegeId | varchar | College association | Scope-controlled |
| departmentId | varchar | Department association | Scope-controlled; transfer workflow TBD |
| createdById | varchar | Creator User ID | Auth-derived |
| name | varchar(255) | Group name | Proposed editable |
| description | text | Description | Proposed editable; limits TBD |
| inviteCode | varchar(20) | Invite code | Dedicated F006; security lifecycle TBD |
| isActive | boolean | Active flag | F005; transition policy TBD |
| createdAt | timestamp | Created time | System-managed |
| updatedAt | timestamp | Updated time | System-managed |

### 5.2 Related entities (source-confirmed schema)

| Entity | Fields | Module relevance |
|---|---|---|
| GroupMember | id, groupId, studentId, joinedAt | membership |
| GroupLab | groupId, labAssignmentId, createdAt | Group–Lab relation |
| GroupContest | groupId, contestId, createdAt | Group–Contest relation |
| GroupCourseProblem | groupId, courseId, problemId, createdAt | Group–Course–Problem relation |
| Assessment | includes groupId | Group association; semantics owned by Module 9 |

### 5.3 Schema gaps (do not silently fill)
- Group has no sectionId.
- No Group-to-Manager assignment relation is present.
- GroupMember has no status, leftAt, removedAt, or member role.
- No invite code expiry, revocation, usage limit, or generated-at field.
- No capacity field.
- No Group-specific submission, score, or ranking entity.
- No source-defined uniqueness constraint for Group name, inviteCode, or GroupMember pair.
- No explicit Group deletion field/state.
- Conceptual schema does not prove database constraints or deployed behavior.

## 6. Cross-module dependencies

| Module | Dependency / ownership |
|---|---|
| 1 Authentication & User Lifecycle | authenticated identity, account status, role |
| 2 College / Department / Section | College/Department/Section scope and lifecycle; Manager mapping gap |
| 3 Subscription & Seat Management | Student/college eligibility only if product policy requires it; Group source does not specify |
| 4 Course & Enrollment | Course/Enrollment/CourseProblem eligibility |
| 8 Labs & Lab Scoring | LabAssignment, Lab submissions, lab lifecycle |
| 9 Assessments | Assessment.groupId, eligibility, attempts, timing, results |
| 11 Contests (submodules) | Contest assignment, participation, scoring, timing, ranking |
| 13 Notifications | Group event/recipient policy |
| 15 Audit Logging | canonical audit events and retention |
| 17 Functional State Machines | Group and membership lifecycle |
| 18 Edge Cases & Failure Handling | shared error, retry, idempotency, concurrency behavior |

### 6.1 Cross-module reconciliation notes

- **Module 2:** its organization specification explicitly delegates Group-to-Section/Manager scope to Module 10. The current Group schema has no sectionId or GroupManager relation; GRP-D03/D16 remain blocking recommendations, not inferred permissions.
- **Module 4:** GroupCourseProblem appears in Course/Labs page mappings. Group links must not bypass CourseProblem, CourseSection, CourseEnrollment, student eligibility, or existing content-scope rules.
- **Module 8:** its target baseline states that Module 10 owns GroupLab association writes. Group assignment is distinct from submission/score ownership; the Lab module retains its Student-level submission/scoring contract.
- **Module 9:** Assessment.groupId exists in the current conceptual schema and Module 9 owns assessment eligibility/attempt/result behavior. Because Module 9 decisions were prematurely marked adopted before the whole-system review, its Group membership effects remain subject to the later cross-module decision pass. This Module 10 draft does not adopt assessment attempt/score semantics.
- **Module 11:** Module 10 owns GroupContest association writes; Contest participation, scoring, penalties, ranking, and tie-breaking remain with the Contest module.
- **Modules 13/15/17/18:** notifications, canonical audit events/retention, lifecycle transitions, and shared failure/retry/concurrency policy remain with their owning modules.
## 7. Consolidated decisions## 7. Consolidated decisions

Use the Module 10 README decision log as the canonical list: GRP-D01–D17. Function-specific references above do not create additional decisions unless added to that log.

## 8. Readiness checklist

- [x] Function inventory includes 11 target functions derived from business-flow, page/entity mapping, and role-scope evidence; lifecycle/API details remain Recommendation/TBD where source documents are silent.
- [x] Function-level detail is expanded according to Standard v1.0 template.
- [x] Source facts, recommendations, and decision gaps are separated.
- [x] Permissions and organizational scope are described.
- [x] UI, inputs, validation, flows, backend/data changes, alternate paths, privacy, acceptance criteria, and QA links are present.
- [x] Related schema entities and gaps are documented.
- [x] Cross-module dependencies are listed.
- [ ] Product/security/engineering decisions GRP-D01–D17 resolved.
- [ ] Target behavior approved; implementation readiness not claimed.

**Review status:** Detailed draft ready for later product review after the remaining modules are authored. Do not resolve the open decisions during the current all-modules-first drafting pass.
