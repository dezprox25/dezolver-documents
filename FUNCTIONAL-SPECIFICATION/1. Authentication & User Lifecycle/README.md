# 1. Authentication & User Lifecycle

**Document ID:** DEZ-AUTH-FS-001  
**Document type:** Functional Specification  
**Version:** 0.4 — Product decisions approved and identity model updated  
**Status:** Product decisions AUTH-D01–AUTH-D25 approved; implementation details and cross-module integration remain subject to technical review  
**Repository target:** `FUNCTIONAL-SPECIFICATION/1. Authentication & User Lifecycle/README.md`  
**Review basis:** Functional Specification Standard v1.0; Business Flow (original and v2); Page-to-DB Mapping (original and v2); RBAC; schema

> **Important:** This document distinguishes source evidence from target requirements. A source page mapping or schema field does not prove a workflow is implemented. Product Owner decisions AUTH-D01–AUTH-D25 are approved target direction; technical details delegated to Modules 2, 3, 13, 15, 17, and 18 remain cross-module work. Approved decisions in Section 16 supersede conflicting “TBD/decision required” statements retained in older function-level narrative until those sections are normalized.


## Approved target decisions — Product Owner review (2026-11-05)

The decisions in Section 16 are approved target requirements. This section is the concise implementation reference; cross-module ownership and detailed technical design remain applicable.

### Identity, role, login, and onboarding
- One global user identity per person, with globally unique normalized email and one email/password credential for the demo.
- One shared login experience. After authentication, resolve the active role/membership and display role-specific workspace identity/dashboard.
- FINANCE_TEAM is a first-class role and must be added to the role model/schema enum.
- Use a global User identity plus organizational membership records (conceptually UserMembership): membership holds college, role, department/section/group scope, and membership status. A user may hold multiple memberships across colleges and switch active membership after login. SUPER_ADMIN has platform-wide scope; tenant roles are scoped to their memberships.
- Account creation authority is role-specific. All account creation must validate the creator’s role and scope.

| Created by | SUPER_ADMIN | FINANCE_TEAM | HOD | MANAGER | STUDENT |
|---|---:|---:|---:|---:|---:|
| SUPER_ADMIN | — | Yes | Yes | Yes | Yes |
| FINANCE_TEAM | No | No | Yes | Yes | No |
| HOD | No | No | No | Yes | Yes |
| MANAGER | No | No | No | No | Yes |
| STUDENT | No | No | No | No | No |

- Finance Team operates at college scope: manage subscription/plan and finance relationship between the college and Super Admin; create HODs and Managers for departments. Super Admin may create HODs/Managers for a college upon request. Finance Team does not directly create students under the approved creation matrix.
- Email verification is required for SUPER_ADMIN, FINANCE_TEAM, HOD, and MANAGER; required for STUDENT where an email is available. Password recovery uses email reset for all roles; Super Admin additionally has an admin recovery path.
- Role-specific default destinations: SUPER_ADMIN dashboard, Finance dashboard, HOD dashboard, Manager dashboard, Student dashboard. If a user was redirected to login from an authorized protected route, return them to that route after authentication.

### Credential security and authentication protection
- Demo authentication: email/password. SSO/passwordless/other methods are future extensions, not demo scope.
- Password policy: minimum 12 characters; maximum at least 64; no mandatory character-class combinations; allow spaces and password managers/paste; block commonly used/compromised passwords; password history blocks last 5; use Argon2id (preferred) or appropriately configured bcrypt.
- Login protection: 5 consecutive failed attempts triggers 15-minute account lockout; 10 attempts per IP per 15 minutes; reset consecutive failure count on successful authentication while continuing to monitor repeated abuse.
- Password-reset token: single-use, expires after 15 minutes. Password reset/change revokes all sessions.
- MFA policy is role-based and DEFERRED FROM DEMO: mandatory for SUPER_ADMIN and FINANCE_TEAM in production; recommended for HOD/MANAGER; standard password protection for STUDENT.
- Shared login, backend authorization, account status, membership scope, and ownership checks must be enforced server-side.

### Status, sessions, and membership lifecycle
- User status enum: INVITED, ACTIVE, SUSPENDED, DEACTIVATED, ARCHIVED. Only ACTIVE global users may authenticate. INVITED, SUSPENDED, DEACTIVATED, and ARCHIVED cannot log in.
- status is authoritative; remove redundant isActive. Email verification, MFA state, and mustChangePassword (if introduced) are independent properties, not account lifecycle status.
- Sessions: 15-minute access token; 7-day refresh session; remember-me 30 days; automatic refresh before access token expiry; multiple concurrent devices allowed; users can log out all devices. Idle timeout: 30 minutes for SUPER_ADMIN/FINANCE_TEAM/admin-class users; 7 days for STUDENT.
- User can view active sessions and revoke individual sessions.
- Invalidation: password change/reset, email change, and account suspension/deactivation/archive/invitation revoke all sessions. Email change requires verification. Role/department/section change revokes sessions for the affected membership; membership suspension/removal revokes that membership’s sessions. Name/profile-photo changes do not revoke sessions.
- Impersonation is allowed only with a reason, audit trail, and restricted actions. Detailed restriction matrix, disclosure, and termination behavior must be specified before implementation.
- No permanent account deletion. User removal from operational use is by deactivation/archive only; preserve historical attribution and apply retention rules by status/data type.

### User permissions, profile, audit, and notifications
- Manager (within assigned scope): view, create individual students, bulk-import students, edit student details, suspend, and deactivate students. Email changes, password reset, reactivation, and department/section reassignment are restricted; cannot permanently delete, change role, or transfer college.
- Self-editable: full name, email (verify new email and revoke sessions), phone (verify if used for auth/recovery), profile photo (type/size validation), password (current-password verification or secure recovery), and explicitly approved personal fields. Role, college, department, section, academic batch, student/register number, and account status are admin-controlled.
- Email is globally unique, normalized (trim + case-insensitive comparison), and database-enforced. Email changes require verification.
- Audit all listed events: successful/failed login, logout/session revocation, password/MFA changes, user creation/update/suspension/deactivation, role/scope changes, impersonation, and profile changes. Apply Module 15 event schema, privacy controls, retention, and exceptions; never log secrets.
- Security-sensitive email notifications: password changed/reset and email changed. Do not email for MFA enable/disable (MFA is deferred in demo), role/scope change, account suspension/deactivation, or new-device login under current decision.
- Field-level permissions should be defined for sensitive fields; ordinary CRUD/action permissions remain role/scope-based.
- Retention varies by account status and data type; no permanent deletion. Detailed retention belongs to applicable data/governance modules.


## 1. Purpose and module outcome

Define DEZOLVER's identity and account context used by protected application functions: identifying the user, resolving role and organizational scope, managing user records, account status and profile, and handling sessions and security-related context.

The module outcome is a reliable authenticated identity and an authorization context that downstream modules can use without bypassing role, tenant, scope, or ownership checks.

## 2. Scope

### 2.1 In scope
- Authentication entry point and authenticated identity (mechanics remain TBD).
- Role and organizational scope resolution.
- User Management pages and user-record operations identified in RBAC/Page-to-DB Mapping.
- Profile & Settings page mapping.
- Account status fields and lifecycle decisions.
- Session records, expiry/revocation fields, and Security / Sessions page mapping.
- MFA and impersonation fields present in the schema, without assuming their workflows are enabled.
- User/account/security audit context.
- College onboarding context insofar as it affects Finance Team/user provisioning.

### 2.2 Out of scope
- College, department, and section creation/maintenance (Module 2).
- Subscription, seat entitlement, and billing eligibility (Module 3).
- Notification delivery for invites, password recovery, verification, and security events (Module 13).
- Canonical audit event catalog, retention, export, and audit UI (Module 15).
- Canonical account/session state machines (Module 17).
- Shared system failure and retry conventions (Module 18).
- Feature-specific authorization decisions owned by Courses, Labs, Groups, Assessments, Contests, and other modules.

## 3. Evidence labels and interpretation

| Label | Meaning in this document |
|---|---|
| **Source-confirmed** | Explicitly represented in a supplied Business Flow, RBAC, Page-to-DB Mapping, or schema source. |
| **Derived** | A necessary or reasonable conclusion from multiple source facts, identified as derived rather than directly stated. |
| **Recommendation** | Suggested target behavior; not approved. |
| **TBD / Decision Required** | Source is missing, incomplete, or conflicting. Do not implement as settled behavior. |
| **Legacy-verified** | Directly verified in legacy code or forensic evidence. **None is claimed in this draft.** |
| **Target requirement** | Product Owner-approved target behavior; see Section 16. Approved decisions supersede earlier decision-gated draft wording. |

## 4. Sources and evidence traceability

| Source | Relevant evidence | Limitations |
|---|---|---|
| [Business Flow v2](../../docs/source-text/Business%20Flow%20v2.md) | Stakeholder roles; SUPER_ADMIN starts College Onboarding; College Created → Setup → Finance Team → Subscription; Finance Team handles plan/subscription; HOD department administration; Manager assigned-section scope. | Does not define login, user invitation, credential, or session workflows. |
| [Business Flow (original)](../../docs/source-text/Business%20Flow.md) | Earlier version of stakeholder flow and responsibilities. | High-level flow only; not a detailed auth specification. Differences should be reviewed when source versions diverge. |
| [Page-to-DB Mapping v2](../../docs/source-text/Page%20-%20DB%20Mapping%20v2.md) | Super Admin Login; User Management; Finance Team Management; Finance HOD Accounts; HOD Managers/Lab Staff and Students; Manager Students; Profile & Settings; Security / Sessions. | Page/entity mapping does not define field-level behavior, UI controls, or permissions beyond the associated pages. |
| [Page-to-DB Mapping (original)](../../docs/source-text/Page%20-%20DB%20Mapping.md) | Earlier page mappings for login, User Management, HOD Accounts, Profile & Settings, Security / Sessions. | Screen/entity mapping only. Preserve as source version; do not infer workflows. |
| [RBAC](../../docs/source-text/RBAC.md) | Five role definitions; platform/page/API permission matrices; college/department/section/group/ownership scope; backend enforcement; audit guidance. | User Management permissions differ between platform-level and page/API matrices. |
| [Schema](../../docs/source-text/scheema.md) | User, RefreshSession, AuditLog fields; Role and EntityStatus enums; MFA and impersonation-related fields. | Conceptual schema is not proof of deployed constraints, implementation, or workflow. |

### 4.1 Source facts relevant to this module

- RBAC identifies `SUPER_ADMIN`, `FINANCE_TEAM`, `HOD`, `MANAGER`, and `STUDENT`.
- RBAC scopes: SUPER_ADMIN global; FINANCE_TEAM college; HOD college + department; MANAGER college + department + assigned section/permitted group; STUDENT self + permitted academic data.
- RBAC states that role permission is not enough: backend permission, scope, and ownership checks are required.
- Page mapping lists a **Super Admin Login** page backed by `User` and `RefreshSession`. It does not explicitly map a shared login page for every role.
- Page mapping identifies SUPER_ADMIN User Management, Finance Team Management/HOD Accounts, HOD Managers/Lab Staff and Students, Manager Students, Profile & Settings, and Security / Sessions.
- Source schema includes `User.status`, redundant `User.isActive` (remove in target), MFA fields, and `RefreshSession` expiry/revocation/impersonation fields.
- Business Flow v2 establishes onboarding context but not the mechanics of creating or inviting user accounts.

### 4.2 Source-version comparison (Module 1 relevance)

| Source pair | Confirmed overlap | Relevant differences / authoring treatment |
|---|---|---|
| Business Flow (original) vs v2 | Both describe SUPER_ADMIN, FINANCE_TEAM, HOD, MANAGER, and STUDENT responsibilities and the college/academic hierarchy. | v2 makes the college-onboarding sequence and Finance Team subscription/plan flow more explicit and adds backend/Judge0 detail to the overall flow. Neither version defines authentication credentials, invitations, login routing, account states, or session behavior. Treat both as business context only. |
| Page-to-DB Mapping (original) vs v2 | Both map Super Admin Login to User/RefreshSession; Super Admin User Management; Finance Team HOD Accounts; HOD/Manager user pages; Profile & Settings; Security / Sessions. | v2 expands related entities on HOD Managers/Lab Staff and Students (including Group/GroupMember-related references) and explicitly includes AuditLog in Profile & Settings mappings for Finance/HOD/Manager/Student. These are page/entity associations, not workflow or field-level requirements. |
| RBAC platform matrix vs Page/API matrices | Both define role and data-scope boundaries and require backend enforcement. | User Management differs for MANAGER in source matrices; AUTH-D12 now resolves the target policy. Source conflict remains historical evidence. |
| RBAC/Business Flow vs schema Role enum | RBAC and Business Flow list FINANCE_TEAM. | Source schema omits FINANCE_TEAM; approved AUTH-D01 requires adding it to the target role enum/model. |

### 4.3 Standard compliance map

