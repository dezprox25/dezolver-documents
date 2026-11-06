# 2. College / Department / Section Management

**Document ID:** DEZ-ORG-FS-002  
**Document type:** Functional Specification  
**Version:** 0.2 — Functional Specification Standard v1.0 compliance pass  
**Status:** Detailed draft — ready for product review; target behavior not approved  
**Repository target:** `FUNCTIONAL-SPECIFICATION/2. College - Department - Section Management/README.md`  
**Review basis:** Functional Specification Standard v1.0; Business Flow (original and v2); Page-to-DB Mapping (original and v2); RBAC; schema  
**QA companion:** [ORG QA Test Cases](./QA-TEST-CASES.md)

> **Evidence boundary:** This is a source-grounded target functional draft, not proof of legacy implementation. Schema and page mappings confirm represented data/pages, not implemented workflows. Source-confirmed permissions are preserved; inferred Section permissions and lifecycle choices remain TBD. Recommendations are not approved requirements.

---

## 1. Purpose and module outcome

Define the organizational structure used by DEZOLVER:

`College → Department → Section`

The module provides the organization records and parent-child relationships needed to scope Finance Team, HOD, Manager, and Student activity. At completion, the organization hierarchy must be represented consistently, and protected organization operations must be restricted to the actor's authorized role and scope.

The module does **not** define user credential/invitation behavior, subscription billing/seat arithmetic, academic enrollment, or the platform-wide audit/state-machine implementations owned by other modules.

## 2. Scope

### 2.1 In scope
- College records and College Management / College Details context.
- College onboarding record context and the business-flow transition to College Created.
- Department records under a College.
- Section records under a Department.
- Department HOD reference and HOD-to-department consistency.
- Organizational references used by User: `collegeId`, `departmentId`, `sectionId`.
- Parent-child integrity, tenant isolation, hierarchy lookups, and scope filtering.
- Organization status field and status-related decisions.
- College/Department seat fields as data references only; seat accounting belongs to Module 3.
- Organization-related audit and cross-module dependencies.

### 2.2 Out of scope / delegated ownership

| Concern | Owner / boundary |
|---|---|
| User create/edit/delete, role changes, credentials, invitations, account activation | Module 1. This module defines organization-reference validity only. |
| Subscription, billing, purchased seats, seat accounting, allocation, quota enforcement | Module 3. This module does not define seat calculations. |
| Course assignment to Section, enrollment, academicBatch eligibility | Module 4. |
| Group and Group-to-Section/Manager scope details | Module 10. |
| Notifications | Module 13. |
| Audit event catalog, retention, audit UI | Module 15. |
| Canonical lifecycle state machines | Module 17. |
| Common failure/retry/consistency patterns | Module 18. |

## 3. Source documents and evidence

### 3.1 Source set
- `docs/source-text/Business Flow.md`
- `docs/source-text/Business Flow v2.md`
- `docs/source-text/Page - DB Mapping.md`
- `docs/source-text/Page - DB Mapping v2.md`
- `docs/source-text/RBAC.md`
- `docs/source-text/scheema.md`
- `FUNCTIONAL-SPECIFICATION/FUNCTIONAL-SPECIFICATION-STANDARD.md`

### 3.2 Source interpretation

| Source | Source-supported content | Limit |
|---|---|---|
| Business Flow original and v2 | SUPER_ADMIN starts college onboarding; flow depicts College Onboarding → College Created → Setup → Finance Team → Subscription. HOD handles department-level academic structure; Manager operates within assigned Section. | Diagrams are high-level; they do not define exact UI, approvals, lifecycle transitions, invitations, or validation messages. |
| RBAC | SUPER_ADMIN global; Finance Team college-scoped; HOD college+department-scoped; Manager college+department+section/permitted group; Student self+permitted academic data. College Management: SUPER_ADMIN Full CRUD, Finance Team View, HOD View in matrix. Department Management: SUPER_ADMIN Full CRUD, Finance Team View, HOD View/Update. | Dedicated Section Management row and detailed field-level permissions are not explicitly specified. |
| Page-to-DB Mapping original and v2 | SUPER_ADMIN College Management and College Details pages map to College/User/Department/Section and related entities; HOD Department Management maps to College/Department/Section. | Page/entity mapping is not a workflow or UI field specification. |
| Schema | College, Department, Section, User, CollegeOnboarding entities and fields; Department has `collegeId` and `hodId`; Section has `departmentId`; User has `collegeId`, `departmentId`, `sectionId`. | Schema does not establish requiredness, uniqueness, cascade behavior, field editability, workflow states, or implementation. |

### 3.3 Evidence labels used
- **Source-confirmed:** explicit source fact.
- **Legacy-verified:** verified directly in implementation/forensic evidence. No legacy implementation verification is included in this document.
- **Target requirement:** approved target behavior. None of the newly authored recommendations in this draft are product-approved.
- **Derived:** logical safeguard or relationship conclusion based on source facts.
- **Recommendation:** suggested behavior requiring approval.
- **TBD / Decision Required:** absent, ambiguous, or conflicting source information.

## 4. Terminology

| Term | Meaning in this specification | Evidence |
|---|---|---|
| College | Tenant-level organization record | Source-confirmed schema/RBAC |
| Department | Academic organizational unit belonging to one College | Source-confirmed schema |
| Section | Academic/operational unit belonging to one Department | Source-confirmed schema |
| HOD | Role responsible for assigned College+Department scope | Source-confirmed RBAC |
| Manager | Role operating within assigned College+Department+Section / permitted Group scope | Source-confirmed RBAC |
| Parent chain | College → Department → Section relationship | Source-confirmed / derived |
| Organization assignment | User references to College, Department, and Section | Source-confirmed schema; workflow delegated to Module 1 |

## 5. Actors, roles, permissions, and scope

### 5.1 Source-based permission matrix

| Organization action | SUPER_ADMIN | FINANCE_TEAM | HOD | MANAGER | STUDENT | Status |
|---|---|---|---|---|---|---|
| View College | Allow globally | Allow assigned College | View appears in RBAC College Management matrix; scope/route details TBD | No management permission shown | No management permission shown | Source-confirmed except HOD page detail |
| Create / update / delete College | Full CRUD | Deny for write | Deny for write | Deny | Deny | Source-confirmed RBAC |
| View Department | Allow globally | Allow within assigned College | Allow assigned Department | No management permission shown | No management permission shown | Source-confirmed RBAC |
| Create Department | Allow globally | Deny | Not explicitly allowed | Deny | Deny | Source-confirmed RBAC |
| Update Department | Allow globally | Deny | Allow within assigned Department; fields TBD | Deny | Deny | Source-confirmed permission; field limits TBD |
| Delete Department | Allow globally per Full CRUD matrix | Deny | Not explicitly allowed | Deny | Deny | Source-confirmed role action; deletion semantics TBD |
| View Section | Allow globally | College-view context only if permitted; detailed need TBD | Department-view context; explicit Section permission TBD | Assigned Section context only, not configuration access | No organization-admin access | Section access partly derived/TBD |
| Create / update / delete Section | Allow only if Full CRUD hierarchy permission is confirmed for this page/action | Deny unless explicitly approved | TBD: RBAC lacks dedicated Section row | Deny unless explicitly approved | Deny | TBD |
| Assign/change Department HOD | Allow; source supports Department Full CRUD | Deny | TBD; no explicit HOD assignment action | Deny | Deny | TBD |
| View organization hierarchy | Global | Assigned College only | Assigned College+Department | Assigned College+Department+Section | Own permitted context only; no admin browsing | Derived from scope |

**Important:** “Full CRUD” for College and Department is the explicit RBAC matrix wording. It does not settle physical deletion, cascade behavior, or whether child records block deletion. Do not implement destructive behavior without decisions.

### 5.2 Role scope
- **SUPER_ADMIN:** global/platform-wide.
- **FINANCE_TEAM:** assigned College only.
- **HOD:** assigned College + Department.
- **MANAGER:** assigned College + Department + assigned Section / permitted Group.
- **STUDENT:** own records + permitted academic resources; no organization administration.

### 5.3 Mandatory server-side authorization

For every protected organization request, server-side checks must establish:
1. Authenticated actor and active authorization context (Module 1).
2. Permission for the requested action.
3. College scope.
4. Department scope where relevant.
5. Section scope where relevant.
6. Parent-child consistency and ownership/membership where applicable.

**Source-confirmed RBAC principle:** frontend visibility is not security. A client-supplied College/Department/Section ID must not override server-derived scope. Unauthorized requests must not expose protected records or perform writes.

## 6. Navigation and entry points

| Page / entry point | Role | Source mapping | What can be specified now |
|---|---|---|---|
| College Management | SUPER_ADMIN | Page-to-DB Mapping | Page exists and is associated with College/User/Department/Section/Subscription. Search/filter/layout/actions are TBD. |
| College Details | SUPER_ADMIN | Page-to-DB Mapping | Page exists and maps College/Department/Section/User. Detail tabs/fields/actions TBD. |
| Department Management | HOD | Page-to-DB Mapping | Page exists and maps College/Department/Section. Exact field-level controls TBD. |
| College onboarding flow | SUPER_ADMIN in Business Flow | Business Flow original/v2 | High-level flow exists; intake channel and approval UI TBD. |
| Direct URL/API / system events | All applicable roles | RBAC security principle | Must enforce server authorization and scope; route/API names are not supplied. |

## 7. Functional inventory

Function IDs are stable and module-specific. “Decision required” means the behavior is documented but cannot be treated as approved.

| Function ID | Function | Primary actor | Type | Specification status |
|---|---|---|---|---|
| ORG-F001 | Create / capture College onboarding record | SUPER_ADMIN / submitter TBD | Workflow/Create | Decision required |
| ORG-F002 | View onboarding records and details | SUPER_ADMIN | Read | Decision required |
| ORG-F003 | Review onboarding record | SUPER_ADMIN/reviewer TBD | Workflow/Update | Decision required |
| ORG-F004 | Create College from approved onboarding or admin input | SUPER_ADMIN | Create/workflow | Decision required |
| ORG-F005 | List and view Colleges | SUPER_ADMIN; scoped Finance/HOD view where allowed | Read | Draft |
| ORG-F006 | Update College details | SUPER_ADMIN | Update | Decision required |
| ORG-F007 | Change College status | SUPER_ADMIN | Lifecycle | Decision required |
| ORG-F008 | Create Department under College | SUPER_ADMIN | Create | Draft |
| ORG-F009 | View/list Department and hierarchy | SUPER_ADMIN/Finance/HOD by scope | Read | Draft |
| ORG-F010 | Update Department details | SUPER_ADMIN; HOD limited fields TBD | Update | Decision required |
| ORG-F011 | Delete Department | SUPER_ADMIN | Delete | Decision required |
| ORG-F012 | Create Section under Department | SUPER_ADMIN; HOD permission TBD | Create | Decision required |
| ORG-F013 | View/list Sections | SUPER_ADMIN/HOD/Finance/Manager by scope | Read | Draft |
| ORG-F014 | Update Section details | SUPER_ADMIN; HOD permission TBD | Update | Decision required |
| ORG-F015 | Delete Section | SUPER_ADMIN; HOD permission TBD | Delete | Decision required |
| ORG-F016 | Assign/change Department HOD | SUPER_ADMIN; other roles TBD | Assignment | Decision required |
| ORG-F017 | Validate user organization placement | Module 1 user provisioning/update with Module 2 constraints | Validation | Derived / decision required |
| ORG-F018 | Resolve/filter organization hierarchy by actor scope | System / authorized actor | Read/security | Derived |
| ORG-F019 | Validate parent-child relationships and prevent cross-scope writes | System | Validation/security | Derived |
| ORG-F020 | Handle organization status and hierarchy impact | SUPER_ADMIN / system | Lifecycle/cross-module | Decision required |

## 8. Detailed function specifications

### ORG-F001 — Create / capture College onboarding record

**Requirement status:** Source-confirmed business-flow concept; intake workflow TBD  
**Specification status:** Decision required  
**Priority:** TBD  
**Primary actor:** SUPER_ADMIN is shown initiating onboarding; whether an external College contact can submit is TBD.  
**Supporting actor:** System; reviewer identity is represented by schema field `reviewedById`.

#### 8.1 Purpose and outcome
Capture prospective College details in a `CollegeOnboarding` record before the high-level flow reaches College Created. The source Business Flow depicts onboarding but does not define the intake actor, exact form, or workflow.

#### 8.2 Permissions and scope
- SUPER_ADMIN: source shows onboarding initiation; create permission for the onboarding record is derived from this flow.
- External submitter: TBD.
- Finance Team/HOD/Manager/Student: no source-confirmed onboarding-create permission.

#### 8.3 Preconditions
- Actor is authenticated/authorized per Module 1 if submission is restricted to platform users.
- Required onboarding fields and duplicate handling must be approved before implementation (ORG-D01/D02).

#### 8.4 Entry point and UI behavior
- Entry point: College Onboarding step in the Business Flow.
- Page/form layout, whether onboarding is public or internal, save-draft behavior, and confirmation text are TBD.
- The source does not specify a status queue, wizard, or step count; do not assume one.

#### 8.5 Inputs

| Field | Schema type | Required? | Source/validation |
|---|---|---|---|
| `collegeName` | varchar(255) | TBD | Field exists; non-empty rule and normalization TBD |
| `contactPerson` | varchar(255) | TBD | Field exists; requiredness TBD |
| `contactEmail` | varchar(255) | TBD | Field exists; format/uniqueness TBD |
| `phone` | varchar(30) | TBD | Field exists; format/country code TBD |
| `address` | text | TBD | Field exists; requiredness/format TBD |
| `studentCount` | int | TBD | Field exists; non-negative and allowed range TBD |
| `departmentDetails` | JSON | TBD | Field exists; structure/validation TBD |
| `paymentStatus` | varchar(30) | System/flow TBD | Allowed values and whether it is set during intake TBD |

System fields: `id`, `createdAt`, `updatedAt`. Exact generation and update rules are schema-level assumptions/TBD.

#### 8.6 Main flow (source-supported outline)
1. The initiating actor opens the College Onboarding flow.
2. Actor supplies the onboarding data required by the approved form (exact field requirements TBD).
3. System validates the approved fields and persists a `CollegeOnboarding` record.
4. System returns a success result and onboarding reference (UI/result format TBD).
5. Record becomes available for review if a review process is approved (ORG-F002/F003).

#### 8.7 Business rules and validation
- ONB-R01: `CollegeOnboarding` is a separate schema entity from `College` (source-confirmed).
- ONB-R02: The Business Flow links onboarding conceptually to College Created; conversion/linking mechanism is TBD.
- ONB-R03: Duplicate college-name/domain/contact detection is not defined; do not enforce a uniqueness rule until approved.
- ONB-R04: `departmentDetails` JSON shape is unspecified; do not invent nested field contracts.