| Functional Specification Standard v1.0 requirement | Module 1 location | Compliance treatment |
|---|---|---|
| Metadata/review status | Header | Version, document status, repository path, and behavior-approval distinction are explicit. |
| Purpose, outcome, scope | Sections 1–2 | In/out scope defined. |
| Source evidence and terminology | Sections 3–5, 19 | Both original and v2 source pairs are identified; unsupported behavior remains TBD. |
| Actors, permissions, scope | Section 6 | Source roles, scope, User Management matrix conflict, Profile & Settings matrix, and server-side security principles are captured. |
| Navigation/entry points | Section 7 | Confirmed page mappings separated from unknown routes/actions. |
| Functional inventory | Section 8 | AUTH-F001 onward enumerated; delete is separated from deactivate/archive (AUTH-F016). |
| Function-level template | Section 9 | Function details include actors, permissions/scope, preconditions, entry, inputs, flows, rules, validation, system/data behavior, outcomes, failure/edge paths, audit/privacy, AC, QA, and decisions; system-only UI is N/A and unknown behavior is explicitly TBD. |
| Business rules/validation | Sections 9–10 | Stable AUTH-BR/AUTH-V identifiers. |
| Data/state | Sections 11–12 | Source fields/enums, constraints known vs unknown, and state/session decision gaps. |
| Audit/notifications/privacy/integrations | Section 13 | Module 13/15 ownership referenced; no unapproved triggers assumed. |
| Cross-module dependencies | Section 14 | Dependencies listed. |
| Edge/failure handling | Sections 9, 15 | Function-level and module-level cases. |
| Decision log | Section 16 | Open decisions, recommendations, and owner/status separated. |
| Acceptance and QA | Sections 17 and [QA test case detail](./QA-TEST-CASES.md) | AC included; full QA preconditions/steps/expected results are in linked test artifact. Decision-gated tests are not execution-ready. |
| Review/approval checklist | Section 18 | Explicit product approval gate. |

## 5. Terminology

| Term | Meaning |
|---|---|
| Authenticated user | User identity established by the approved authentication mechanism. |
| Role | RBAC role assigned to the user. |
| Organizational scope | College, department, section, group, or permitted academic scope associated with the user/resource. |
| Ownership | Resource-level check tying a private record to the authenticated user, e.g. student-owned records. |
| Account status | `User.status` enum value from schema. |
| Active flag | Deprecated/redundant `User.isActive`; remove it. `User.status` is the lifecycle source of truth. |
| Refresh session | `RefreshSession` schema record associated with a User. |
| Impersonation | Approved feature requiring reason, audit, and restricted actions; detailed operational controls remain to be specified. |

## 6. Actors, roles, permissions, and scope

### 6.1 Role scope from RBAC

| Role | Source-described responsibility | Maximum source scope |
|---|---|---|
| SUPER_ADMIN | Platform administration | Global |
| FINANCE_TEAM | College billing/subscription administration | Assigned College |
| HOD | Department-level academic administration | College + Department |
| MANAGER | Section-level students and operational activities | College + Department + assigned Section / permitted Group |
| STUDENT | Personal academic activities and submissions | Self + permitted academic resources |

### 6.2 User Management permission evidence

RBAC contains two different User Management permission representations:

| Role | Platform-level matrix | Page/API matrix | Draft treatment |
|---|---|---|---|
| SUPER_ADMIN | Full CRUD | V/C/U/D | Full CRUD source-confirmed. |
| FINANCE_TEAM | View/Create* | V/C* | View/Create within college scope; exact Finance user-creation targets require clarification. |
| HOD | View/Create/Update* | V/C/U* | View/Create/Update within assigned college/department. |
| MANAGER | View* | V/U* | Approved target decision AUTH-D12: Managers may perform approved student management actions within assigned scope; sensitive operations remain restricted. |
| STUDENT | View own | V OWN | Own-user view; profile update is separately covered by Profile & Settings matrix. |

`*` means applicable scope restriction. Delete is only allowed where explicitly listed.

**Approved target decision:** Manager permissions are defined in Section 16/AUTH-D12 and are limited to assigned scope and the sensitive-action restrictions stated there.

### 6.3 Profile & Settings permission evidence

RBAC Page matrix lists:
- SUPER_ADMIN: V/U
- FINANCE_TEAM: V/U
- HOD: V/U
- MANAGER: V/U
- STUDENT: V/U OWN

This confirms the page-level permission labels, but not which fields are editable, how email/credentials are verified, or what happens after a profile change.

### 6.4 User/session page map

| Role | Source-listed page | Source-listed entities | Evidence |
|---|---|---|---|
| SUPER_ADMIN | Super Admin Login | User, RefreshSession | Page-to-DB Mapping v2 |
| SUPER_ADMIN | User Management | User, College, Department, Section | Page-to-DB Mapping v2 |
| SUPER_ADMIN | Finance Team Management | College, User | Page-to-DB Mapping v2 |
| SUPER_ADMIN | Security / Sessions | RefreshSession, User | Page-to-DB Mapping v2 |
| FINANCE_TEAM | HOD Accounts | User, College, Department | Page-to-DB Mapping v2 |
| FINANCE_TEAM | Profile & Settings | User, RefreshSession, AuditLog | Page-to-DB Mapping v2 |
| HOD | Managers / Lab Staff | User, Department, Section, Group | Page-to-DB Mapping v2 |
| HOD | Students | User, Department, Section, CourseEnrollment, GroupMember | Page-to-DB Mapping v2 |
| HOD | Profile & Settings | User, RefreshSession, AuditLog | Page-to-DB Mapping v2 |
| MANAGER | Students | User, College, Department, Section, CourseEnrollment, GroupMember | Page-to-DB Mapping v2 |
| MANAGER | Profile & Settings | User, RefreshSession, AuditLog | Page-to-DB Mapping v2 |
| STUDENT | Profile & Settings | User, RefreshSession, AuditLog | Page-to-DB Mapping v2 |

The mapping establishes page/entity association only. The source does not fully define page routes, list columns, controls, field editability, or record-level actions.

### 6.5 Mandatory authorization principles

**Source-confirmed RBAC:**
1. Authenticate the request.
2. Identify role and required permission.
3. Apply college/department/section/group scope.
4. Apply ownership checks where relevant.
5. Only then perform the operation.
6. Record sensitive events where audit policy requires it.

- Frontend restrictions are not security enforcement.
- A valid role does not authorize access to all records.
- Student private records must use authenticated ownership, not a client-supplied student ID.
- Foreign-tenant or foreign-scope requests must be denied.
- Scope changes must not be accepted merely because the client submits a different `collegeId`, `departmentId`, or `sectionId`.

## 7. Module overview and navigation

### 7.1 Source-supported entry points
- Super Admin Login page.
- Super Admin User Management and Finance Team Management.
- Finance Team HOD Accounts.
- HOD Managers / Lab Staff and Students.
- Manager Students.
- Profile & Settings for Finance Team, HOD, Manager, Student.
- SUPER_ADMIN Security / Sessions.

### 7.2 Entry points not confirmed
- Whether all roles share one login page.
- Login URL and route names.
- Whether a user-management detail page, create-user modal, edit-user form, or user invitation page exists.
- Whether user session listing/revocation is available to end users or only SUPER_ADMIN.
- Exact post-login landing page for each role.

Do not specify visual styling or exact UI copy unless approved.

## 8. Functional inventory

| Function ID | Function | Primary actor | Entry point | Type | Requirement evidence | Specification status |
|---|---|---|---|---|---|---|
| AUTH-F001 | Sign in / establish identity | User; role-specific availability TBD | Super Admin Login; other login entry points TBD | Workflow | Page mapping + User/RefreshSession | Detailed draft; auth mechanics TBD |
| AUTH-F002 | Resolve authorization context and protect resources | System | Protected page/API request | System/security | RBAC | Source-confirmed principle; integration details TBD |
| AUTH-F003 | Sign out / end current session | User | UI action TBD | Workflow | RefreshSession schema; derived | Behavior decision required |
| AUTH-F004 | Refresh/renew session | System/user | Session renewal trigger TBD | System workflow | RefreshSession schema | Mechanics TBD |
| AUTH-F005 | View/list user records | SUPER_ADMIN, Finance, HOD, Manager, Student | Role-mapped User pages | Read | RBAC + page mapping | Detailed draft; permission conflict noted |
| AUTH-F006 | Create/provision user | SUPER_ADMIN, Finance/HOD conditional | User Management / Finance Team Management / Managers & Students | Create | RBAC + page mapping | Provisioning workflow TBD |
| AUTH-F007 | Update user record | SUPER_ADMIN, Finance/HOD/Manager conditional | User Management / mapped user pages | Update | RBAC | Detailed draft; Manager permission conflict |
| AUTH-F008 | Assign/change role and organization scope | Authorized administrator TBD | User Management / account admin | Update/security | User schema + RBAC | Decision required |
| AUTH-F009 | Change account status | Authorized administrator TBD | User Management | Lifecycle | EntityStatus + isActive schema | Decision required |
| AUTH-F010 | View/update own profile and settings | All mapped roles; Super Admin profile availability TBD | Profile & Settings | Read/Update | RBAC + page mapping | Field-level behavior TBD |
| AUTH-F011 | Credential setup, change, recovery, verification | User/admin TBD | Entry points not mapped | Workflow/security | Email/passwordHash schema only | Not source-specified; decision required |
| AUTH-F012 | MFA enrollment/challenge/recovery | User/admin TBD | Entry points not mapped | Security workflow | mfaEnabled/mfaSecret schema only | Not source-specified; decision required |
| AUTH-F013 | View/manage sessions | SUPER_ADMIN; other actors TBD | Security / Sessions; Profile & Settings entity mapping | Read/revoke | Page mapping + RefreshSession | Session UI/action policy TBD |
| AUTH-F014 | Audit authentication/account/security actions | System; audit readers per RBAC | System events; Audit Logs page | System/audit | AuditLog schema + RBAC audit guidance | Event catalog/retention TBD |
| AUTH-F015 | Impersonation session handling | Actor/target roles TBD | No source page specified | Security workflow | RefreshSession fields only | Feature existence/behavior TBD |
| AUTH-F016 | Delete user record | SUPER_ADMIN per CRUD matrix; other roles not granted delete in source matrix | User Management; exact action/control TBD | Delete/lifecycle | RBAC CRUD matrix; User schema | Delete permission is source-confirmed for SUPER_ADMIN; deletion semantics and retention are TBD (AUTH-D24) |

## 9. Detailed function specifications

> For functions whose workflow is absent from source documents, the details below define the questions and evidence needed for product review. They are not approved target behavior. Function-level QA summaries reference [QA-TEST-CASES.md](./QA-TEST-CASES.md), which includes the Standard-required preconditions/test data, steps, expected results, and decision readiness.

### AUTH-F001 — Sign in / establish identity

**Requirement status:** Source-confirmed page/entity mapping; login workflow TBD.  
**Specification status:** Detailed draft; decision required.  
**Priority:** TBD.  
**Primary actor:** User (role-specific availability TBD).  
**Supporting actor:** Authentication service/system.

#### 9.1 Purpose and outcome
Establish an authenticated user identity so protected DEZOLVER operations can apply role and scope checks. The source explicitly maps a Super Admin Login page backed by `User` and `RefreshSession`; approved target behavior extends this to one shared login experience for all roles.

#### 9.2 Permissions and data scope
- Login is an identity-establishment action, not a data-scope bypass.
- Successful login must resolve an existing user and role.
- Account status/access eligibility must be checked according to a future approved status policy (AUTH-D09/D10).
- Use the shared login entry point; role/membership context is resolved after authentication (AUTH-D01/D02/D11).

#### 9.3 Preconditions
- Login entry point is available.
- User has an identity record in the approved account model.
- Demo uses email/password; account creation follows the role-based creator matrix; email verification/recovery follows AUTH-D05.

#### 9.4 Entry point and trigger
- Source-confirmed: Super Admin Login page.
- Other role login entry points: TBD.
- Trigger: user submits login credentials/identity proof; exact form and protocol are TBD.

#### 9.5 UI behavior
Source only confirms a Super Admin Login page. Identifier field, password/SSO controls, MFA challenge, error copy, remember-me, recovery link, and redirect destination are TBD. Do not invent exact labels or layout.

#### 9.6 Inputs

| Input | Source | Type | Required? | Format/default | Validation | Notes |
|---|---|---|---|---|---|---|
| Login identifier | TBD | TBD | TBD | TBD | Must match approved identity method | Schema includes email; does not prove email is the login identifier. |
| Credential / identity proof | TBD | TBD | TBD | TBD | Approved authentication protocol | Schema includes passwordHash; authentication mechanism is not specified. |
| MFA challenge | Schema fields only | TBD | Conditional if MFA approved | TBD | MFA decision/policy | AUTH-D07. |

#### 9.7 Main flow
1. User opens an approved login entry point.
2. UI displays the approved identity/credential controls.
3. User submits identity proof.
4. System locates the account using the approved identity method.
5. System verifies the credential using the approved authentication method.
6. System evaluates account status using the approved account-state policy.
7. If MFA is required, system performs the approved challenge flow.
8. System establishes authenticated identity and role/scope context.
9. System creates/updates session data according to approved session policy.
10. UI navigates to the approved post-login destination.

Steps 4–10 describe the required decision points, not an already source-confirmed implementation.

#### 9.8 Business rules
- AUTH-BR-001: Protected operations require authentication (derived from RBAC security flow).
- AUTH-BR-002: Role and scope are resolved from trusted authenticated account context, not arbitrary client values (source-confirmed RBAC).
- AUTH-BR-003: Login eligibility by status is TBD (AUTH-D09/D10).
- AUTH-BR-004: Authentication method, email uniqueness/case policy, MFA policy, and lockout behavior are TBD.

#### 9.9 Validation and feedback

| Validation ID | Condition | Validation point | Expected behavior | Feedback |
|---|---|---|---|---|
| AUTH-V001 | Missing required identifier/credential | Submit; requiredness TBD | Reject request without creating session | Message TBD |
| AUTH-V002 | Invalid identity proof | Server | Reject authentication; no authenticated session | Generic error wording TBD |
| AUTH-V003 | Account status disallows login | Server | Reject per approved status policy | Feedback TBD |
| AUTH-V004 | Required MFA challenge fails | Server | Reject or retry per approved MFA policy | TBD |

#### 9.10 System/backend behavior
- Validate credentials using the approved auth mechanism.
- Do not expose passwordHash, MFA secret, tokenHash, or sensitive session data in client responses.
- Resolve role and scope from server-controlled account data.
- Session issuance, token format, rotation, and storage are not specified; see AUTH-D08.