#### 8.8 System/data behavior
- Create a `CollegeOnboarding` record with submitted fields and timestamps (entity fields source-confirmed; write behavior derived from onboarding flow).
- Schema has no explicit `collegeId` field on CollegeOnboarding in the supplied diagram. Linking to a College requires decision/schema review.
- No automatic College creation, subscription activation, or Finance Team provisioning is confirmed by this function.

#### 8.9 Success, alternate, and failure paths
- Success: onboarding record is saved; actor receives confirmation/reference (confirmation UX TBD).
- Invalid field: reject the invalid submission; exact client/server validation and message TBD.
- Duplicate candidate: handling TBD; do not silently merge or reject without a decision.
- Save failure: preserve user input where possible is a recommendation, not approved; retry/idempotency TBD.
- Unauthorized actor: deny; no onboarding data should be disclosed/created.

#### 8.10 Audit, notification, privacy
- Onboarding may contain contact information and address; access, retention, privacy notice, and audit requirements need approval.
- Audit event/notification details belong to Modules 13/15 and are not defined here.

#### 8.11 Acceptance criteria
- AC-ORG-F001-1: Authorized onboarding submission creates one onboarding record containing the supplied accepted fields.
- AC-ORG-F001-2: Unauthorized submission does not create a record.
- AC-ORG-F001-3: System does not create a College or activate subscription unless an approved conversion/activation rule is defined.
- AC-ORG-F001-4: Required fields and duplicate behavior remain configurable/decision-bound until ORG-D01/D02 are approved.

#### 8.12 QA
See ORG-QA-001–006 in the QA companion. Required form validation and final actor policy are blocked by decisions.

#### 8.13 Unresolved items
ORG-D01, ORG-D02, ORG-D03, ORG-D04, ORG-D05.

---

### ORG-F002 — View onboarding records and details

**Requirement status:** Derived from onboarding/review business flow and schema  
**Specification status:** Decision required  
**Primary actor:** SUPER_ADMIN/reviewer (reviewer identity field exists; list page is not mapped explicitly).

- Permission: SUPER_ADMIN-only is a recommendation; confirm reviewer roles and whether Finance Team can view.
- Entry point: onboarding review context; page/route not shown in Page-to-DB Mapping.
- Data displayed: CollegeOnboarding fields; whether contact details, paymentStatus, or departmentDetails are visible to each role is TBD.
- Scope: onboarding records are platform-wide if SUPER_ADMIN-only. Any delegated reviewer must have a defined scope.
- Flow: actor requests list/detail → server checks reviewer permission → retrieves only permitted onboarding records → UI displays data. Search/filter/sort/pagination are TBD.
- Data changes: none.
- Failure: unauthorized access denied; nonexistent ID returns not-found without revealing unrelated records; loading/error UX TBD.
- Acceptance: only approved reviewers can retrieve onboarding; returned record is limited to permitted data; no record mutation occurs from read.
- QA: ORG-QA-007–009.
- Decisions: ORG-D03/D04/D05/D06.

### ORG-F003 — Review onboarding record

**Requirement status:** Schema-confirmed reviewer fields; workflow decision required  
**Primary actor:** reviewer role TBD (SUPER_ADMIN shown in business flow is a possible actor, not a confirmed review permission).

- Schema: `reviewedById`, `reviewedAt` exist on CollegeOnboarding.
- Review outcomes (approve/reject/request changes), status field, comments, and transition rules are not defined.
- Entry point, actions, confirmation dialog, required review notes, and notification behavior are TBD.
- Approved future flow must validate that record exists and is eligible for review; update reviewer reference/time only when a review action is approved.
- Do not infer `paymentStatus` to mean onboarding approval status; field semantics are not documented.
- Acceptance: review actor and timestamp are recorded only for an approved review event; unauthorized actor cannot review; review does not create a College unless conversion is explicitly part of the approved action.
- QA: ORG-QA-010–013.
- Decisions: ORG-D03–D07.

### ORG-F004 — Create College from onboarding or administrative input

**Requirement status:** College creation source-confirmed; conversion linkage/trigger TBD  
**Primary actor:** SUPER_ADMIN  
**Permissions:** SUPER_ADMIN Full CRUD in College Management.

#### Inputs and schema fields
College:
- `name` varchar(255)
- `domain` varchar(255)
- `status` EntityStatus
- `totalSeats` int
- `usedSeats` int
- `id`, `createdAt`, `updatedAt`

Requiredness, default status, uniqueness, seat-counter initial values, and source mapping from onboarding are TBD.

#### Main flow
1. SUPER_ADMIN opens College Management or an approved onboarding conversion action.
2. System displays College creation fields approved by product.
3. Actor submits.
4. Server validates SUPER_ADMIN global scope and approved College constraints.
5. Server creates a College record.
6. Server returns the new College record and navigates/updates the management UI (exact UX TBD).

#### Rules / data
- College is the parent of Department.
- If creation originates from onboarding, the approved linkage must prevent duplicate conversion; source schema currently has no explicit onboarding-to-college foreign key.
- Seat counters must not be initialized/updated using assumptions from this module; Module 3 owns seat policy.
- Physical/soft deletion is not defined by creation behavior.

#### Failure/alternate
- Invalid values → no College created.
- Duplicate candidate → uniqueness behavior TBD.
- Unauthorized role → deny.
- Conversion retried → duplicate-prevention/idempotency TBD.
- Concurrent creation with same candidate → conflict behavior TBD.

#### Acceptance
- SUPER_ADMIN can create a College when required inputs and policies are approved.
- Non-authorized roles cannot create.
- Parentless College is allowed by schema (top-level entity).
- If onboarding conversion is implemented, a conversion must have an approved link/duplicate guard.

QA: ORG-QA-014–018. Decisions: ORG-D08–D11.

### ORG-F005 — List and view Colleges

**Requirement status:** Page mapping and RBAC source-confirmed  
**Primary actor:** SUPER_ADMIN; Finance Team; HOD (HOD View appears in RBAC College Management matrix, but mapped page is not explicit).

#### Permission/scope
- SUPER_ADMIN: global view.
- Finance Team: assigned College only.
- HOD: College View is shown in one RBAC matrix; actual page entry and exact fields require confirmation.
- Manager/Student: no College Management view permission. Any contextual college name in a permitted academic UI is outside this admin function.

#### Entry/UI
- College Management / College Details pages are mapped for SUPER_ADMIN.
- Table columns, filters, search, sorting, pagination, and empty state are not source-defined.
- Finance Team/HOD view pages and fields are TBD.

#### Main flow
1. Actor opens approved College view.
2. Server resolves actor role and scope.
3. Server applies College scope before retrieving records.
4. UI renders returned authorized records/details.
5. User opens a record only if permitted.

#### Rules
- Finance Team queries must be constrained to assigned `collegeId`.
- HOD visibility is limited to assigned College+Department context; whether HOD sees a full College detail record is TBD.
- Do not rely on frontend filtering for tenant isolation.

#### Acceptance
- SUPER_ADMIN sees global records.
- Finance Team cannot retrieve another College by list, direct ID, search, or modified request.
- HOD visibility follows approved matrix; pending HOD College-view policy remains decision-bound.
- Manager/Student cannot access admin College data.

QA: ORG-QA-019–022. Decisions: ORG-D12/D13.

### ORG-F006 — Update College details