#### 9.11 Data/state changes
- User: read identity, role, status, isActive, and scope fields.
- RefreshSession: schema includes session record fields; exact creation/issuance timing is TBD.
- AuditLog: login audit event requirement TBD (AUTH-D15).

#### 9.12 Success result
Authenticated context established and user redirected according to approved role routing. Exact response, token/session behavior, and destination TBD.

#### 9.13 Alternate/failure handling
- Invalid credentials: deny; no authenticated access.
- Unknown account: deny without revealing account existence (security recommendation; pending review).
- Suspended/deactivated/archived account: behavior depends on AUTH-D09.
- MFA failure: policy TBD.
- Authentication provider/network failure: shared Module 18 handling.
- Duplicate/replayed requests: session idempotency behavior TBD.

#### 9.14 Edge cases
- Role/scope updated during authentication.
- User record has missing college/department/section assignment.
- FINANCE_TEAM is not represented in the schema Role enum.
- Multiple user records share an identity field (uniqueness TBD).

#### 9.15 Audit/privacy
Audit event and success/failure logging are TBD under AUTH-D15 / Module 15. Never log raw credentials, passwordHash, MFA secret, or bearer/refresh token values.

#### 9.16 Acceptance criteria
- Login is available only through approved entry points.
- Invalid identity proof never establishes an authenticated session.
- Role/scope comes from trusted account data.
- Account status and MFA behavior follow approved policies.
- Sensitive credential/session values are not returned or logged.
- Post-login destination follows approved routing.

#### 9.17 QA cases
| Test ID | Scenario | Preconditions/data | Steps | Expected result | Trace |
|---|---|---|---|---|---|
| AUTH-QA-001 | Valid login | Active test account; approved auth method | Submit valid identity proof | Authenticated context established | AUTH-D03/D09 |
| AUTH-QA-002 | Invalid credential | Existing account | Submit invalid credential | Denied; no session | AUTH-D03 |
| AUTH-QA-003 | Account blocked by status | Status policy approved | Attempt login | Result follows approved status matrix | AUTH-D09 |
| AUTH-QA-004 | Role/scope spoof attempt | Client sends forged role/college/department | Attempt login/request | Server identity/scope unaffected | AUTH-BR-002 |
| AUTH-QA-005 | MFA required but invalid | MFA enabled/policy approved | Submit invalid challenge | Denied per MFA policy | AUTH-D07 |

#### 9.18 Unresolved items
AUTH-D01–D09, D17; login route, authentication method, provisioning, status policy, MFA, and routing.

---

### AUTH-F002 — Resolve authorization context and protect resources

**Requirement status:** Source-confirmed RBAC principle; system behavior derived.  
**Specification status:** Detailed draft.  
**Priority:** Must-have security principle (source RBAC).  
**Primary actor:** System/backend on behalf of authenticated actor.

**Supporting/system actor(s):** Requesting user/client, authentication context provider, protected backend/resource service.

#### Purpose and outcome
Ensure every protected operation checks identity, role permission, organizational scope, and ownership before accessing or changing data.

#### Permissions and scope
- Applies to all protected pages/APIs.
- SUPER_ADMIN: global.
- FINANCE_TEAM: assigned College.
- HOD: College + Department.
- MANAGER: assigned College + Department + permitted Section/Group.
- STUDENT: own records + permitted academic resources.

#### Preconditions
Request is received; authentication context is available or the request is rejected.

#### Entry/trigger
Every protected page/API request; this is not a user-facing navigation item.

#### Inputs
| Input | Source | Type | Required | Validation |
|---|---|---|---|---|
| Authenticated user identity | Auth system | ID/context | Yes | Trusted server-side context |
| Role | User record/auth context | Role | Yes | Must be valid role |
| Resource/action | Request | Action/resource ID | Yes | Permission lookup |
| Scope/ownership attributes | Trusted resource and user data | IDs | Conditional | Must match permitted scope/owner |

#### Main flow
1. Receive protected request.
2. Validate authenticated identity.
3. Resolve role and required action permission.
4. Determine resource scope from trusted data.
5. Apply college/department/section/group filters.
6. Apply ownership check where applicable.
7. Permit operation only if all checks pass.
8. Return authorized result; record audit/security event where required.

#### Business rules
- AUTH-BR-005: role permission and data scope are both required (RBAC).
- AUTH-BR-006: frontend hiding is not a security control (RBAC).
- AUTH-BR-007: Student-owned records use authenticated student identity, not request-supplied studentId (RBAC).
- AUTH-BR-008: cross-college, cross-department, cross-section, or unauthorized group access is denied (RBAC).
- AUTH-BR-009: authorization failures must not execute the protected database operation (RBAC).

#### Validation/feedback
| ID | Condition | Behavior | Feedback |
|---|---|---|---|
| AUTH-V005 | Missing/invalid auth context | Deny request | Shared auth error TBD |
| AUTH-V006 | Role lacks permission | Deny | Shared access-denied behavior |
| AUTH-V007 | Resource outside scope | Deny | Do not disclose restricted data |
| AUTH-V008 | Student ownership mismatch | Deny | Do not disclose record existence |

#### System/data behavior
No protected data mutation occurs until checks pass. Apply scope filters to read/write operations; do not rely solely on fetching a record by ID then trusting the request. Detailed API contracts are out of scope.

#### Success/failure/edge
- Success: operation continues with permitted resource.
- Failure: deny; no protected mutation/query result returned.
- Edge: resource changes scope between check and operation; concurrency/transaction handling is Module 18/technical design.
- A valid role requesting a record outside scope must still be denied.

#### Audit/privacy
RBAC says failed security-sensitive operations may be recorded where required. Canonical events/retention belong to Module 15. Do not expose sensitive record existence through error messages.

#### Acceptance criteria
- All protected operations apply role, scope, and ownership.
- Client-supplied role/tenant/scope IDs cannot expand permissions.
- A denial prevents protected data mutation.
- Student private data is owner-checked.
- Frontend controls are not the sole authorization barrier.

#### QA
| Test ID | Scenario | Expected result | Trace |
|---|---|---|---|
| AUTH-QA-006 | User with valid role requests permitted resource | Allow | RBAC |
| AUTH-QA-007 | College A actor requests College B record | Deny | RBAC tenant isolation |
| AUTH-QA-008 | HOD requests foreign department record | Deny | RBAC scope |
| AUTH-QA-009 | Manager requests unassigned section | Deny | RBAC scope |
| AUTH-QA-010 | Student requests another student's submission/profile | Deny | RBAC ownership |
| AUTH-QA-011 | Client forges role or collegeId | No scope expansion | RBAC backend enforcement |

#### Unresolved items
AUTH-D01, D12; role/permission conflict for Manager User Management; shared denial behavior Module 18.


#### Standard-template clarifications
- **UI behavior:** Not applicable as a standalone UI; this is a backend/system authorization function. Protected pages/APIs must rely on the same server-side decision.
- **Data/state:** Reads trusted identity, role/permission, resource scope, and ownership data; does not mutate the target resource when authorization fails.
- **Success result:** The downstream operation proceeds only after all required checks pass.
- **Alternate/failure:** Missing identity, insufficient role permission, out-of-scope resource, ownership mismatch, or stale scope all deny access; exact shared error format is Module 18.
- **Edge cases:** Include nested group/section relationships, changed organization assignments, and concurrent scope changes.

---

---

### AUTH-F003 — Sign out / end current session

**Requirement status:** Derived from session lifecycle need and RefreshSession schema; UI not source-mapped.  
**Specification status:** Decision required.  
**Primary actor:** Authenticated user.

**Supporting/system actor(s):** Authentication/session subsystem.
**Priority:** TBD (not agreed).

#### Purpose/outcome
End the current authenticated session according to an approved logout policy.

#### Permissions/scope
User may request logout for their own current session. Admin session termination is covered by AUTH-F013. This is a recommendation pending confirmation.

#### Preconditions/entry
Authenticated session exists; logout entry point and UI location TBD.

#### Inputs
| Input | Source | Type | Required | Validation |
|---|---|---|---|---|
| Current session identifier/token | Auth/session context | Opaque identifier | Yes | Must identify current session |
| User identity | Auth context | ID | Yes | Server-derived |

#### Main flow
1. User invokes logout.
2. System identifies current session.
3. System applies approved revocation/invalidating behavior.
4. Client clears local session material according to approved design.
5. User is redirected to approved public/login screen.

#### Rules/data changes
- RefreshSession contains `isRevoked`, `expiresAt`, and tokenHash.
- Whether logout sets `isRevoked=true`, removes session, or invalidates another mechanism is TBD (AUTH-D08).
- Logout must not terminate unrelated user sessions unless “logout all” is explicitly chosen.

#### Validation/failure
Invalid/stale session; repeated logout; backend unavailable; session already revoked. Idempotency and feedback are TBD.

#### Audit/privacy
Logout event logging TBD. Never log token values.

#### Acceptance criteria
Logout ends the current session under the approved policy; stale session cannot continue protected access; unrelated sessions remain unaffected unless explicitly requested.

#### QA
| Test ID | Scenario | Expected result | Trace |
|---|---|---|---|
| AUTH-QA-012 | Logout current session | Session ends per approved policy | AUTH-D08 |
| AUTH-QA-013 | Repeat logout | Safe/idempotent behavior | AUTH-D08 |
| AUTH-QA-014 | Attempt protected request after logout | Denied | AUTH-F002 |
| AUTH-QA-015 | Logout one device/session | Other sessions unaffected unless policy says otherwise | AUTH-D08 |

#### Unresolved
AUTH-D08, AUTH-D15; UI placement and current-session representation.


#### Standard-template clarifications
- **UI behavior:** Logout action, placement, confirmation, and destination are not mapped by source and remain TBD.
- **System/backend:** Identify current session from trusted auth context; apply the approved current-session revocation/invalidating mechanism; do not trust client-supplied session/user IDs.
- **Data/state:** RefreshSession update/delete behavior is AUTH-D08; no other session may be affected by default.
- **Success result:** Current protected access ends and the user reaches the approved public/login state.
- **Alternate/failure:** Already expired/revoked or repeated logout should not re-enable access; backend outage/retry policy is Module 18.
- **Edge cases:** Logout from multiple tabs/devices; logout after role/status change; current session is also an admin session.

---

---

### AUTH-F004 — Refresh/renew session

**Requirement status:** RefreshSession schema source-confirmed; renewal workflow not specified.  
**Specification status:** Decision required.  
**Primary actor:** System/user.

**Supporting/system actor(s):** Authentication/session service.
**Priority:** TBD (not agreed).

#### Purpose/outcome
Maintain or renew an authenticated session according to approved lifetime and refresh policy.

#### Preconditions
A valid session/refresh mechanism exists; session has not expired or been revoked.

#### Entry/trigger
Automatic renewal or user-triggered renewal; trigger is TBD.

#### Inputs
| Input | Source | Type | Required | Validation |
|---|---|---|---|---|
| Refresh credential/token | Auth mechanism | Secret/opaque | TBD | Approved token validation |
| RefreshSession ID | Session context | ID | TBD | Must be current and not revoked |
| Current time | System | Timestamp | Yes | Compare with expiresAt |

#### Main flow
1. Renewal trigger occurs.
2. System validates refresh credential/session.
3. System checks expiry and revocation.
4. System applies approved rotation/reuse policy.
5. System returns/establishes renewed session state or denies renewal.

#### Rules/data changes
RefreshSession fields include tokenHash, expiresAt, isRevoked, createdAt, updatedAt. Rotation, expiry duration, session maximum age, and whether record is updated or replaced are TBD.

#### Failure/alternate paths
Expired, revoked, invalid, reused, or compromised token; account status changed; role/scope changed; service failure. Expected behavior is linked to AUTH-D08/D13 and Module 18.

#### Audit/privacy
Security events for invalid/reused refresh credentials are TBD. Never log token values or tokenHash.

#### Acceptance criteria
A revoked/expired session cannot be renewed; renewal follows the approved token lifecycle; no credential material is exposed.

#### QA
| Test ID | Scenario | Expected result | Trace |
|---|---|---|---|
| AUTH-QA-016 | Valid refresh | Renew according to policy | AUTH-D08 |
| AUTH-QA-017 | Expired session | Renewal denied | AUTH-D08 |
| AUTH-QA-018 | Revoked session | Renewal denied | AUTH-D08 |
| AUTH-QA-019 | Role/scope changed before refresh | Applies approved invalidation/update policy | AUTH-D13 |

#### Unresolved
AUTH-D08, D09/D10, D13.


#### Standard-template clarifications
- **Permissions/scope:** Session renewal is only for the authenticated user's own session; administrative session changes are AUTH-F013. Exact protocol remains TBD.
- **Entry/UI:** Background refresh vs explicit user action and any loading/reauthentication UI are not source-mapped.
- **System/data:** Check expiration/revocation and account/role/scope state before renewing. Whether RefreshSession is rotated, updated, or replaced is AUTH-D08.
- **Success result:** New or continued auth state is established only according to the approved session policy.
- **Alternate/failure:** Expired/revoked/invalid/replayed token, changed account status, changed role/scope, or service failure; no refresh credential should be returned to logs or unrelated clients.
- **Edge cases:** simultaneous refresh requests, token reuse, and refresh after account status/role change.

---

---

### AUTH-F005 — View/list user records

**Requirement status:** Page mapping and RBAC source-confirmed.  
**Specification status:** Detailed draft; permission conflict unresolved.  
**Primary actors:** SUPER_ADMIN, FINANCE_TEAM, HOD, MANAGER, STUDENT (own record).

**Supporting/system actor(s):** User Management service; organization/scope data provider.
**Priority:** TBD (not agreed).