**Requirement status:** SUPER_ADMIN Full CRUD source-confirmed  
**Primary actor:** SUPER_ADMIN

- Editable fields: `name`, `domain`, `status`, and seat fields are schema candidates; exact editability must be approved. Seat counters should be treated as Module 3-managed values.
- Entry: College Details page/action (page mapped; edit form not mapped).
- Main flow: select College → open edit action → edit approved fields → submit → server validates global permission and field rules → persist approved changes → show success.
- Derived safeguards: preserve ID/createdAt; update updatedAt; validate parent/unique domain/name only if such constraints are approved.
- Finance Team/HOD are View only for College management; no College write access.
- Failure: not found, permission denied, invalid field, stale update, concurrent update behavior TBD.
- Acceptance: SUPER_ADMIN may update approved editable fields; Finance/HOD cannot write; fields outside edit allowlist cannot be changed through API.
- QA: ORG-QA-023–026. Decisions: ORG-D14/D15.

### ORG-F007 — Change College status

**Requirement status:** College `status` field source-confirmed; lifecycle semantics TBD  
**Primary actor:** SUPER_ADMIN

Schema status enum: `ACTIVE`, `SUSPENDED`, `DEACTIVATED`, `ARCHIVED`.

- Status meaning, transition matrix, confirmation requirements, reversibility, and effect on users/subscription/activity are not specified.
- UI must not offer transitions until allowed transition rules and impact are approved.
- Proposed status change flow: actor selects a permitted transition → system validates current and next state per Module 17 → confirms if required → updates status → records audit according to Module 15.
- No implication is made that College status changes subscription status or automatically disables all users.
- Acceptance: only approved transitions are accepted; invalid transition leaves state unchanged; downstream effects follow approved cross-module rules.
- QA: ORG-QA-027–030. Decisions: ORG-D16–D18.

### ORG-F008 — Create Department under College

**Requirement status:** Source-confirmed  
**Primary actor:** SUPER_ADMIN  
**Permissions:** SUPER_ADMIN Full CRUD; Finance Team View; HOD View/Update, not Create per RBAC matrix.

#### Inputs

| Field | Type | Required? | Validation / notes |
|---|---|---|---|
| `collegeId` | varchar | Yes, derived from parent relationship | Must reference existing College |
| `name` | varchar(255) | TBD | Requiredness, whitespace normalization, uniqueness TBD |
| `code` | varchar(50) | TBD | Requiredness/uniqueness/format TBD |
| `allocatedSeats` | int | TBD | Module 3 owns allocation semantics |
| `usedSeats` | int | System-managed/TBD | Module 3 owns counting |
| `hodId` | varchar | Optional/TBD | Must reference eligible HOD if assigned |

System: id, createdAt, updatedAt.

#### Main flow
1. SUPER_ADMIN selects a College.
2. Actor opens Department creation.
3. System supplies/derives parent College context.
4. Actor enters approved Department data.
5. Server validates College exists and actor is allowed to create under it.
6. Server validates approved field constraints and HOD consistency if a HOD is supplied.
7. Server creates Department under that College.
8. UI shows new Department under the correct College.

#### Business rules
- Department belongs to exactly one College via `collegeId`.
- Department cannot be created under a nonexistent College.
- Finance Team and HOD cannot create Department unless permissions are changed/approved.
- Seat fields are not recalculated here.

#### Failure
Invalid parent, insufficient permission, validation error, concurrent duplicate (if uniqueness approved), or storage failure must not create a partial/parentless Department.

#### Acceptance
- Created Department always has a valid College parent.
- Department is visible only within authorized scope.
- Non-SUPER_ADMIN create requests are denied under current source matrix.
- Seat values follow Module 3 policy; no assumed arithmetic.

QA: ORG-QA-031–035. Decisions: ORG-D19/D20.

### ORG-F009 — View/list Department and organization hierarchy

**Requirement status:** Page mapping and RBAC source-confirmed  
**Actors:** SUPER_ADMIN, Finance Team, HOD

- SUPER_ADMIN can view global Department data.
- Finance Team can view Departments within assigned College.
- HOD can view assigned Department within assigned College+Department scope.
- Manager may view permitted contextual Section/Department information for operational pages, but not administer/browse Department Management. Exact display context is TBD.
- Entry: College Details / HOD Department Management pages.
- Main flow: actor opens view → scope derived server-side → retrieve parent-linked Department/Section data → render permitted records.
- Hierarchy output must not leak records from unrelated Colleges/Departments.
- Search/filter/pagination fields and UX TBD.
- Acceptance: query results respect assigned scope; direct object ID and altered filter do not bypass scope.
- QA: ORG-QA-036–039. Decisions: ORG-D12/D21.

### ORG-F010 — Update Department details

**Requirement status:** SUPER_ADMIN Full CRUD; HOD View/Update source-confirmed  
**Primary actor:** SUPER_ADMIN or HOD in assigned scope

#### Field-level permission
- SUPER_ADMIN: update approved Department fields globally.
- HOD: Update permission is source-confirmed, but editable fields are not specified. HOD update must remain restricted to an approved field allowlist.
- Finance Team: View only.
- Manager/Student: no Department management write.

#### Main flow
1. Actor opens Department detail in authorized scope.
2. UI exposes only approved editable fields for actor.
3. Actor submits changes.
4. Server checks permission, College/Department scope, field allowlist, parent consistency, and validation.
5. Server saves permitted fields and returns updated record.

#### Constraints
- Changing `collegeId` is a high-impact move and is not assumed to be permitted by normal Department update.
- HOD cannot change department ownership/scope or HOD assignment unless separately approved.
- Seat fields follow Module 3 ownership.
- Name/code uniqueness and update conflict rules are TBD.

#### Acceptance
- Finance Team cannot update.
- HOD updates only fields explicitly approved for HOD.
- A request cannot move a Department to another College without a separately approved migration flow.
- Invalid updates leave record unchanged.

QA: ORG-QA-040–044. Decisions: ORG-D20/D22.

### ORG-F011 — Delete Department

**Requirement status:** RBAC Full CRUD for SUPER_ADMIN; delete semantics TBD  
**Primary actor:** SUPER_ADMIN

- Source matrix says SUPER_ADMIN Full CRUD, but does not state physical deletion, soft deletion, archive, or child dependency behavior.
- Department may have Sections, HOD reference, users, Courses/Assessments/Groups and other related resources.
- No cascade delete behavior is inferred.
- UI confirmation, deletion blockers, dependency summary, archive alternative, and audit behavior TBD.
- Safe interim requirement: do not implement destructive deletion until ORG-D23 is approved.
- Acceptance: deletion behavior is not treated as approved until dependencies, cascade/restrict/soft-delete policy and recovery rules are decided; unauthorized roles denied.
- QA: ORG-QA-045–047. Decision: ORG-D23.

### ORG-F012 — Create Section under Department

**Requirement status:** Section hierarchy source-confirmed; role permission TBD  
**Primary actor:** SUPER_ADMIN; HOD actor is possible but not confirmed

#### Inputs
- `departmentId` (varchar): parent Department; required by relationship.
- `name` (varchar(255)): requiredness TBD.
- `code` (varchar(50)): requiredness/uniqueness TBD.
- `id`, `createdAt`, `updatedAt`: system fields.

#### Permissions
- SUPER_ADMIN: College/Department Full CRUD suggests administration through hierarchy, but dedicated Section permission row is absent. Section-specific create permission needs explicit confirmation.
- HOD: department management view/update and source flow mentions Sections; Section create permission is not explicit.
- Finance Team: view-only if Section is included in College-level view; not explicit.
- Manager/Student: no Section configuration write.