#### Purpose/outcome
Allow each role to view user records only within its authorized function and scope.

#### Entry points
SUPER_ADMIN User Management; Finance HOD Accounts; HOD Managers/Lab Staff and Students; Manager Students; Student/role Profile & Settings (own profile only).

#### Inputs
| Input | Source | Type | Required | Validation |
|---|---|---|---|---|
| Search/filter values | UI TBD | Text/filters | Optional | Validate allowed filters |
| User ID | Page/route | ID | For detail | Scope/ownership check |
| College/department/section filter | Page mapping/RBAC scope | IDs | Optional | Must not expand authenticated scope |

#### Main flow
1. Actor opens mapped user page.
2. System resolves role and permission.
3. System applies scope/ownership filters.
4. System returns only authorized user fields/records.
5. UI displays permitted records.

#### Business rules
- SUPER_ADMIN: global User Management view.
- FINANCE_TEAM: college-scoped User/HOD Accounts view per RBAC.
- HOD: assigned department/college user view.
- MANAGER: assigned section/group scope; User Management action discrepancy AUTH-D12.
- STUDENT: own profile only.
- List columns, filters, sorting, pagination, and sensitive field visibility are TBD.
- PasswordHash, MFA secret, and session token data must never be exposed.

#### Validation/system/data
Reject/ignore forged scope filters; data queries must be scoped server-side. Read-only operation.

#### Failure/edge
No users; foreign user ID; missing organizational assignment; role changed; record moved out of scope; Finance Team account mapping conflict.

#### Audit/privacy
Read audit policy TBD; restrict personal information to necessary fields.

#### Acceptance criteria
Role/scope determines returned user records; student sees own user only; sensitive credential fields are never returned.

#### QA
| Test ID | Scenario | Expected result | Trace |
|---|---|---|---|
| AUTH-QA-020 | SUPER_ADMIN lists users | Global records allowed | RBAC |
| AUTH-QA-021 | HOD lists department users | Only assigned scope | RBAC |
| AUTH-QA-022 | Manager lists users | Only permitted section; action rights pending D12 | RBAC |
| AUTH-QA-023 | Student requests another user | Denied | RBAC ownership |
| AUTH-QA-024 | API response contains passwordHash/mfaSecret | Must not expose | Schema privacy |

#### Unresolved
AUTH-D01, D11, D12, D16; field visibility and list behavior.


#### Standard-template clarifications
- **Permissions/scope:** Apply the role matrix in Sections 6.2–6.3 and trusted organization scope; student list/detail access is own-record only unless another source-approved academic function grants access.
- **Preconditions:** Actor authenticated; mapped page available; list/detail permission and scope are valid.
- **UI behavior:** Exact columns, filters, pagination, sorting, empty/loading states, and detail route are not specified by source; mark each TBD rather than designing from entity names.
- **System/data:** Read only authorized User fields; apply scope in the query and again for detail access; omit credential/security secrets.
- **Success/failure:** Return only scoped results; empty result is not an authorization failure; foreign ID/invalid scope is denied without revealing restricted data.
- **Edge cases:** Search/filter tampering, moved user, missing role/scope assignment, and duplicate/ambiguous identity.

---

---

### AUTH-F006 — Create/provision user

**Requirement status:** RBAC/page mapping supports user-management create permissions for selected roles; actual provisioning flow is not specified.  
**Specification status:** Decision required.  
**Primary actors:** SUPER_ADMIN; Finance/HOD conditional by matrix.  
**Supporting actors:** College onboarding/Module 2/3, Notifications Module 13.

**Priority:** TBD (not agreed).

#### Purpose/outcome
Create or provision a user account in the correct role and organizational scope.

#### Entry points
SUPER_ADMIN User Management / Finance Team Management; Finance Team HOD Accounts; HOD Managers/Lab Staff and Students. Exact creation controls are not mapped.

#### Preconditions
Actor has approved create permission; target college/department/section exists and is in actor scope; account provisioning policy approved.

#### Inputs
| Field | Schema/source | Type | Required? | Validation |
|---|---|---|---|---|
| email | User.email varchar(255) | Email/text | TBD | Format/uniqueness/case rule TBD |
| fullName | User.fullName varchar(255) | Text | TBD | Length/requiredness TBD |
| role | User.role enum | Role | TBD | Allowed role per actor; FINANCE_TEAM enum discrepancy |
| status | User.status EntityStatus | Enum | TBD | Initial status policy TBD |
| isActive | User.isActive boolean | Boolean | TBD | Initial value/precedence TBD |
| collegeId | User.collegeId | ID | Role-dependent | Must be actor-authorized |
| departmentId | User.departmentId | ID | Role-dependent | Must be in college scope |
| sectionId | User.sectionId | ID | Role-dependent | Must be in department scope |
| academicBatch | User.academicBatch | Text | Optional/TBD | Rules TBD |
| passwordHash | Schema field | Credential hash | System only | Must not be supplied as plaintext DB field by user/admin |
| MFA fields | Schema | Boolean/secret | TBD | AUTH-D07 |

#### Main flow
1. Authorized actor opens user provisioning flow.
2. UI presents fields approved for that role.
3. Actor submits data.
4. System validates role, scope, and required attributes.
5. System checks identity uniqueness under approved rule.
6. System creates account using approved credential/invitation process.
7. System records organization assignments and initial account status.
8. System displays outcome and sends any approved notification via Module 13.

#### Business rules
- Role assignment is constrained by actor's permission and scope.
- SUPER_ADMIN global; HOD restricted to assigned college/department.
- Finance/HOD/Manager create permissions follow RBAC and the Manager discrepancy.
- Finance Team role must be reconciled with schema enum (AUTH-D01).
- Whether onboarding auto-creates Finance Team/HOD users is not stated in Business Flow v2.
- No source-defined password, invite, verification, seat, or uniqueness rule.

#### Validation
| ID | Condition | Expected behavior | Feedback |
|---|---|---|---|
| AUTH-V009 | Role not assignable by actor | Reject | Message TBD |
| AUTH-V010 | College/department/section outside scope | Reject | Message TBD |
| AUTH-V011 | Duplicate identity/email | Behavior depends on uniqueness decision | AUTH-D17 |
| AUTH-V012 | Required field absent/invalid | Reject | Field message TBD |
| AUTH-V013 | FINANCE_TEAM role selected | Reject/allow only after enum/model decision | AUTH-D01 |

#### System/data
Create User with system-generated ID/timestamps; assign role/scope only after validation. Credential/invitation creation mechanism is TBD. Do not create a user partially if provisioning fails; transaction/compensation behavior needs technical design.

#### Success/failure
Success: account created and presented according to approved initial state. Failure: no unintended partial account or notification. Duplicate/retry idempotency TBD.

#### Edge cases
Concurrent creation; onboarding rollback; subscription/seat restrictions (Module 3); department/section changes during creation; role enum conflict.

#### Audit/privacy
RBAC lists user creation as sensitive audit candidate. Audit details and notifications are Module 15/13. Never log plaintext password, invite secret, MFA secret.

#### Acceptance criteria
Only authorized roles create accounts; scope cannot be spoofed; role and initial state follow approved rules; account creation respects identity uniqueness; credential material is protected.

#### QA
| Test ID | Scenario | Expected result | Trace |
|---|---|---|---|
| AUTH-QA-025 | SUPER_ADMIN creates user | Created within approved schema/role | RBAC |
| AUTH-QA-026 | HOD creates department-scoped user | Scope validated | RBAC |
| AUTH-QA-027 | Manager attempts user create | Denied unless approved matrix permits | AUTH-D12 |
| AUTH-QA-028 | Create Finance Team user | Role model follows resolved enum decision | AUTH-D01 |
| AUTH-QA-029 | Duplicate email/identity | Result follows AUTH-D17 | AUTH-D17 |
| AUTH-QA-030 | User create with foreign departmentId | Denied/no record | RBAC |

#### Unresolved
AUTH-D01, D04, D05, D11, D12, D17; invitation and account activation.


#### Standard-template clarifications
- **Permissions/scope:** Use the platform and page/API matrices; SUPER_ADMIN is global, Finance Team is college-scoped, HOD is college+department, and Manager create permission is unresolved. Student cannot provision other users.
- **UI behavior:** Create form fields, role selectors, invite vs direct create, validation timing, and cancellation behavior are TBD.
- **System/data:** User creation must be all-or-nothing from the functional perspective; create only approved fields and do not persist raw credential material.
- **Success result:** A user record is created in the approved initial state; any invite/verification notification is conditional on Module 13 approval.
- **Alternate/failure:** Invalid scope/role, duplicate identity, missing required fields, seat/eligibility failure, notification failure, and concurrent create are decision/Module 3/13/18 gated.

---

---

### AUTH-F007 — Update user record

#### Preconditions
- Actor is authenticated.
- Target User exists and is within actor scope.
- User Update permission and field-level allow-list are approved; Manager rights remain blocked by AUTH-D12 until resolved.

**Requirement status:** RBAC permission matrix and user page mapping source-confirmed; editable fields not defined.  
**Specification status:** Decision required.  
**Primary actors:** SUPER_ADMIN, HOD; Finance/Manager conditional; Student own profile is AUTH-F010.

**Supporting/system actor(s):** User Management service; organization/scope provider; Module 15 audit subsystem.
**Priority:** TBD (not agreed).

#### Purpose/outcome
Update authorized user data without allowing role/tenant/scope escalation through ordinary profile editing.

#### Entry points
User Management, HOD Managers/Lab Staff, Students, Manager Students, Finance HOD Accounts. Exact edit actions are TBD.

#### Inputs
User ID plus approved editable fields. Field-level edit matrix is TBD; possible schema fields include fullName, email, role, status, isActive, collegeId, departmentId, sectionId, academicBatch.

#### Main flow
1. Actor opens permitted user record.
2. Backend verifies role, action permission, scope, and target user.
3. UI displays only fields actor may edit.
4. Actor submits changes.
5. Backend rechecks permission/scope and validates each field.
6. System updates permitted fields and updatedAt.
7. UI displays updated data.

#### Business rules
- User updates must not change role/organization scope unless actor has separate assignment permission (AUTH-F008).
- Student own profile update follows AUTH-F010, not admin User Management.
- Manager User Management Update conflict remains AUTH-D12.
- Email change verification/immutability is TBD.
- Avoid updating credential/MFA secrets through generic profile forms.

#### Validation/system/data
Validate field formats and role-specific constraints; apply server-side scope; prevent mass assignment of protected fields. Update only approved fields; concurrency behavior TBD.

#### Failure/edge
Foreign target, stale edit, duplicate email, concurrent update, scope transfer, inactive target.

#### Audit/privacy
RBAC identifies user updates as audit candidates. Record approved changed-field metadata without logging secrets.

#### Acceptance criteria
Unauthorized field changes rejected; user cannot elevate role or change tenant via ordinary profile edit; updates remain scoped; audit event behavior follows Module 15.

#### QA
| Test ID | Scenario | Expected result | Trace |
|---|---|---|---|
| AUTH-QA-031 | HOD updates user in department | Allowed fields updated | RBAC |
| AUTH-QA-032 | HOD updates user outside department | Denied | RBAC |
| AUTH-QA-033 | User submits role/collegeId in profile update | Protected field not changed | AUTH-D12 |
| AUTH-QA-034 | Duplicate email update | Follows identity uniqueness rule | AUTH-D17 |
| AUTH-QA-035 | Manager edits user | Allow/deny follows resolved D12 | AUTH-D12 |

#### Unresolved
AUTH-D12, D16, D17; field-level edit matrix, concurrency.


#### Standard-template clarifications
- **Permissions/scope:** Enforce User Management V/C/U matrix; Manager Update conflict remains AUTH-D12. Self-profile updates are AUTH-F010.
- **UI behavior:** Editable fields, save/cancel behavior, change confirmation, and field visibility are TBD pending AUTH-D23.
- **System/data:** Apply field allow-list server-side; recheck target scope at save time; only update permitted fields and system timestamps.
- **Success result:** Approved fields persist and the displayed record reflects the saved version.
- **Alternate/failure:** Stale/concurrent edit, target moved out of scope, duplicate email, unsupported/protected field, and validation failure leave unauthorized fields unchanged.
- **Edge cases:** Bulk edit/mass assignment, role/scope changes submitted alongside ordinary profile edits, and update after user deactivation.

---

---

### AUTH-F008 — Assign/change role and organization scope

**Requirement status:** User schema/RBAC scope source-confirmed; workflow and authority TBD.  
**Specification status:** Decision required.  
**Primary actor:** Authorized administrator TBD.

**Supporting/system actor(s):** User Management service; Module 2 organization hierarchy; Module 15 audit subsystem.
**Priority:** TBD (not agreed).

#### Purpose/outcome
Assign or change a user's role and college/department/section scope in a controlled administrative operation.

#### Preconditions
Actor has explicit assignment permission; target user and target organizational entities exist; Module 2 scope hierarchy is valid.

#### Entry point
User Management/account administration; exact route/control TBD.

#### Inputs
| Field | Source | Required | Validation |
|---|---|---|---|
| targetUserId | User.id | Yes | Target is in actor scope |
| role | User.role | Yes for role change | Approved role list; Finance Team enum decision |
| collegeId | User.collegeId | Role-dependent | Actor can assign target college |
| departmentId | User.departmentId | Role-dependent | Department belongs to college |
| sectionId | User.sectionId | Role-dependent | Section belongs to department |
| academicBatch | User.academicBatch | TBD | Batch eligibility policy TBD |

#### Main flow
1. Admin opens assignment workflow.
2. System checks assignment authority.
3. UI offers only roles/scopes the actor can assign.
4. Admin selects new role/scope.
5. Backend validates organizational hierarchy and actor scope.
6. System updates user assignment.
7. System applies approved session invalidation/effective-time policy.
8. Audit event is recorded according to Module 15.

#### Business rules
- Assignment cannot exceed actor's authority.
- HOD assignment is limited to their department/college.
- Manager cannot assign across permitted section scope unless approved.
- Student cannot assign role/scope.
- Role/scope change impact on existing sessions and historical records is TBD (AUTH-D13/D18).

#### Validation/failure
Invalid role; role not in enum; department/college mismatch; section/department mismatch; actor unauthorized; target record stale. No partial assignment.

#### Data/state
Update User.role and applicable collegeId/departmentId/sectionId/academicBatch. Schema does not support multi-role/multi-org memberships explicitly.

#### Acceptance criteria
Assignment respects scope hierarchy; unauthorized actors cannot elevate roles or transfer users; session effect follows approved policy; historical user-owned records remain attributable.

#### QA
| Test ID | Scenario | Expected result | Trace |
|---|---|---|---|
| AUTH-QA-036 | SUPER_ADMIN assigns valid role | Allowed after role model approved | AUTH-D01 |
| AUTH-QA-037 | HOD assigns user to own department | Allowed if approved | RBAC |
| AUTH-QA-038 | HOD assigns user to foreign department | Denied | RBAC |
| AUTH-QA-039 | Manager attempts role escalation | Denied | RBAC |
| AUTH-QA-040 | Change role while session active | Follows approved invalidation policy | AUTH-D13 |

#### Unresolved
AUTH-D01, D11, D12, D13, D18; authority matrix and assignment audit.


#### Standard-template clarifications
- **Permissions/scope:** Assignment permission must be separate from ordinary User Update; source does not specify who may assign every role/scope.
- **UI behavior:** Role/scope selectors and allowable choices are TBD; UI must not offer unauthorized options once policy is approved.
- **System/backend:** Validate role against canonical enum, college/department/section hierarchy, and actor's assignment authority.
- **Success result:** User role/scope is changed only after approved validation; session and historical-data effects follow AUTH-D13/D18.
- **Alternate/failure:** Invalid hierarchy, missing target entity, stale assignment, role enum conflict, or privilege escalation attempt must not partially update User.
- **Audit/privacy:** Role/scope changes are sensitive audit candidates; Module 15 defines exact event and metadata.

---

---

### AUTH-F009 — Change account status

#### Preconditions
- Actor is authenticated and has approved status-change authority for the target.
- Target User exists.
- The requested transition is approved in AUTH-D09/D10 and the canonical Module 17 state machine.

**Requirement status:** Status enum and `isActive` field source-confirmed; lifecycle behavior TBD.  
**Specification status:** Decision required.  
**Primary actor:** Authorized administrator TBD.

**Supporting/system actor(s):** User Management service; session service; Module 17 state machine; Module 15 audit subsystem.
**Priority:** TBD (not agreed).

#### Purpose/outcome
Apply a defined account lifecycle status change while preserving appropriate records and access controls.

#### Source data
`User.status`: ACTIVE, SUSPENDED, DEACTIVATED, ARCHIVED.  
`User.isActive`: boolean; meaning and precedence unresolved.

#### Entry point
User Management; exact status action/control TBD.

#### Inputs
User ID, requested status, optional reason (reason field not present in schema; capture/log support TBD).

#### Main flow
1. Authorized actor opens account status action.
2. System shows current status and approved transitions.
3. Actor selects target status and confirms if required.
4. Backend validates actor permission and transition.
5. System updates canonical status field (must be decided).
6. System applies session invalidation and downstream access policy.
7. Audit event recorded per Module 15.

#### Business rules
- Do not infer login/access behavior from status labels.
- Canonical status vs `isActive` precedence must be approved (AUTH-D10).
- Allowed transitions, restoration, archiving, and historical record behavior are TBD.
- Account status must not delete academic/financial/audit history without an explicit retention policy.

#### Validation/system/data
Reject unknown status and unauthorized transitions. Data update and session effects are TBD pending state machine Module 17.

#### Success/failure/edge
Repeated same-status action; concurrent status change; active sessions; archived user restoration; user owns historical submissions/certificates.

#### Acceptance criteria
Only approved status transitions are available; account access follows approved state matrix; session effects are consistent; history remains retained per approved policy.

#### QA
| Test ID | Scenario | Expected result | Trace |
|---|---|---|---|
| AUTH-QA-041 | Change ACTIVE to SUSPENDED | Follows approved transition matrix | AUTH-D09 |
| AUTH-QA-042 | Suspended account requests login | Follows approved login policy | AUTH-D09 |
| AUTH-QA-043 | Status changed but isActive inconsistent | System follows canonical precedence | AUTH-D10 |
| AUTH-QA-044 | Archive user with historical data | Retention policy preserved | AUTH-D18 |

#### Unresolved
AUTH-D09, D10, D13, D18; transition matrix Module 17.


#### Standard-template clarifications
- **Permissions/scope:** Account-status authority is not fully defined by RBAC; AUTH-D23/AUTH-D09 must identify allowed actors and scope.
- **UI behavior:** Status controls, confirmation, reason capture, and restoration UI are TBD.
- **System/data:** Validate transition against Module 17 canonical state machine; apply the canonical `status`/`isActive` policy only after AUTH-D10.
- **Success result:** User status and session/access effects reflect the approved transition.
- **Alternate/failure:** Invalid transition, already-in-target-state, concurrent status change, or missing reason (if required) does not produce partial state.
- **Edge cases:** Last admin, own account, active sessions, and status change during login/refresh.

---

---

### AUTH-F010 — View/update own profile and settings

#### Preconditions
- User is authenticated and the account is available under the approved status policy.
- Self-service field permissions and profile settings have been approved.

**Requirement status:** Profile & Settings page mapping and RBAC permission labels source-confirmed; field rules TBD.  
**Specification status:** Detailed draft; decision required.  
**Primary actor:** FINANCE_TEAM, HOD, MANAGER, STUDENT; SUPER_ADMIN availability TBD.

**Supporting/system actor(s):** Profile service; authentication/authorization subsystem; Module 15 audit subsystem.
**Priority:** TBD (not agreed).

#### Purpose/outcome
Allow a user to view and update only the profile/settings fields approved for self-service.

#### Entry point
Profile & Settings page listed for Finance Team, HOD, Manager, Student. SUPER_ADMIN profile page is not explicitly mapped.

#### Inputs
User identity from authenticated context; candidate profile fields include email, fullName, academicBatch, and other settings—but editable fields are not source-defined.

#### Main flow
1. User opens Profile & Settings.
2. System loads own user profile and permitted settings.
3. UI displays editable/read-only fields per role policy.
4. User submits changes.
5. Backend verifies ownership and validates allowed fields.
6. System saves approved fields and displays updated profile.

#### Business rules
- Student profile data is owner-scoped.
- Profile editing does not grant role or organization-scope editing.
- Email change, verification, password, MFA, and session settings are handled by separate approved workflows.
- Profile page mapping references User, RefreshSession, AuditLog but does not prove all three are displayed.

#### Validation/data
Field format, requiredness, uniqueness, editability, and confirmation are TBD. No cross-user updates. No role/scope fields editable through self-service unless separately approved.

#### Failure/edge
Stale session; account changed by admin; duplicate email; unsupported field; concurrent update.

#### Audit/privacy
Profile update audit is a candidate event under RBAC; final event catalog Module 15. Do not expose credentials or private session data.

#### Acceptance criteria
User can view own permitted fields; cannot view/edit another user; role/scope cannot be changed through self-service; fields follow approved edit policy.

#### QA
| Test ID | Scenario | Expected result | Trace |
|---|---|---|---|
| AUTH-QA-045 | Student views own profile | Own permitted fields | RBAC |
| AUTH-QA-046 | Student requests another user profile | Denied | RBAC ownership |
| AUTH-QA-047 | Profile update changes role/department | Rejected/ignored unless separately authorized | AUTH-D16 |
| AUTH-QA-048 | Email update | Follows verification/uniqueness decision | AUTH-D05/D17 |

#### Unresolved
AUTH-D16, D17; role-specific profile fields and Super Admin profile access.


#### Standard-template clarifications
- **Permissions/scope:** RBAC Profile & Settings is V/U for SUPER_ADMIN, FINANCE_TEAM, HOD, MANAGER; Student V/U OWN. Page mapping does not explicitly list a Super Admin Profile & Settings screen.
- **UI behavior:** Profile fields, settings controls, edit/save/cancel, and exact messages are TBD.
- **System/data:** Use authenticated user identity for self-service; ignore/reject client-supplied target user IDs and protected role/scope fields.
- **Success result:** Only approved self-editable fields change.
- **Alternate/failure:** Email conflict/verification, stale session, changed-by-admin conflict, and unsupported fields follow approved policies.
- **Edge cases:** Student has no optional scope values; user role changes while profile is open; profile email used as login identifier (AUTH-D03/D17).

---

---

### AUTH-F011 — Credential setup, change, recovery, and verification

#### Preconditions
- The relevant credential setup/change/recovery/verification workflow has been approved (AUTH-D03–D06).
- The actor has valid identity/recovery proof for the selected operation, subject to the approved workflow.

**Requirement status:** User.email/passwordHash schema fields source-confirmed; workflow not source-specified.  
**Specification status:** Decision required; no workflow is approved.  
**Primary actor:** User; admin involvement TBD.

**Supporting/system actor(s):** Credential/authentication service; Module 13 notification delivery if approved.
**Priority:** TBD (not agreed).

#### Purpose/outcome
Support approved account credential creation, change, and recovery without exposing or mishandling credential secrets.

#### Evidence
Schema contains `email` and `passwordHash`. This does not confirm password-based authentication, email verification, reset tokens, invitation links, or password policy.

#### Entry points
Not mapped in supplied sources. Possible UI locations are recommendations only: login/recovery and Profile & Settings.

#### Inputs
Credential setup/change/recovery inputs, reset token, verification code, or new password are all TBD pending auth method.

#### Main flow
Cannot be finalized until AUTH-D03–D06 are approved. At minimum, product must decide:
1. Account creation credential method.
2. Whether email verification is required.
3. Password change/recovery initiation and verification.
4. Reset token lifetime/reuse and invalidation.
5. Password requirements and history.
6. Whether credential changes revoke existing sessions.

#### Business/validation rules
No password strength, reset expiry, lockout, email verification, or recovery rules are source-confirmed. Do not invent these as final requirements.

#### System/data
`passwordHash` is schema-level storage representation; plaintext credentials must not be stored or returned. Any credential changes should be handled by the approved auth mechanism, not generic User Management update.

#### Failure/edge
Expired/used reset token, account enumeration, recovery delivery failure, concurrent password reset, compromised account, credential change during active sessions.

#### Audit/privacy
Credential/security event logging policy TBD. Never log plaintext credentials, reset tokens, or authentication secrets.

#### Acceptance criteria
Credential workflows exist only after their method/policy is approved; secrets are protected; invalid/expired recovery proof cannot change credentials; session effect follows approved policy.

#### QA
| Test ID | Scenario | Expected result | Trace |
|---|---|---|---|
| AUTH-QA-049 | Credential setup for new user | Follows approved onboarding method | AUTH-D04 |
| AUTH-QA-050 | Recovery with invalid/expired token | No credential change | AUTH-D05 |
| AUTH-QA-051 | Password/credential changed | Session invalidation follows policy | AUTH-D13 |
| AUTH-QA-052 | Recovery request for unknown email | Response follows anti-enumeration policy | AUTH-D05/D06 |

#### Unresolved
AUTH-D03–D06, D08, D13, D17.


#### Standard-template clarifications
- **Permissions/scope:** User credential operations apply only to the account owner unless an approved administrator-assisted reset policy grants a scoped exception.
- **UI behavior:** Set-password, change-password, verification, recovery request, and recovery completion screens/routes are not source-mapped.
- **System/data:** Credential secrets are handled by the approved credential service; passwordHash is not returned or logged. Token storage, expiry, one-time use, and rate limiting are AUTH-D05/D06.
- **Success result:** Credential state changes only after valid proof; account/session behavior follows AUTH-D13.
- **Alternate/failure:** Invalid/expired/reused recovery proof, unknown account, rate limit, and notification delivery failure are policy-gated.
- **Edge cases:** Recovery request races, credential reset while session active, and account status change during recovery.

---

---

### AUTH-F012 — MFA enrollment, challenge, and recovery

#### Preconditions
- MFA is explicitly approved for the target product/release and the account is eligible under AUTH-D07.
- Supported method, actor authority, and recovery policy are approved.

**Requirement status:** MFA fields source-confirmed; workflow not source-specified.  
**Specification status:** Decision required.

**Primary actor(s):** User/account holder; authorized administrator only if approved.
**Supporting/system actor(s):** Authentication/MFA service; security/audit subsystem.
**Priority:** TBD (not agreed).

#### Purpose/outcome
If MFA is approved for DEZOLVER, enroll and verify an additional factor according to policy.

#### Evidence
User schema includes `mfaEnabled` and `mfaSecret`. The source does not define MFA method, requirement, enrollment, challenge, backup/recovery, or secret protection.

#### Actors/entry
User/admin actors, entry point, and enforcement conditions TBD.

#### Inputs
MFA enrollment proof, challenge response, recovery proof, and factor secret are TBD.

#### Main flow
No source-confirmed or approved MFA flow exists. A target flow must be authored after AUTH-D07; until then, this function is not implementation-ready. The minimum decision-gated sequence is: (1) user/admin opens an approved MFA action, (2) system verifies actor/account eligibility, (3) system performs the approved enrollment/challenge/recovery proof, (4) system updates MFA state only after successful proof, and (5) system returns the approved result. This sequence is a review scaffold, not an approved target flow.