#### Main flow
1. Authorized actor selects a Department.
2. Opens Section creation (availability depends on ORG-D24).
3. Enters approved name/code.
4. Server verifies Department exists and actor's College/Department scope.
5. Server validates constraints and creates Section under the Department.
6. Section appears within the parent Department.

#### Rules
- Section belongs to one Department.
- College scope is inherited from the parent Department.
- Do not accept a conflicting independent College ID for Section creation.
- Duplicate code/name handling is TBD.

#### Acceptance
- Section references valid Department.
- Unauthorized Section creation denied.
- No cross-College/Department Section creation.
- HOD permission is not assumed before decision.
QA: ORG-QA-048–052. Decision: ORG-D24/D25.

### ORG-F013 — View/list Sections

**Requirement status:** Page mapping source-confirmed; exact Section access matrix TBD  
**Actors:** SUPER_ADMIN, HOD; Finance Team/Manager contextual visibility TBD

- SUPER_ADMIN: global Section view via College Management/College Details mapping.
- HOD: Section list under assigned Department appears in Department Management mapping; visibility is derived from scope.
- Finance Team: Section detail visibility is not explicit; do not assume more than approved College-level views.
- Manager: only own assigned Section context should be visible; Manager is not granted organization-admin browsing.
- Student: no Section Management page; display of own Section label is academic context and not admin access.

Main flow: open approved view → scope filter server-side → return Section(s) whose Department is authorized → render. Direct ID lookups must apply parent Department and College scope.

Acceptance: no unrelated Section records returned; manager cannot enumerate Sections; HOD cannot query another Department's Sections.
QA: ORG-QA-053–056. Decisions: ORG-D24/D26.

### ORG-F014 — Update Section details

**Requirement status:** Section entity source-confirmed; permissions/field rules TBD  
**Actor:** SUPER_ADMIN; HOD may be considered subject to ORG-D24.

- Editable fields `name`, `code` are schema-supported; editability, validation and uniqueness TBD.
- `departmentId` change is a parent move, not an ordinary name/code edit; migration behavior must be decided.
- Manager/Finance Team/Student cannot update Section configuration under currently documented source permissions.
- Flow: authorized actor opens Section → edits approved fields → server checks scope, field allowlist, parent integrity and constraints → save → show updated data.
- Invalid or unauthorized changes do not mutate data.
- QA: ORG-QA-057–060. Decisions: ORG-D24/D27.

### ORG-F015 — Delete Section

**Requirement status:** Section management action is not explicitly specified  
**Actor:** SUPER_ADMIN / HOD TBD

- No dedicated Section CRUD row exists in RBAC. Do not assume Section delete permission from hierarchy alone.
- Section may be referenced by Users, CourseSection, Groups or academic records (schema relationships; exact deletion blockers not specified).
- Physical deletion/cascade/soft-delete/archive/reassignment behavior is TBD.
- No deletion should proceed until ORG-D28 is approved.
- Acceptance: no destructive Section delete without approved permission, dependent-record handling, audit, and recovery policy.
- QA: ORG-QA-061–063. Decision: ORG-D28.

### ORG-F016 — Assign or change Department HOD

**Requirement status:** Department `hodId` source-confirmed; assignment workflow TBD  
**Actor:** SUPER_ADMIN is the likely administrator based on Department Full CRUD; exact permission requires approval.

#### Input
- Department ID
- HOD User ID (nullable/unassign behavior TBD)

#### Validation
- Department must exist.
- HOD candidate must be a valid User.
- Candidate role should be HOD and organization placement should match Department College/Department; this is a **derived safeguard**, pending approval of role/assignment model.
- Schema supports one `hodId` per Department, not multiple HOD references.
- Whether one HOD may be assigned to multiple Departments is not defined.

#### Flow
1. Authorized actor opens Department.
2. Selects candidate HOD.
3. Server validates candidate, role and organization relationship according to approved policy.
4. Server updates Department.`hodId`.
5. Module 1 organization/role assignment remains consistent if user organization fields also change.
6. Updated assignment is displayed.

#### Alternate/failure
- Candidate not found/wrong role/scope → reject, no assignment.
- Department already has HOD → replace/deny/allow multiple behavior TBD.
- HOD removal/reassignment side effects and active-session behavior TBD.

Acceptance: HOD reference cannot point to a nonexistent User; role/scope checks follow approved policy; unauthorized actor denied.
QA: ORG-QA-064–067. Decisions: ORG-D29–D31.

### ORG-F017 — Validate user organization placement (Module 1 interface)

**Requirement status:** Schema-confirmed organization fields; derived relationship validation  
**Primary actor:** Module 1 user create/update/role assignment workflows; this module owns organization constraints, not user CRUD.

User fields:
- `collegeId`
- `departmentId`
- `sectionId`
- `role`

#### Relationship checks
- If Department is supplied, it must belong to the supplied College.
- If Section is supplied, its Department must match the supplied Department.
- A Manager's Section must be in the assigned College/Department.
- A Student's Section must be in the assigned College/Department.
- Finance Team should be scoped to College; HOD to College+Department; exact role assignment semantics are Module 1 decisions.
- Nullability and cross-section/multiple-assignment behavior are TBD.

#### Flow
1. Module 1 receives create/update user organization values.
2. This module's organization validation checks referenced records and parent chain.
3. If invalid, reject user assignment before persistence.
4. If valid, Module 1 performs the user operation and records role/scope change per its own rules.

Acceptance: mismatched parent references are rejected; no orphan/mismatched user assignment is stored; role lifecycle/session effects remain Module 1-owned.
QA: ORG-QA-068–072. Decisions: ORG-D32–D35.

### ORG-F018 — Resolve and filter organization hierarchy by actor scope

**Requirement status:** RBAC source-confirmed; derived system behavior  
**Actor:** System for each scoped request

- SUPER_ADMIN queries may be global.
- Finance Team queries constrained to assigned College.
- HOD queries constrained to assigned College+Department.
- Manager queries constrained to assigned College+Department+Section/permitted Group.
- Student organization context must not expose admin records or other users' private organization assignments.

#### Main flow
1. Request enters backend.
2. Authentication/role/scope context is obtained from trusted server-side identity/session.
3. Requested resource's parent chain is resolved.
4. Permission and scope constraints are applied before data is returned.
5. Only authorized records are returned.

#### Security rules
- Never trust client-provided scope IDs as authorization.
- Scope must apply to list, search, direct record access, exports and mutations.
- Do not return “exists but forbidden” details where that would leak protected information; exact 403/404 policy is TBD.
- Access denial must not change organization state.

Acceptance: changing IDs/query filters cannot expand actor scope; HOD and Manager cannot cross Department/Section boundary.
QA: ORG-QA-073–078. Decisions: ORG-D36/D37.

### ORG-F019 — Validate parent-child relationships and prevent cross-scope writes

**Requirement status:** Derived from schema relationships and RBAC scope  
**Actor:** System

#### Required integrity checks
- Department.`collegeId` references an existing College.
- Section.`departmentId` references an existing Department.
- Department parent College is authoritative for child Section College scope.
- Department.`hodId`, if set, references a valid User; exact role/scope constraints await ORG-D29–D31.
- User Department/Section chain matches College/Department chain.
- Parent moves are distinct workflows and must not be permitted through arbitrary ID edits.

#### Main flow
1. Receive create/update request.
2. Validate actor authorization.
3. Load parent/related records.
4. Check relationship integrity and scope.
5. If all checks pass, perform the write.
6. If any check fails, reject before mutation.

Acceptance: no mismatched hierarchy or cross-tenant relationship is persisted; request tampering cannot bypass parent validation.
QA: ORG-QA-079–083. Decision: ORG-D38.

### ORG-F020 — Handle College status and hierarchy impact

**Requirement status:** College status enum source-confirmed; effects TBD  
**Actor:** SUPER_ADMIN / dependent system modules

- Status values: ACTIVE, SUSPENDED, DEACTIVATED, ARCHIVED.
- College status effects on login, User status, Departments/Sections, Courses, submissions, Groups, Subscription, and billing are not described.
- This function is a cross-module decision and must reference Module 1 (user/session), Module 3 (subscription), Module 17 (state transitions), and Module 18 (failure/edge handling).
- Do not automatically cascade a College status change to child records/users until approved.

Acceptance: approved College status transition changes only the defined record/state; downstream effects match approved decision; invalid transition is rejected.
QA: ORG-QA-084–087. Decisions: ORG-D16–D18/D39.

## 9. Business rules and validations

| Rule ID | Rule | Evidence / status |
|---|---|---|
| ORG-R01 | Department must belong to one College through `collegeId`. | Source-confirmed schema |
| ORG-R02 | Section must belong to one Department through `departmentId`. | Source-confirmed schema |
| ORG-R03 | Section College scope is inherited from Department → College. | Derived |
| ORG-R04 | Finance Team data queries are scoped to assigned College. | Source-confirmed RBAC |
| ORG-R05 | HOD data queries are scoped to assigned College+Department. | Source-confirmed RBAC |
| ORG-R06 | Manager operations are scoped to assigned College+Department+Section/permitted Group. | Source-confirmed RBAC |
| ORG-R07 | Student admin access is not granted by organization hierarchy; student access is self/permitted academic data. | Source-confirmed RBAC |
| ORG-R08 | Frontend visibility must not replace backend authorization. | Source-confirmed RBAC |
| ORG-R09 | Parent relationship must be checked on create/update; child must not be attached to an unrelated parent. | Derived |
| ORG-R10 | HOD assignment must not create a dangling `hodId`; detailed eligibility is TBD. | Derived + decision |
| ORG-R11 | Seat counters and allocation arithmetic are owned by Module 3. | Cross-module boundary |
| ORG-R12 | Physical delete/cascade/soft delete cannot be inferred from “Full CRUD”. | TBD / decision |
| ORG-R13 | College status transitions and downstream effects cannot be inferred from EntityStatus enum alone. | TBD / decision |
| ORG-R14 | Onboarding-to-College linkage is not defined by the current schema excerpt. | Source gap |

## 10. Validation matrix

| Validation ID | Condition | Applies to | Expected behavior | Status |
|---|---|---|---|---|
| ORG-V01 | College parent does not exist | Department create/update | Reject before write | Derived |
| ORG-V02 | Department parent does not exist | Section create/update | Reject before write | Derived |
| ORG-V03 | User Department belongs to another College | User org assignment | Reject before user persistence | Derived |
| ORG-V04 | User Section belongs to another Department | User org assignment | Reject before user persistence | Derived |
| ORG-V05 | Actor is outside College scope | Any organization read/write | Deny and do not disclose data | RBAC |
| ORG-V06 | HOD outside Department scope | Department/Section operation | Deny | RBAC |
| ORG-V07 | Manager outside assigned Section | Section-scoped operation | Deny | RBAC |
| ORG-V08 | Attempt to modify protected parent IDs | Department/Section update | Reject unless approved move workflow | Derived |
| ORG-V09 | HOD candidate role/scope invalid | HOD assignment | Reject per approved eligibility rules | TBD |
| ORG-V10 | College status transition invalid | Status change | Reject, no state change | TBD / Module 17 |
| ORG-V11 | Name/code duplicate | College/Department/Section | Behavior TBD; no assumed unique constraint | TBD |
| ORG-V12 | Seat values inconsistent/over quota | College/Department | Validate through Module 3 policy; do not calculate here | Module 3 |

## 11. Data and state reference

### 11.1 College

| Field | Type | Meaning | Required/editability/constraint |
|---|---|---|---|
| `id` | varchar | Identifier | Primary identifier; generation TBD |
| `name` | varchar(255) | College name | Requiredness and uniqueness TBD |
| `domain` | varchar(255) | Domain | Intended meaning (email domain, website domain, etc.) TBD |
| `status` | EntityStatus | College lifecycle/status | Values source-confirmed; transition behavior TBD |
| `totalSeats` | int | Capacity-related field | Module 3 owns semantics/calculation |
| `usedSeats` | int | Usage-related field | Module 3 owns semantics/calculation |
| `createdAt` | timestamp | Creation time | Schema field |
| `updatedAt` | timestamp | Last update time | Schema field |

### 11.2 Department

| Field | Type | Meaning | Required/editability/constraint |
|---|---|---|---|
| `id` | varchar | Identifier | Generation TBD |
| `collegeId` | varchar | Parent College | Parent relationship source-confirmed |
| `name` | varchar(255) | Department name | Requiredness/uniqueness TBD |
| `code` | varchar(50) | Department code | Requiredness/uniqueness/format TBD |
| `allocatedSeats` | int | Seat allocation | Module 3 |
| `usedSeats` | int | Seat usage | Module 3 |
| `hodId` | varchar | HOD reference | Assignment rules TBD |
| `createdAt`, `updatedAt` | timestamp | Record timestamps | Schema fields |

### 11.3 Section

| Field | Type | Meaning | Required/editability/constraint |
|---|---|---|---|
| `id` | varchar | Identifier | Generation TBD |
| `departmentId` | varchar | Parent Department | Parent relationship source-confirmed |
| `name` | varchar(255) | Section name | Requiredness/uniqueness TBD |
| `code` | varchar(50) | Section code | Requiredness/uniqueness/format TBD |
| `createdAt`, `updatedAt` | timestamp | Record timestamps | Schema fields |

### 11.4 CollegeOnboarding

| Field | Type | Meaning | Constraint/status |
|---|---|---|---|
| `id` | varchar | Identifier | Schema field |
| `collegeName` | varchar(255) | Proposed College name | Requiredness/uniqueness TBD |
| `contactPerson` | varchar(255) | Contact | Requiredness TBD |
| `contactEmail` | varchar(255) | Contact email | Format/uniqueness/verification TBD |
| `phone` | varchar(30) | Contact phone | Format TBD |
| `address` | text | Address | Requiredness TBD |
| `studentCount` | int | Expected student count | Range TBD |
| `departmentDetails` | json | Proposed department information | JSON schema TBD |
| `paymentStatus` | varchar(30) | Payment-related status | Allowed values and business meaning TBD |
| `reviewedById` | varchar | Reviewer reference | Review workflow TBD |
| `reviewedAt` | timestamp | Review time | Review workflow TBD |
| `createdAt`, `updatedAt` | timestamp | Timestamps | Schema fields |

**Schema gap:** the supplied schema does not show an explicit `CollegeOnboarding.collegeId` or a College foreign key. The onboarding-to-College linkage must be resolved before conversion is finalized.

### 11.5 User organization references (Module 1 interface)

| Field | Type | Module 2 responsibility |
|---|---|---|
| `collegeId` | varchar | Verify referenced College exists and role scope aligns |
| `departmentId` | varchar | Verify Department belongs to selected College |
| `sectionId` | varchar | Verify Section belongs to selected Department |
| `role` | Role | Role-specific organization constraints are coordinated with Module 1 |