#### Validation and feedback
| Validation | Condition | Expected behavior | Feedback |
|---|---|---|---|
| MFA-V001 | MFA not approved or actor not eligible | Do not start MFA workflow | Product/security decision required. |
| MFA-V002 | Invalid challenge/enrollment proof | Reject according to approved MFA policy; do not mark MFA enabled | Exact retry/lockout feedback TBD. |
| MFA-V003 | Recovery proof invalid/expired/reused | Reject according to approved recovery policy | Feedback and rate limit TBD. |
| MFA-V004 | MFA secret is missing/malformed or cannot be safely stored | Do not complete enrollment | Technical/security handling TBD. |

#### Flow
Cannot be finalized until AUTH-D07 is approved. The product must determine:
- Optional vs mandatory MFA; role-specific enforcement.
- Supported method(s).
- Enrollment and verification process.
- Recovery/backup process.
- Lost-device handling.
- Session challenge behavior.
- MFA disable/reset authority.

#### Rules/data
- `mfaEnabled` boolean and `mfaSecret` field exist.
- No secret should be displayed or logged.
- Whether the schema field is encrypted/managed securely is a technical/security decision.
- MFA should not be treated as enabled merely because the schema fields exist.

#### Failure/edge
Invalid challenge, recovery, lost factor, admin reset, MFA setup interrupted, secret leakage.

#### Acceptance criteria
MFA is enforced only according to approved policy; secret material is protected; invalid proof cannot complete challenge/enrollment; recovery is controlled.

#### QA
| Test ID | Scenario | Expected result | Trace |
|---|---|---|---|
| AUTH-QA-053 | MFA enrollment | Follows approved method | AUTH-D07 |
| AUTH-QA-054 | Invalid MFA challenge | Denied | AUTH-D07 |
| AUTH-QA-055 | Lost MFA factor | Approved recovery path only | AUTH-D07 |
| AUTH-QA-056 | MFA secret returned/logged | Must not expose | AUTH-D07 |

#### Unresolved
AUTH-D07 and security review.


#### Standard-template clarifications
- **Permissions/scope:** No MFA actor, enforcement, or reset permission is source-confirmed; all depend on AUTH-D07.
- **UI behavior:** Enrollment, challenge, disable, and recovery UI are TBD; do not expose raw secret values.
- **System/data:** If approved, validate proof and update MFA state only after enrollment verification; `mfaEnabled`/`mfaSecret` fields do not prove that workflow exists.
- **Success result:** MFA status changes only under approved policy.
- **Alternate/failure:** Invalid proof, interrupted enrollment, lost factor, reset by unauthorized admin, or recovery failure must not bypass the approved security policy.
- **Edge cases:** MFA setup in multiple tabs, factor reset during active session, and account recovery when MFA is required.

---

---

### AUTH-F013 — View/manage sessions

#### Preconditions
- Actor is authenticated.
- Session visibility/revocation permissions and scope are approved (AUTH-D19/AUTH-D08).
- Target session exists if a detail/revoke action is requested.

**Requirement status:** Security / Sessions page and RefreshSession entity source-confirmed; actions/actors partly TBD.  
**Specification status:** Detailed draft; decision required.

**Primary actor(s):** SUPER_ADMIN; self-service user access is TBD.
**Supporting/system actor(s):** Session service; authentication subsystem; Module 15 audit subsystem.
**Priority:** TBD (not agreed).

#### Purpose/outcome
Allow authorized actor(s) to inspect and/or terminate sessions under an approved policy.

#### Actors/entry
- Source-confirmed: SUPER_ADMIN Security / Sessions page.
- Profile & Settings maps RefreshSession for Finance Team, HOD, Manager, Student.
- Whether end users can see/revoke their sessions is not specified.
- RBAC does not provide a specific Security / Sessions permission matrix.

#### Inputs
Session ID, target user ID, status/time filters (all UI fields TBD).

#### Main flow
1. Authorized actor opens Security / Sessions.
2. System determines permitted session scope.
3. System returns permitted session metadata only.
4. Actor selects a session termination action if authorized.
5. System applies approved revocation policy.
6. UI reflects resulting session state.

#### Business rules
- RefreshSession fields: id, userId, tokenHash, expiresAt, isRevoked, isImpersonation, actorId, createdAt, updatedAt.
- SUPER_ADMIN global visibility is suggested by platform page mapping, but exact session action permissions are TBD.
- Other roles should not be assumed to inspect all sessions; own-session access requires explicit decision.
- tokenHash and token material must never be displayed.
- Session revoke effects and race handling are TBD (AUTH-D08).

#### Validation/data
Validate session ID and actor scope; never accept arbitrary target user ID without authorization. Revoke semantics are TBD.

#### Failure/edge
Expired session; already revoked; target user outside scope; current session revocation; active impersonation session; account status changed.

#### Audit/privacy
Session viewing/revocation audit policy TBD; revoke events are sensitive. Do not log/display tokenHash.

#### Acceptance criteria
Only authorized session metadata is exposed; unauthorized session termination is denied; revoked sessions cannot be used under approved policy; token material remains secret.

#### QA
| Test ID | Scenario | Expected result | Trace |
|---|---|---|---|
| AUTH-QA-057 | SUPER_ADMIN opens Security / Sessions | View follows approved global policy | Page mapping |
| AUTH-QA-058 | User attempts to view another user's session | Denied unless specifically authorized | AUTH-D08 |
| AUTH-QA-059 | Authorized actor revokes session | Session invalidated per policy | AUTH-D08 |
| AUTH-QA-060 | API response exposes tokenHash | Must not expose | Schema/security |

#### Unresolved
AUTH-D08, D13, D15; admin/end-user session permissions and display fields.


#### Standard-template clarifications
- **Permissions/scope:** SUPER_ADMIN Security / Sessions page is mapped. Self-session listing/revocation and access by other roles are TBD; do not assume broad session visibility.
- **UI behavior:** Session list columns, device/client metadata, revoke action, confirmation, and empty/loading states are not mapped.
- **System/data:** Return only permitted metadata; never return tokenHash or raw token. Revoke only a session within actor scope and follow AUTH-D08.
- **Success result:** Authorized session action is reflected; a revoked session cannot be used under the approved session policy.
- **Alternate/failure:** Already expired/revoked session, target out of scope, current-session revoke, concurrent revoke, or stale session list.
- **Edge cases:** Impersonation session, account deletion/status change, and user has multiple active sessions.

---

---

### AUTH-F014 — Audit authentication/account/security actions

#### Preconditions
- An authentication/account/security event occurs.
- Module 15 defines whether the event is required to be recorded, its event taxonomy, and its criticality.

**Requirement status:** AuditLog schema and RBAC audit guidance source-confirmed; exact event list TBD.  
**Specification status:** Detailed draft; Module 15 dependency.

**Primary actor(s):** System event producer; authorized audit reader as defined by RBAC/Module 15.
**Supporting/system actor(s):** Audit subsystem; protected user/account/session modules.
**Priority:** Must-have traceability principle where RBAC requires audit; event catalog remains TBD.

#### Purpose/outcome
Provide traceability for user/account/security changes and selected authentication events.

#### Evidence
AuditLog schema fields: id, action, actorId, targetId, targetType, metadata, ipAddress, userAgent, createdAt. RBAC lists user creation/update/deletion and role/access changes among audit candidates. RBAC describes audit information including actor/user, action, resource, resource ID, timestamp, result/status, and context; exact implementation must follow schema/Module 15.

#### Actors/entry
System event trigger; audit readers per RBAC:
- SUPER_ADMIN: view.
- FINANCE_TEAM/HOD/MANAGER: view with scope restrictions (`*`).
- STUDENT: no Audit Logs page permission.
Audit visibility and record-level filtering are Module 15.

#### Inputs/event data
Event name, actor, target, target type, metadata, IP/user agent, timestamp. Canonical event naming and metadata schema TBD.

#### Validation and feedback
- Validate required actor/target/event fields according to Module 15’s approved event schema.
- Reject or safely handle malformed metadata; never allow secrets into event payloads.
- Missing context, event write failure, and redaction failure follow Module 15 criticality/retry policy.

#### Main flow
1. Sensitive auth/account/security operation occurs.
2. System determines whether the operation is in approved audit catalog.
3. System writes the event with permitted context.
4. Authorized audit reader can query the event through Module 15 controls.

#### Business rules
- User create/update/delete and role/access changes are audit candidates (RBAC).
- Failed security-sensitive operations may be recorded where required.
- Audit records must not contain passwords, passwordHash, MFA secrets, tokenHash, refresh tokens, or recovery secrets.
- Exact events, success/failure inclusion, retention, immutability, and access filters are TBD.

#### System/data
AuditLog record creation per approved Module 15 policy. This module does not define audit UI, export, retention, or purge.

#### Failure/edge
Audit write failure, duplicate event, missing actor (system action), privacy redaction, multi-step partial action. Shared audit failure policy is TBD.

#### Acceptance criteria
Approved sensitive events are traceable; actor/target are accurate; sensitive values are excluded; readers see only permitted audit scope.

#### QA
| Test ID | Scenario | Expected result | Trace |
|---|---|---|---|
| AUTH-QA-061 | User role/access change | Event recorded if required by Module 15 | RBAC audit candidates |
| AUTH-QA-062 | Audit metadata includes credential secret | Must be redacted/excluded | Security |
| AUTH-QA-063 | HOD requests out-of-scope audit record | Denied | RBAC scope |
| AUTH-QA-064 | Audit write fails during operation | Follows approved critical-event policy | Module 15 |

#### Unresolved
AUTH-D15; Module 15 event catalog, retention, audit-failure policy.


#### Standard-template clarifications
- **Permissions/scope:** RBAC audit-reader matrix is scoped; exact visibility/query action belongs to Module 15.
- **UI behavior:** Module 1 produces events; audit page, filters, exports, and detail UI are Module 15, not defined here.
- **System/data:** Write approved audit events with actor/target/context and safe metadata; exact required fields and event taxonomy are Module 15.
- **Success result:** Approved event is recorded and visible only to permitted readers.
- **Alternate/failure:** Audit write failure, unavailable event context, system actor, or redaction failure follows Module 15 policy.
- **Edge cases:** Failed auth attempts, actor/target same user, account deleted/archived, and events during impersonation.

---

---

### AUTH-F015 — Impersonation session handling

#### Preconditions
- AUTH-D14 explicitly approves impersonation as a feature. Until then, no impersonation precondition can be considered satisfied and no acting-as flow may be enabled.

**Requirement status:** Schema fields source-confirmed; feature not confirmed.  
**Specification status:** TBD / Decision Required; not implementation-ready.

**Primary actor(s):** TBD; do not assume SUPER_ADMIN is authorized.
**Supporting/system actor(s):** Session/authentication service; audit/security subsystem if feature approved.
**Priority:** TBD; feature existence not confirmed.

#### Evidence
RefreshSession includes `isImpersonation` and `actorId`. These fields alone do not prove impersonation is supported or enabled.

#### Purpose/outcome
If product explicitly approves impersonation, establish a controlled session in which an authorized actor operates as a target user while preserving the original actor identity.

#### Actors/entry
No source-defined actor roles, UI, or entry point. SUPER_ADMIN impersonation must not be assumed.

#### Inputs
Actor identity, target user, reason, scope, and session metadata would be required if feature is approved; all are TBD.

#### Main flow
No main flow is approved or source-supported. If AUTH-D14 approves impersonation, a separate reviewed flow must define actor authentication, authorization, target/scope selection, reason/approval, session establishment, visible disclosure, audit, and termination. If excluded, no impersonation flow exists.

#### Validation and feedback
- If the feature is excluded or the actor is unauthorized, reject the request and do not create an impersonation session.
- If the feature is approved, all actor/target/scope/session checks and failure feedback must be specified in AUTH-D14 before implementation.

#### Required decisions before specification
- Whether feature exists in target product.
- Allowed actor roles and target scope.
- Reason/approval requirement.
- Visible banner/notification to actor/target.
- Audit details and retention.
- Session separation and expiry.
- Whether sensitive operations are prohibited.
- Termination and revocation behavior.

#### Data/privacy
Schema fields are available, but semantics and constraints are unresolved. Do not use these fields as authorization by themselves.

#### Acceptance/QA
No implementation acceptance criteria can be approved until AUTH-D14 is decided. If excluded, verify impersonation fields are unused/removed according to schema governance. See AUTH-QA-067 and AUTH-QA-068 in [QA-TEST-CASES.md](./QA-TEST-CASES.md).

#### Unresolved
AUTH-D14 and Security review.


#### Standard-template clarifications
- **Permissions/scope:** No impersonation permission is source-confirmed. Do not enable based on schema fields; AUTH-D14 is a security/product gate.
- **Preconditions/flow:** If approved, actor authorization, target scope, reason/approval, separate session context, termination, and revocation must be specified before a detailed main flow can be approved.
- **UI behavior:** No source page, banner, notice, or control is mapped. All are TBD.
- **System/data:** `isImpersonation` and `actorId` are only schema fields; they do not authorize acting-as behavior.
- **Success/failure:** No success behavior is approved; unauthorized or unapproved impersonation must not occur.
- **Edge cases:** actor loses permission, target changes role/scope/status, sensitive action during impersonation, and session termination.

---

### AUTH-F016 — Remove user from active use (deactivate/archive; no permanent deletion)

**Requirement status:** Source RBAC lists SUPER_ADMIN CRUD Delete, but approved AUTH-D24 supersedes this source action: no permanent deletion is allowed. Use deactivation/archive only.
**Specification status:** Product policy approved; detailed retention/restore behavior is delegated to relevant lifecycle/data modules.
**Priority:** TBD (not prioritized in source).
**Primary actor(s):** SUPER_ADMIN.
**Supporting/system actor(s):** User Management backend; audit subsystem (Module 15); organization and academic modules owning related records.