User create/update, role assignment, credential and account lifecycle remain Module 1-owned.

### 11.6 State-transition table

| Current state | Trigger | Next state | Side effects | Status |
|---|---|---|---|---|
| College status enum value | SUPER_ADMIN requests status change | One of ACTIVE/SUSPENDED/DEACTIVATED/ARCHIVED | Downstream user/subscription/academic effects TBD | Decision required |
| CollegeOnboarding record | Review/approve/reject action | Status/state not represented clearly in schema | reviewedById/reviewedAt may be populated | Decision required |
| Department / Section | Delete/archive/deactivate action | Lifecycle field not shown in schema | Child and academic dependency behavior TBD | Decision required |

## 12. Notifications, audit, and privacy

### 12.1 Audit
RBAC source calls out College creation/update/deletion and Department/Section changes as sensitive actions to consider for AuditLog. Audit schema fields include actor, target, action, metadata, IP, user agent, and timestamp.

**Derived requirement:** organization changes should be traceable through Module 15's approved audit events. Exact event names, metadata, retention, and visibility are Module 15 decisions.

### 12.2 Notifications
No organization-specific notification schedule/recipient is specified. Any onboarding review, College creation, HOD assignment, or organization change notification must be defined in Module 13 and approved before implementation.

### 12.3 Privacy
Onboarding contains contact details, phone and address. Access roles, data minimization, retention, and external-submit privacy notice are TBD. Do not expose onboarding contact details to broad College/Department users without approved access rules.

## 13. Cross-module dependencies

| Module | Dependency |
|---|---|
| 1 — Authentication & User Lifecycle | Role identity, user CRUD, role assignment, user organization fields, access/session changes after org assignment changes |
| 3 — Subscription & Seat Management | College/Department seat fields, seat capacity/allocation/usage, College vs Subscription state |
| 4 — Course & Enrollment | Course-to-Section assignments and enrollment references |
| 10 — Groups | Group and section scope; Manager permitted Group behavior |
| 13 — Notifications | Onboarding, HOD assignment, status and organization change notifications |
| 15 — Audit Logging | Organization event names, actor/target metadata, retention |
| 17 — Functional State Machines | College, onboarding, Department/Section lifecycle |
| 18 — Edge Cases & Failure Handling | Concurrent edits, stale parent links, partial failures, status propagation, delete dependencies |

## 14. Edge cases and failure handling

| ID | Condition | Expected behavior | Decision/status |
|---|---|---|---|
| ORG-E01 | Department creation references nonexistent College | Reject; no orphan Department | Derived |
| ORG-E02 | Section creation references nonexistent Department | Reject; no orphan Section | Derived |
| ORG-E03 | Department is moved to another College | Treat as high-impact parent migration; block ordinary update pending policy | TBD |
| ORG-E04 | Section is moved to another Department | Treat as parent migration; block ordinary update pending policy | TBD |
| ORG-E05 | HOD assigned to Department in another College/Department | Reject under approved role/scope consistency policy | Derived/TBD |
| ORG-E06 | College has Departments and user/academic references when deletion requested | Do not cascade implicitly; policy TBD | TBD |
| ORG-E07 | Department has Sections and dependent records when deletion requested | Do not cascade implicitly; policy TBD | TBD |
| ORG-E08 | Section has assigned Users/CourseSection/group/academic references when deletion requested | Do not cascade implicitly; policy TBD | TBD |
| ORG-E09 | Actor changes request College/Department/Section IDs | Recompute scope server-side; deny mismatch | RBAC |
| ORG-E10 | HOD or Manager assignment changes during active session | Session/authorization refresh policy belongs to Module 1 | TBD |
| ORG-E11 | Duplicate College/Department/Section name or code | Constraint and feedback TBD; avoid unsupported uniqueness assumption | TBD |
| ORG-E12 | College status changes while subscription status differs | No assumed cascade; coordinate Module 3/17 | TBD |
| ORG-E13 | Seat count exceeds plan/capacity | Module 3 determines response; this module must not silently adjust counters | Module 3 |
| ORG-E14 | Concurrent users update same Department/Section | Conflict detection and merge policy TBD | TBD |
| ORG-E15 | Onboarding conversion is retried | Idempotency/linking policy TBD; avoid duplicate College creation | TBD |
| ORG-E16 | External/onboarding contact data is requested by unauthorized role | Deny/minimize response | Derived privacy safeguard |

## 15. Open decisions / decision log