#### Purpose and outcome
Do not expose or execute a permanent-delete action. When an authorized Super Admin removes a user from operational use, use the approved deactivation/archive lifecycle. Preserve historical records and attribution; apply retention rules by account status and data type.

#### Permissions and data scope
- RBAC User Management: SUPER_ADMIN has Full CRUD / V-C-U-D.
- FINANCE_TEAM, HOD, MANAGER, and STUDENT have no Delete permission in the cited User Management matrices.
- SUPER_ADMIN scope is global per RBAC.
- No other role may delete a user unless a future approved RBAC decision explicitly changes the matrix.

#### Preconditions
- Actor is authenticated as SUPER_ADMIN and passes backend authorization.
- Target User exists.
- Approved AUTH-D24 prohibits permanent deletion; deactivation/archive and retention rules govern the operation.
- Any required retention/legal/audit review has been completed.

#### Entry point and trigger
- Source-supported page: SUPER_ADMIN User Management.
- Delete button/menu, confirmation dialog, route, and exact UI wording are not mapped and remain TBD.
- Trigger: authorized actor requests deletion of a selected user.

#### UI behavior
The source confirms the CRUD permission, not the UI. A delete affordance, confirmation step, warning text, and post-action navigation must be approved before implementation. Do not infer a permanent-delete confirmation dialog from the CRUD label.

#### Inputs
| Input | Source | Type | Required? | Validation / notes |
|---|---|---|---|---|
| Target user ID | User.id | ID | Yes | Must identify existing target; target must be in actor's allowed scope. |
| Confirmation / reason | Not in schema | TBD | TBD | Whether confirmation or reason is required is AUTH-D24; no reason field is shown in User schema. |

#### Main flow (decision-gated)
1. SUPER_ADMIN opens the target user record in User Management.
2. System verifies actor identity, role, and global permission.
3. UI presents the approved delete action and confirmation behavior (TBD).
4. Actor confirms the action if the approved flow requires confirmation.
5. Backend validates target existence and the approved deletion constraints.
6. System applies the approved delete/archive/deactivate/retention behavior (AUTH-D24).
7. System records the approved audit event and any permitted related side effects (Module 15).
8. UI displays the approved outcome and updates the list/detail view.

#### Business rules
- AUTH-BR-013: Permanent deletion is prohibited by approved AUTH-D24; source CRUD Delete is superseded by deactivate/archive.
- AUTH-BR-014: Deactivation/archive must preserve historical submissions, assessments, certificates, billing, and audit attribution; retention is status/data-type-specific.
- AUTH-BR-015: Deletion permission is not implied for Finance Team, HOD, Manager, or Student.
- AUTH-BR-016: AUTH-F016 must use deactivation/archive lifecycle behavior; no permanent deletion endpoint/action is allowed.

#### Validation and feedback
| Validation ID | Condition | Validation point | Expected behavior | Feedback |
|---|---|---|---|---|
| AUTH-V014 | Actor is not SUPER_ADMIN | Backend | Deny; no mutation | Shared access-denied response TBD. |
| AUTH-V015 | Target user does not exist | Backend | No deletion; no unrelated mutation | Not-found feedback TBD. |
| AUTH-V016 | Target has protected/history relationships | Backend | Apply approved retention/dependency policy; if undefined, block implementation | Product feedback TBD. |
| AUTH-V017 | Deletion semantics AUTH-D24 unresolved | Product/release gate | Do not implement a destructive action based on CRUD label alone | Decision required. |

#### System/backend behavior
- Enforce permission server-side; frontend visibility is not authorization.
- Validate related records and referential/retention constraints according to approved behavior.
- Avoid partial deletion of user and related data. Transaction, archive, anonymization, cascade, and restore mechanics are not specified in source and must not be guessed.

#### Data changes and state transitions
Potentially affected object: User and related records. The source does not establish which records are deleted, retained, anonymized, detached, or archived. Permanent deletion is not approved. Deactivation/archive is approved; data retention specifics follow the approved cross-module policy. Preserve actor/target traceability according to Module 15.

#### Success result
The account is deactivated or archived, never permanently deleted; historical records remain according to retention policy.

#### Alternate/failure handling
- Unauthorized actor: deny; no data change.
- Target not found: no data change.
- Related historical records exist: apply approved retention behavior; do not silently cascade.
- Concurrent deletion/status change: conflict behavior TBD under Module 18.
- Audit failure: follow Module 15 critical-event policy.

#### Edge cases
- SUPER_ADMIN attempts to delete own account.
- User is the only SUPER_ADMIN / only administrator for an organization.
- Target owns historical submissions, scores, certificates, support tickets, or billing records.
- Target has active RefreshSession rows or impersonation context.
- Duplicate/retry request or simultaneous status change.

#### Audit, notification, and privacy
RBAC identifies User deletion as a sensitive audit candidate. Event name, before/after metadata, notification behavior, and retention are owned by Modules 15 and 13. Never include credential hashes, MFA secrets, tokenHash, or raw session secrets in audit metadata.

#### Acceptance criteria
- Only SUPER_ADMIN can invoke the source-defined User Management Delete permission.
- Unauthorized attempts do not mutate user or related data.
- Deletion semantics are explicitly approved and are not inferred from `status`/`isActive`.
- Related historical records follow an approved retention/attribution policy.
- Audit and session effects follow approved Module 15/13/17 decisions.

#### QA
See AUTH-QA-065 and AUTH-QA-066 in [QA-TEST-CASES.md](./QA-TEST-CASES.md). These cases are decision-gated until AUTH-D24 is approved.

#### Unresolved items
AUTH-D18, AUTH-D24; related-record retention, self-delete, last-admin safeguards, active sessions, restore/reversibility, and audit/notification policy.


#### Standard-template clarifications
- **Permissions/scope:** Only SUPER_ADMIN Delete is source-confirmed; no Delete for other roles in the User Management matrix.
- **UI behavior:** No delete control/confirmation screen is source-mapped.
- **System/data:** No physical delete, cascade, anonymization, or archive is authorized until AUTH-D24 is decided.
- **Success/failure:** The exact result is TBD; unauthorized requests and decision-gated deletion must not mutate records.
- **Edge cases:** self-delete, last platform admin, active sessions, and historical user-owned records.

---

---

## 10. Business rules and validation summary

| Rule ID | Rule | Evidence/status |
|---|---|---|
| AUTH-BR-001 | Protected operations require authenticated identity. | Derived from RBAC authorization flow |
| AUTH-BR-002 | Role/scope come from trusted account context, not arbitrary client values. | Source-confirmed RBAC principle |
| AUTH-BR-003 | Account login eligibility by status is not defined by source. | TBD |
| AUTH-BR-004 | Authentication method, identity uniqueness, MFA, lockout, and recovery policy are not source-defined. | TBD |
| AUTH-BR-005 | Permission + scope + ownership checks must pass before protected operation. | Source-confirmed RBAC |
| AUTH-BR-006 | Frontend visibility is not security enforcement. | Source-confirmed RBAC |
| AUTH-BR-007 | Student private records use authenticated ownership. | Source-confirmed RBAC |
| AUTH-BR-008 | Cross-college/department/section/group access is denied. | Source-confirmed RBAC |
| AUTH-BR-009 | Authorization denial must not execute protected mutation. | Source-confirmed RBAC |
| AUTH-BR-010 | User Management Manager permission conflict is unresolved. | Source discrepancy |
| AUTH-BR-011 | Finance Team exists in Business Flow/RBAC but is absent from schema Role enum. | Source discrepancy |
| AUTH-BR-012 | User.status and User.isActive semantics/precedence are unresolved. | Source discrepancy |
| AUTH-BR-013 | CRUD Delete permission does not define hard-delete/soft-delete/archive semantics. | Source-confirmed permission; behavior TBD (AUTH-D24) |
| AUTH-BR-014 | Historical user-owned records must not be silently cascade-deleted by an unapproved account deletion workflow. | Derived safety requirement; retention decision AUTH-D24 |
| AUTH-BR-015 | User Management Delete is source-listed only for SUPER_ADMIN. | RBAC platform/page/API matrices |
| AUTH-BR-016 | Delete and account-status change are separate functions until approved otherwise. | Source/schema distinction; AUTH-D24 |

## 11. Data and state reference

### 11.1 User entity (source schema)

| Field | Schema type | Source meaning/functional relevance | Required/editable/constraints |
|---|---|---|---|
| id | varchar | User identifier | Generation/constraints TBD |
| email | varchar(255) | Email/identity/contact field | Requiredness, uniqueness, case rules, editability TBD |
| passwordHash | varchar(255) | Credential hash field | Auth method and security storage policy TBD; never exposed |
| fullName | varchar(255) | Display name | Requiredness/validation TBD |
| role | Role enum / membership role | Role authorization | Add FINANCE_TEAM; target role/scope resides on membership for tenant roles |
| status | UserStatus | Global account lifecycle | INVITED, ACTIVE, SUSPENDED, DEACTIVATED, ARCHIVED; only ACTIVE authenticates |
| isActive | boolean | Redundant lifecycle flag | Remove from target schema; status is authoritative |
| mfaEnabled | boolean | MFA state flag | MFA policy TBD |
| mfaSecret | varchar(255) | MFA secret field | Method/storage protection TBD; never exposed |
| collegeId | varchar | Source single-college scope | Move tenant scope to membership relation in target identity model |
| departmentId | varchar | Source single-department scope | Move department scope to membership relation in target identity model |
| sectionId | varchar | Source single-section scope | Move section scope to membership relation in target identity model |
| academicBatch | varchar | Academic batch | Validation/eligibility TBD |
| createdAt | timestamp | Creation timestamp | System-managed behavior TBD |
| updatedAt | timestamp | Update timestamp | System-managed behavior TBD |

### 11.2 Role enum and target identity model

Source schema lists SUPER_ADMIN, HOD, MANAGER, STUDENT and omits FINANCE_TEAM. Approved target role set includes FINANCE_TEAM. Add it to the target enum/model.

The target identity model is global User + organizational membership records (e.g. UserMembership). Keep globally unique normalized email and credentials on User. Membership records hold role and college/department/section/group scope and membership status. SUPER_ADMIN is platform-wide; tenant roles use membership scope. A user may have multiple college memberships and switch membership/workspace after authentication. Exact physical schema, migration, and foreign-key design are Engineering deliverables.

### 11.3 Account status target enum

- INVITED
- ACTIVE
- SUSPENDED
- DEACTIVATED
- ARCHIVED

Approved AUTH-D09/D10: `status` is authoritative; remove `isActive`. Only ACTIVE global users may authenticate. INVITED, SUSPENDED, DEACTIVATED, and ARCHIVED are blocked from login. Membership-level status separately governs access to a particular college membership.

### 11.4 RefreshSession entity (source schema)

| Field | Type | Meaning in source | Functional rules |
|---|---|---|---|
| id | varchar | Session identifier | Generation/constraints TBD |
| userId | varchar | Associated User | Ownership/scope checks required |
| tokenHash | varchar(255) | Token hash | Never return/log; token handling TBD |
| expiresAt | timestamp | Expiry timestamp | Lifetime/refresh rules TBD |
| isRevoked | boolean | Revocation flag | Revocation semantics TBD |
| isImpersonation | boolean | Impersonation flag | Feature support TBD |
| actorId | varchar | Actor associated with impersonation | Meaning/authority TBD |
| createdAt | timestamp | Creation timestamp | System-managed |
| updatedAt | timestamp | Update timestamp | System-managed |

### 11.5 AuditLog entity (source schema)

| Field | Type | Meaning | Functional rules |
|---|---|---|---|
| id | varchar | Audit identifier | Generation TBD |
| action | varchar(100) | Action name | Canonical event catalog Module 15 |
| actorId | varchar | Actor | System actor semantics TBD |
| targetId | varchar(255) | Target record | Optionality TBD |
| targetType | varchar(100) | Target type | Canonical taxonomy TBD |
| metadata | json | Context | Redaction/privacy rules Module 15 |
| ipAddress | varchar(255) | IP context | Collection/retention policy TBD |
| userAgent | varchar(1000) | Client context | Collection/retention policy TBD |
| createdAt | timestamp | Event time | System-managed |

### 11.6 Schema gaps / cautions
- No explicit User invitation/reset-token entity is shown.
- Source schema lacks multi-membership; target requires a global User identity plus membership relation (e.g. UserMembership) holding tenant role/scope. One person can have multiple memberships and switch active membership/workspace.
- No account lockout/failure counter field is shown.
- No explicit session device/name/last-used fields are shown.
- No canonical impersonation authorization relation is shown.
- No MFA recovery/backup-code fields are shown.
- Conceptual schema does not establish whether constraints exist in the deployed database.

## 12. State and lifecycle reference

### 12.1 Account status transition table

| Current status | Trigger | Next status | Login/access/session effect | Status |
|---|---|---|---|---|
| ACTIVE | Suspend | SUSPENDED | TBD | Decision required |
| ACTIVE | Deactivate | DEACTIVATED | TBD | Decision required |
| ACTIVE | Archive | ARCHIVED | TBD | Decision required |
| SUSPENDED | Reactivate | ACTIVE? | TBD | Decision required |
| DEACTIVATED | Reactivate | ACTIVE? | TBD | Decision required |
| ARCHIVED | Restore | ACTIVE? | TBD | Decision required |

This is a list of questions/transitions to approve, not an approved state machine. Module 17 owns canonical state transitions.

### 12.2 Session lifecycle

| State/condition | Source representation | Behavior |
|---|---|---|
| Created | RefreshSession row | Creation/issuance policy TBD |
| Valid | expiresAt in future and isRevoked=false | Validity rules TBD |
| Expired | expiresAt reached | Renewal/denial policy TBD |
| Revoked | isRevoked=true | Reuse denied under recommended security treatment; confirm AUTH-D08 |
| Impersonation | isImpersonation=true | Feature and rules TBD |