| Decision ID | Question / conflict | Options (if known) | Recommendation (not approved) | Impact | Owner | Status |
|---|---|---|---|---|---|---|
| ORG-D01 | Who can submit College onboarding? | SUPER_ADMIN-only; public/external form; both | Clarify intake actor before UI/API | Onboarding access | Product/Security | Open |
| ORG-D02 | Required onboarding fields and duplicate detection? | Required fields/duplicate match criteria TBD | Define field contract and duplicate policy | Data quality | Product | Open |
| ORG-D03 | Is onboarding a review/approval workflow? | Direct create; review/approve/reject; request changes | Define status and transitions | College setup | Product | Open |
| ORG-D04 | What do `reviewedById`/`reviewedAt` mean? | Single review; multiple review events; audit-only | Define use | Review traceability | Product | Open |
| ORG-D05 | What are `paymentStatus` values and role? | Payment state vs onboarding state | Do not overload it as approval state | Billing/onboarding | Product/Finance | Open |
| ORG-D06 | Onboarding-to-College link? Schema lacks explicit FK. | Add FK; metadata/link table; another model | Resolve schema before conversion | Duplicate prevention | Product/Architecture | Open |
| ORG-D07 | Does review approval automatically create College? | Yes; separate action; after subscription | Keep conversion separate until approved | Workflow sequencing | Product | Open |
| ORG-D08 | College creation required fields/default status? | Fields/defaults TBD | Approve field contract | College create | Product | Open |
| ORG-D09 | College name/domain uniqueness and domain semantics? | Unique domain; unique name; neither; scoped | Define normalization/uniqueness | Validation | Product/Architecture | Open |
| ORG-D10 | Initial College seat values? | Module 3 controlled defaults; explicit inputs | Avoid user-editable counters | Seat data | Finance/Product | Open |
| ORG-D11 | Direct College creation vs onboarding-only? | Both; onboarding-only; direct admin only | Confirm supported entry path | Creation flow | Product | Open |
| ORG-D12 | HOD College View permission/page? RBAC matrix says View; Page mapping lists HOD Department Management. | Full College view; limited context; no admin page | Clarify actual view scope | RBAC/UI | Product/Security | Open |
| ORG-D13 | Finance Team College/Department/Section fields? | College-only; department detail; section detail | Least-privilege scope pending review | Data exposure | Product/Security | Open |
| ORG-D14 | Which College fields are editable after creation? | Name/domain/status; restricted fields | Explicit field allowlist | College edit | Product | Open |
| ORG-D15 | College domain semantics and validation? | Website domain; email domain; other | Define purpose/format | College identification | Product | Open |
| ORG-D16 | Meaning and transitions of College EntityStatus values? | ACTIVE/SUSPENDED/DEACTIVATED/ARCHIVED transition matrix | Define in Module 17 | Lifecycle | Product/Security | Open |
| ORG-D17 | Does College status affect User login/session? | Immediate deny; grace; no automatic effect | Coordinate Module 1 | Access | Product/Security | Open |
| ORG-D18 | Does College status couple to Subscription status? | Independent; one derives from another | Define precedence in Module 3 | Billing/access | Product/Finance | Open |
| ORG-D19 | Department required fields and code/name uniqueness? | Unique within College; global; none | Define constraint | Department validation | Product | Open |
| ORG-D20 | HOD's Department update field allowlist? | Name/code; academic metadata; seats; HOD assignment | Limit HOD to approved fields | HOD permission | Product/Security | Open |
| ORG-D21 | Department view list/detail fields and filters? | Fields/search/filter/pagination TBD | Use mapped page as context only | UI | Product/Design | Open |
| ORG-D22 | Can Department change College parent? | No; controlled migration; admin move | Block normal parent edit | Tenant integrity | Product/Architecture | Open |
| ORG-D23 | Department delete semantics and dependent-record policy? | Restrict; soft delete; archive; cascade | Avoid cascade unless explicitly approved | Data loss | Product/Architecture | Open |
| ORG-D24 | Who can create/update/delete Section? | SUPER_ADMIN; HOD; both; read-only HOD | RBAC has no explicit Section row; confirm | Section CRUD | Product/Security | Open |
| ORG-D25 | Section required fields and name/code uniqueness? | Unique per Department; other | Define | Section validation | Product | Open |
| ORG-D26 | Finance/Manager Section visibility? | No admin access; contextual limited display | Least-privilege context | Data scope | Product/Security | Open |
| ORG-D27 | Can Section change Department parent? | No; controlled migration; admin move | Block normal parent edit | Hierarchy integrity | Product/Architecture | Open |
| ORG-D28 | Section delete semantics and dependent-record handling? | Restrict; archive; soft delete; cascade | No implicit cascade | Data loss | Product/Architecture | Open |
| ORG-D29 | Who may assign/change Department HOD? | SUPER_ADMIN; HOD; delegated role | SUPER_ADMIN likely based on CRUD, confirm | Assignment | Product | Open |
| ORG-D30 | HOD cardinality? | One HOD per Department; HOD multiple Departments; multiple HODs | Schema currently shows one `hodId` | Role model | Product | Open |
| ORG-D31 | HOD eligibility and removal behavior? | Role+same College/Department; other | Validate role/scope | Security | Product/Security | Open |
| ORG-D32 | Can user be assigned multiple Departments/Sections? | Single refs (schema); multi-membership | Schema suggests single refs; approve target | User placement | Product | Open |
| ORG-D33 | Is Student Section required? | Required; optional; enrollment-derived | Define academic placement | Enrollment | Product | Open |
| ORG-D34 | Can Manager manage multiple Sections? | One sectionId; multi-section assignment | Schema suggests single Section; approve | Manager scope | Product | Open |
| ORG-D35 | How is academicBatch validated? | Free text; controlled academic year; section-derived | Define Module 4 relationship | Student placement | Product | Open |
| ORG-D36 | Exact deny response for unauthorized organization record? | 403; 404; generic deny | Security review | Info leakage | Security/Architecture | Open |
| ORG-D37 | How are active sessions refreshed after org assignment changes? | Immediate scope refresh; token expiry; revoke session | Module 1 coordination | Access revocation | Security | Open |
| ORG-D38 | Parent-child integrity and concurrent update strategy? | Transaction; optimistic locking; other | Define implementation-level contract separately | Integrity | Architecture | Open |
| ORG-D39 | College status cascade to child Department/Section? | None; inherited; explicit transitions | No implicit cascade | Lifecycle | Product | Open |
| ORG-D40 | Audit/notification events for org changes? | Event list/recipients/retention TBD | Coordinate Modules 13/15 | Traceability | Product/Security | Open |
| ORG-D41 | College/Department seat fields editable here or Module 3 only? | Module 3 owns; role-specific | Keep seat counters Module 3-owned | Seat consistency | Finance/Product | Open |
| ORG-D42 | College/Department/Section archive/restore capability? | Archive; deactivate; delete; none | Define reversible lifecycle | Data retention | Product | Open |

## 16. Acceptance criteria (module-level)

- AC-ORG-01: College/Department/Section hierarchy is represented using the schema parent references.
- AC-ORG-02: A Department cannot be created under a nonexistent College.
- AC-ORG-03: A Section cannot be created under a nonexistent Department.
- AC-ORG-04: Section College scope is resolved through Department → College.
- AC-ORG-05: SUPER_ADMIN organization management operations are global within the documented CRUD matrix.
- AC-ORG-06: Finance Team organization reads are restricted to its assigned College.
- AC-ORG-07: HOD organization reads/updates are restricted to assigned College+Department and approved fields.
- AC-ORG-08: Manager cannot enumerate or administer unrelated Sections.
- AC-ORG-09: Student cannot access organization administration.
- AC-ORG-10: Client-supplied IDs cannot bypass backend permission/scope/parent checks.
- AC-ORG-11: User organization references must match the College → Department → Section chain.
- AC-ORG-12: No delete, status cascade, seat calculation, onboarding conversion, or HOD assignment rule is treated as approved while its decision remains open.
- AC-ORG-13: College status effects follow approved Module 1/3/17 rules.
- AC-ORG-14: Organization audit/notification behavior references approved Modules 13/15.

## 17. QA coverage

Detailed test cases are in [QA-TEST-CASES.md](./QA-TEST-CASES.md). The QA companion covers:
- onboarding creation/view/review/conversion;
- College create/read/update/status;
- Department create/read/update/delete;
- Section create/read/update/delete;
- HOD assignment;
- user organization placement;
- role/scope enforcement;
- hierarchy integrity and cross-parent tampering;
- status and delete decision gates.

QA cases that depend on unresolved decisions explicitly mark the expected behavior as decision-gated; they must not be interpreted as approval of a product policy.

## 18. Review and approval checklist

- [ ] Every in-scope function is inventoried.
- [ ] Source-confirmed facts are separated from derived requirements, recommendations, and TBDs.
- [ ] Role permissions and organizational scope are explicit.
- [ ] Onboarding, College, Department, Section, HOD assignment, and hierarchy validation functions are specified.
- [ ] Input fields, source types, validation and field-level uncertainty are documented.
- [ ] Main flows, data/state changes, success, failure, alternate flows, and edge cases are covered.
- [ ] Delete/cascade, status transitions, seat counters, and onboarding link decisions remain open where source is silent.
- [ ] Acceptance criteria are observable.
- [ ] QA tests include preconditions, steps, expected results and traceability.
- [ ] Dependencies to Modules 1, 3, 4, 10, 13, 15, 17, 18 are documented.
- [ ] Product/Security/Architecture decisions are recorded and assigned.
- [ ] No recommendation is presented as approved target behavior.

## 19. Source references

- [Business Flow (original)](../../docs/source-text/Business%20Flow.md)
- [Business Flow v2](../../docs/source-text/Business%20Flow%20v2.md)
- [Page-to-DB Mapping (original)](../../docs/source-text/Page%20-%20DB%20Mapping.md)
- [Page-to-DB Mapping v2](../../docs/source-text/Page%20-%20DB%20Mapping%20v2.md)
- [RBAC](../../docs/source-text/RBAC.md)
- [Schema](../../docs/source-text/scheema.md)
- [Functional Specification Standard](../FUNCTIONAL-SPECIFICATION-STANDARD.md)

**Review gate:** Module 2 only. Document commit does not approve its target behavior. Product/security/architecture decisions remain open.