## 13. Audit, notifications, and privacy

### 13.1 Audit
- RBAC lists User creation/update/deletion and role/access changes as audit candidates.
- User and Profile & Settings mappings include AuditLog in several roles; Super Admin Audit Logs page is separately mapped.
- Final event names, success/failure inclusion, actor/target metadata, retention, and access belong to Module 15.
- Never log password, passwordHash, MFA secret, tokenHash, access/refresh tokens, or recovery secrets.

### 13.2 Notifications
Business Flow/Page mapping does not specify auth notification triggers or recipients. Account invitations, email verification, password recovery, MFA/security notifications, and account-status notices require Module 13 decisions. Do not assume notification delivery is enabled.

### 13.3 Privacy
- User profile and academic ownership must follow RBAC scope.
- Student cannot access another student's private profile/submissions/assessment attempt.
- Credential and session secrets are never exposed.
- Audit data is sensitive and scope-limited.

## 14. Cross-module dependencies

| Module | Dependency |
|---|---|
| 2 — College / Department / Section | Organizational hierarchy and assignment validity |
| 3 — Subscription & Seat Management | User provisioning/eligibility may depend on seats/subscription; rules not established here |
| 4 — Course & Enrollment | Student identity, enrollment, academic scope |
| 10 — Groups | Group membership and permitted Group scope |
| 13 — Notifications | Invite, verification, recovery, security notification delivery |
| 15 — Audit Logging | Event catalog, retention, visibility, export |
| 17 — Functional State Machines | Account/session lifecycle transitions |
| 18 — Edge Cases & Failure Handling | Shared errors, retries, conflict, service failure |
| All protected modules | Authentication, role, scope, ownership context from this module/RBAC |

## 15. Edge cases and failure handling

| Case | Expected treatment | Evidence/status |
|---|---|---|
| Unauthenticated request to protected resource | Deny before protected operation | RBAC source-confirmed |
| Correct role, wrong college | Deny | RBAC source-confirmed |
| HOD requests foreign department | Deny | RBAC source-confirmed |
| Manager requests unassigned section/group | Deny | RBAC source-confirmed |
| Student requests another student's private record | Deny | RBAC source-confirmed |
| Client submits forged role/tenant/scope | Ignore/reject; no access expansion | Derived from RBAC |
| User.status and isActive conflict | Apply no guessed precedence; decision required | Source discrepancy |
| FINANCE_TEAM login/account role | Block role model finalization until enum discrepancy resolved | Source discrepancy |
| Role/scope changed while session active | Session invalidation/effect TBD | AUTH-D13 |
| User deactivated while active | Login/session policy TBD | AUTH-D09/D13 |
| Duplicate user identity | Uniqueness policy TBD | AUTH-D17 |
| Refresh session expires/revoked | Renewal policy TBD | AUTH-D08 |
| MFA secret or tokenHash returned/logged | Must not expose | Security requirement derived from sensitive schema |
| Audit write failure | Module 15 critical-event policy | TBD |
| Impersonation field present but feature disabled | Do not enable feature by inference | Source/schema caution |

## 16. Decision log

**Decision owner:** Product Owner. The following decisions were approved by the Product Owner on 2026-11-05. Cross-module implementation details are noted where applicable.

| Decision ID | Approved target decision | Impact / implementation note | Status |
|---|---|---|---|
| AUTH-D01 | FINANCE_TEAM is a first-class role. | Add to target role enum/model and RBAC. | Approved |
| AUTH-D02 | One shared login/authentication experience; role-specific workspace identity and dashboard after login. | Do not create separate role authentication systems. | Approved |
| AUTH-D03 | Demo uses email/password. | Other methods may be added later; not demo scope. | Approved |
| AUTH-D04 | Account creation authority varies by creator role. See creator matrix below. | Enforce role + college/department/section scope server-side. | Approved |
| AUTH-D05 | Email verification required for SUPER_ADMIN, FINANCE_TEAM, HOD, MANAGER; STUDENT where email exists. Email reset for all; Super Admin also has admin recovery. | Notification delivery/templates belong to Module 13. | Approved |
| AUTH-D06 | Password/security defaults: min 12 chars; max at least 64; no character-class requirements; allow spaces/paste/password managers; reject common/compromised passwords; 5 consecutive failures → 15-minute lockout; IP limit 10 attempts/15 minutes; last 5 passwords blocked; reset token single-use/15 minutes; Argon2id preferred; password reset invalidates all sessions. | Engineering/Security must implement lockout/rate-limit safely and avoid account-enumeration leakage. | Approved |
| AUTH-D07 | Role-based MFA: mandatory for SUPER_ADMIN/FINANCE_TEAM in production; recommended for HOD/MANAGER; student standard password baseline. MFA is deferred from demo. | MFA design/enrollment/recovery is a later release/security task. | Approved |
| AUTH-D08 | Access token 15 minutes; refresh session 7 days; remember-me 30 days; automatic refresh; multiple devices allowed; logout all devices supported; idle timeout 30 minutes for admin/finance and 7 days for students. | Refresh rotation, cookie/storage implementation, and session risk controls require Engineering/Security design. | Approved |
| AUTH-D09 | User lifecycle status includes INVITED, ACTIVE, SUSPENDED, DEACTIVATED, ARCHIVED. Only ACTIVE global users may authenticate. | Membership status separately governs a particular college membership. | Approved |
| AUTH-D10 | status is the single authoritative global lifecycle field; remove redundant isActive. | Update target schema/migration; emailVerifiedAt/MFA remain independent. | Approved |
| AUTH-D11 | Global User identity + multiple organizational memberships; one global normalized email/credential; membership holds role and scope; user can belong to multiple colleges and switch memberships. SUPER_ADMIN is platform-wide. | Introduce conceptual UserMembership relation; exact DB shape/migration is Engineering design. | Approved |
| AUTH-D12 | Manager can view/create individual students/bulk-import/edit student details, suspend and deactivate within scope. Email changes, password reset, reactivation, and department-section reassignment restricted; cannot delete, change role, or transfer college. | Sensitive actions require explicit higher authority. | Approved |
| AUTH-D13 | Password change/reset, account status to suspended/deactivated/archived/invited, and email change revoke all sessions. Email change also requires verification. Role/department/section changes revoke affected membership sessions; membership suspend/remove revokes its sessions. Name/photo edits do not revoke sessions. | Membership-scoped session invalidation must be represented in target session model. | Approved |
| AUTH-D14 | Super Admin impersonation allowed with reason, audit logging, and restricted actions. | Allowed action list, disclosure, actor/target separation, and termination mechanics must be detailed before implementation. | Approved (concept; technical controls pending) |
| AUTH-D15 | Audit all: successful/failed login, logout/session revocation, password/MFA changes, user create/update/suspend/deactivate, role/scope changes, impersonation, and profile changes. | Module 15 defines canonical events, privacy/redaction, retention, and failure policy. | Approved (event scope) |
| AUTH-D16 | Self-edit full name, email, phone, photo, password, and only explicitly approved personal fields. Role, college, department, section, academic batch, student/register number, and account status are admin-controlled. | Email verification/session invalidation follows D13/D17. | Approved |
| AUTH-D17 | Email globally unique, normalized by trim + case-insensitive comparison, database-enforced. New email requires verification. | Use one identity across memberships; no duplicate account per college. | Approved |
| AUTH-D18 | Retention differs by account status and data type. | Preserve historical attribution; detailed retention policy owned by applicable data/governance modules. | Approved (policy; details cross-module) |
| AUTH-D19 | Users can view active sessions and revoke individual sessions. | Session UI exposes safe metadata only; never tokenHash/raw tokens. | Approved |
| AUTH-D20 | Role-specific default dashboards; return to originally requested authorized page where appropriate. | SUPER_ADMIN/Finance/HOD/Manager/Student each have their own dashboard. | Approved |
| AUTH-D21 | Finance Team manages college subscription/plan/financial relationship and creates HODs/Managers for departments; Super Admin may create HODs/Managers on college request. | Align with Business Flow v2 and Module 2/3 provisioning dependencies. | Approved |
| AUTH-D22 | Email notifications only for password changed/reset and email changed. No notifications for MFA enable/disable (MFA deferred), role/scope changes, suspension/deactivation, or new-device login. | Module 13 owns delivery and templates. | Approved |
| AUTH-D23 | Define field-level permissions for sensitive fields only; ordinary fields follow role/action/scope permissions. | Do not create a blanket field-by-field matrix for every ordinary field. | Approved |
| AUTH-D24 | No permanent deletion. Deactivate/archive only. | Replace source CRUD Delete affordance with deactivate/archive; retain historical records and apply status/data-type retention. | Approved |
| AUTH-D25 | Finance Team is college-scoped: subscription/plan and college↔Super Admin financial management; create HOD/Manager accounts per department. | Detailed billing/financial capabilities remain in Module 3 and related finance specs; creator matrix applies. | Approved (scope; cross-module detail pending) |

### 16.1 Creator authority matrix

| Created by | Finance Team | HOD | Manager | Student |
|---|---:|---:|---:|---:|
| SUPER_ADMIN | Yes | Yes | Yes | Yes |
| FINANCE_TEAM | No | Yes | Yes | No |
| HOD | No | No | Yes | Yes |
| MANAGER | No | No | No | Yes |
| STUDENT | No | No | No | No |

All account creation is restricted to the actor's authorized organizational scope. SUPER_ADMIN is global; Finance Team is college-scoped; HOD is college + department; Manager is assigned department/section scope.

## 17. Acceptance criteria and QA coverage summary

### 17.1 Module-level acceptance criteria
1. The role list is consistent across Business Flow, RBAC, and schema (AUTH-D01 resolved).
2. Every protected operation authenticates and checks permission, scope, and ownership.
3. College, department, section, group, and ownership isolation are enforced server-side.
4. User Management actions are limited by the approved role/action matrix.
5. Student access to user/private records is limited to self and permitted academic data.
6. User provisioning validates role and organizational scope.
7. Account status and `isActive` behavior follow an approved canonical state policy.
8. Session renewal, logout, revocation, and session administration follow approved policy.
9. MFA and impersonation are explicitly enabled or excluded; schema fields alone do not enable them.
10. Profile self-service cannot change role/tenant scope unless a separate approved function authorizes it.
11. Credentials, MFA secrets, tokenHash, and recovery secrets are not exposed or logged.
12. Sensitive user/account changes are audited according to Module 15.
13. No source conflict is silently treated as an approved target requirement.

### 17.2 QA coverage
Detailed test preconditions, data, steps, expected results, and decision-readiness are maintained in [QA-TEST-CASES.md](./QA-TEST-CASES.md), as permitted by Standard v1.0. The function-level QA tables in this README are the traceability summaries. AUTH-QA-001–068 cover:
- Login identity and invalid credentials.
- Account status and MFA decisions.
- Authentication context, role/scope/ownership, tenant isolation.
- Logout, refresh, expiry, revocation, multi-device behavior.
- User list/view permissions.
- User creation, role/scope validation, duplicate identity.
- User update and field-level authorization.
- Account status transition and session effect.
- Profile self-service and ownership.
- Credential recovery and verification.
- MFA enrollment/challenge/recovery.
- Session view/revoke privacy.
- Audit events, redaction, and scope.

QA cases marked with unresolved decisions become executable only after the linked decisions are approved.

## 18. Review and approval checklist

- [ ] Module structure follows Functional Specification Standard v1.0.
- [ ] Source facts are traceable to Business Flow, Page-to-DB Mapping, RBAC, and schema.
- [x] FINANCE_TEAM enum conflict is resolved in target requirements (schema update required).
- [x] User Management MANAGER permission discrepancy is resolved in target requirements.
- [x] Authentication method, login routing, and credential lifecycle are approved.
- [x] Role-based account creation authority is approved.
- [x] Finance Team account creation authority/targets and onboarding relationship are approved (AUTH-D21/D25).
- [x] Account status and removal of `isActive` are approved.
- [x] Session lifetime, refresh, logout, revocation, and self-service session visibility are approved.
- [x] Role-based MFA policy and controlled Super Admin impersonation are approved (MFA deferred from demo).
- [x] Sensitive-field permissions and Manager student-management boundaries are approved.
- [x] Profile editable-field policy is approved.
- [ ] Audit event and notification dependencies are reviewed by Modules 15 and 13.
- [ ] QA cases in [QA-TEST-CASES.md](./QA-TEST-CASES.md) have complete preconditions, steps, expected results, and decision trace; decision-gated tests are not executed prematurely.
- [x] Product Owner approved target decisions AUTH-D01–AUTH-D25 on 2026-11-05.

**Readiness status:** Product decisions AUTH-D01–AUTH-D25 are approved. Module 1 is now a target requirements baseline, but not yet implementation-ready until Engineering resolves membership/session schema, detailed impersonation controls, and cross-module integration (Modules 2, 3, 13, 15, 17, and 18).

## 19. Source references

1. [Business Flow v2](../../docs/source-text/Business%20Flow%20v2.md)
2. [Business Flow (original)](../../docs/source-text/Business%20Flow.md)
3. [Page-to-DB Mapping v2](../../docs/source-text/Page%20-%20DB%20Mapping%20v2.md)
4. [Page-to-DB Mapping (original)](../../docs/source-text/Page%20-%20DB%20Mapping.md)
5. [RBAC](../../docs/source-text/RBAC.md)
6. [Schema](../../docs/source-text/scheema.md)
7. [Functional Specification Standard](../FUNCTIONAL-SPECIFICATION-STANDARD.md)
8. [Module 1 QA Test Cases](./QA-TEST-CASES.md)

---