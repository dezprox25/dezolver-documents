# DEZOLVER Role-Based Access Control (RBAC)

> Markdown reconstruction of `RBAC (1).pdf` from the DEZOLVER source documentation.
> This document preserves the source's role definitions, permission matrices, data-scope rules, authorization flows, Judge0 integration, audit/security guidance, and testing checklist. It is a source-text conversion, not a functional-specification interpretation.

---

## 1. RBAC Overview

RBAC is the access-control mechanism used in DEZOLVER to determine what each stakeholder can access, what actions they can perform, and which data they are allowed to access based on their assigned role.

### DEZOLVER Roles

- **SUPER ADMIN** — Platform-wide administration
- **FINANCE_TEAM** — College-level financial and subscription operations
- **HOD** — Department-level academic administration
- **MANAGER** — Section/lab-level operational management
- **STUDENT** — Student self-service and academic activities

### Two Levels of Access Control

1. **Platform-Level Access**
   - Pages and modules a role can access
   - APIs available to the role
   - Actions permitted: View, Create, Update, Delete

2. **Database-Level Access**
   - Determines which records the role can access
   - Enforces college, department, section, group, and ownership boundaries
   - Prevents unauthorized cross-tenant or cross-scope access

### Core Authorization Principle

```text
Authentication
    ↓
Role Check
    ↓
Permission Check
    ↓
Tenant / Data Scope Check
    ↓
Database Operation
    ↓
Audit Logging (where required)
```

---

## 2. Stakeholders & Role Scope

| Role | Primary responsibility | Access scope |
|---|---|---|
| SUPER_ADMIN | Manage and control the DEZOLVER platform | Global |
| FINANCE_TEAM | Manage college subscription, billing, and related college administration | College |
| HOD | Manage department-level academic activities | College + Department |
| MANAGER | Manage assigned section-level students and activities | College + Department + Section |
| STUDENT | Access personal academic activities and submit work | Self + Permitted Academic Data |

### Role Boundaries

#### SUPER_ADMIN
- Full platform-level access
- Can manage colleges and global platform entities
- No tenant restriction

#### FINANCE_TEAM
- Operates within its assigned college
- Manages subscription and billing-related functions
- Cannot access unrelated college data

#### HOD
- Manages the academic structure and activities of the assigned department
- Access is restricted to the department within the college
- Manages relevant courses, problems, groups, labs, assessments, and contests

#### MANAGER
- Operates within the assigned section
- Manages students and activities within the permitted section/group scope
- Cannot access another section's restricted data

#### STUDENT
- Accesses only permitted academic resources and own records
- Can perform student actions such as submissions and participation
- Cannot access administrative functions or another student's private data

---

## 3. Platform-Level Permission Matrix

| Module | SUPER_ADMIN | FINANCE_TEAM | HOD | MANAGER | STUDENT |
|---|---|---|---|---|---|
| College Management | Full CRUD | View | View | — | — |
| Department Management | Full CRUD | View | View/Update | — | — |
| User Management | Full CRUD | View/Create* | View/Create/Update* | View* | View own |
| Subscriptions & Billing | Full CRUD | View/Create/Update | View | — | View own |
| Courses | Full CRUD | View | Full CRUD | Create/Update* | View/Enroll |
| Course Problems | Full CRUD | View | Full CRUD | Create/Update* | View |
| Labs | Full CRUD | View | Full CRUD | Full CRUD* | View/Submit |
| Problems & Test Cases | Full CRUD | View | Full CRUD | Create/Update* | View/Solve |
| Groups | Full CRUD | View | Full CRUD | Full CRUD* | Join/View |
| Assessments | Full CRUD | View | Full CRUD | Create/Update* | View/Attempt |
| Contests | Full CRUD | View | Full CRUD | Create/Update* | View/Participate |
| Contest Ranking | View/Manage | View | View/Manage | View | View |
| Code Submissions | View/Manage | — | View | View* | Create/View own |
| Audit Logs | Full View | View* | View* | View* | — |

### Permission Rules

- **View** → Can access/read permitted data.
- **Create** → Can create records within the assigned scope.
- **Update** → Can modify records within the assigned scope.
- **Delete** → Only available where explicitly permitted.
- ***** → Restricted by college, department, section, group, or ownership scope.

---

## 4. Database-Level Data Scope

| Role | Data scope | Access boundary |
|---|---|---|
| SUPER_ADMIN | Global | All colleges, departments, sections, groups, and platform data |
| FINANCE_TEAM | College | Assigned college only |
| HOD | Department | Assigned college + department |
| MANAGER | Section / Group | Assigned college + department + permitted section/group |
| STUDENT | Self + Academic | Own records + permitted courses, labs, contests, and assessments |

### Database Scope Rules

#### SUPER_ADMIN
- No tenant restriction.
- Can access platform-wide records.
- Can manage colleges, users, subscriptions, and academic data.

#### FINANCE_TEAM
- Every query must be restricted to the assigned `collegeId`.
- Cannot access unrelated college records.
- Mainly operates on subscriptions, billing, and college-level administration.

#### HOD
Data must satisfy both:
- `collegeId = assigned college`
- `departmentId = assigned department`

Can manage department-level academic resources.

#### MANAGER
Data must satisfy:
- `collegeId = assigned college`
- `departmentId = assigned department`
- `sectionId / groupId = permitted scope`

Access is restricted to operational data within the assigned section/group.

#### STUDENT
- Personal records must be restricted using `studentId = authenticated user`.
- Academic resources are accessible only when the student is enrolled, assigned, or permitted through the relevant course/group/contest.

### Tenant Isolation Rule

Every database operation must apply the appropriate tenant and scope filters.

```text
User Request
    ↓
Authentication
    ↓
Role & Permission Check
    ↓
College Scope Check
    ↓
Department Scope Check
    ↓
Section / Group / Ownership Check
    ↓
Database Operation
```

---

## 5. Role → Page → Permission Mapping

Platform access is controlled by the user's role, while every action is further restricted by the user's authorized college, department, section, group, or ownership scope.

| Application page / module | SUPER_ADMIN | FINANCE_TEAM | HOD | MANAGER | STUDENT |
|---|---|---|---|---|---|
| Dashboard | V | V | V | V | V |
| College Management | V/C/U/D | V | — | — | — |
| Department Management | V/C/U/D | V | V/U | — | — |
| User Management | V/C/U/D | V/C* | V/C/U* | V/U* | V OWN |
| Courses & Labs | V/C/U/D | V | V/C/U/D | V/C/U* | V / SUBMIT |
| Problems | V/C/U/D | — | V/C/U/D | V/C/U* | V / SOLVE |
| Assessments | V/C/U/D | — | V/C/U/D | V/C/U* | V / ATTEMPT |
| Groups | V/C/U/D | V | V/C/U/D | V/C/U* | V OWN |
| Contests | V/C/U/D | V | V/C/U/D | V/C/U* | V / PARTICIPATE |
| Contest Ranking | V | V | V | V | V OWN / PERMITTED |
| Code Submissions | V | — | V | V* | C / V OWN |
| Certificates | V/C/U/D | — | V/C/U* | V* | V OWN |
| Support / Tickets | V/C/U/D | V/C/U* | V/C/U* | V/C/U* | C / V OWN |
| Notifications | V/C/U/D | V/C/U* | V/C/U* | V/C/U* | V OWN |
| Profile & Settings | V/U | V/U | V/U | V/U | V/U OWN |
| Audit Logs | V | V* | V* | V* | — |

### Scope Rules

Permissions in the matrix above do not provide unrestricted access.

Where `*` is present, the action must additionally pass the applicable data-scope check.

```text
College
  ↓
Department
  ↓
Section / Group
  ↓
Ownership
```

### Page-Level Authorization Rule

Access to a page does not automatically provide access to every record displayed on that page.

```text
User
  ↓
Authentication
  ↓
Role Check
  ↓
Page / Module Permission
  ↓
Data Scope Check
  ↓
Ownership Check where applicable
  ↓
Database Operation
```

### Backend Enforcement

The same permissions must be enforced at the backend/API level.

Frontend page restrictions are only for user experience and must not be treated as the security mechanism.

```text
Frontend Restriction
        +
Backend Authorization
        +
Database Scope Validation
        =
RBAC Enforcement
```

**Final principle:** A role determines what a user can do; data scope determines which records they can do it on.

---

## 6. Role → API → Permission Mapping

Page-level restrictions alone are not sufficient for security. Every protected API must verify the authenticated user's role, required permission, and applicable data scope before performing a database operation.

### API Authorization Structure

```text
API Request
    ↓
Authentication
    ↓
Identify User + Role
    ↓
Check Required Permission
    ↓
Check Data Scope
    ↓
Check Resource Ownership
    ↓
Execute Database Operation
    ↓
Audit Log (where required)
```

### API Permission Types

| Permission | API operation | Purpose |
|---|---|---|
| VIEW | GET | Retrieve permitted records |
| CREATE | POST | Create a new record |
| UPDATE | PUT / PATCH | Modify an existing permitted record |
| DELETE | DELETE | Remove a permitted record |

Additional action permissions may be defined for operations that are not simple CRUD:

- **SUBMIT** — Submit code, lab work, assessment attempts, or contest entries
- **ATTEMPT** — Start/complete an assessment
- **PARTICIPATE** — Participate in an authorized contest
- **SOLVE** — Submit a solution for a problem

### API Permission Matrix

| API / Resource | SUPER_ADMIN | FINANCE_TEAM | HOD | MANAGER | STUDENT |
|---|---|---|---|---|---|
| College API | Full CRUD | V* | — | — | — |
| Department API | Full CRUD | V* | V/U* | — | — |
| User API | Full CRUD | V/C* | V/C/U* | V/U* | V OWN |
| Course API | Full CRUD | V* | Full CRUD* | V/C/U* | V |
| Lab API | Full CRUD | V* | Full CRUD* | Full CRUD* | V / SUBMIT |
| Problem API | Full CRUD | — | Full CRUD* | V/C/U* | V / SOLVE |
| Assessment API | Full CRUD | — | Full CRUD* | V/C/U* | V / ATTEMPT |
| Group API | Full CRUD | V* | Full CRUD* | V/C/U* | V OWN / JOIN |
| Contest API | Full CRUD | V* | Full CRUD* | V/C/U* | V / PARTICIPATE |
| Contest Ranking API | V/Manage | V* | V/Manage* | V* | V |
| Code Submission API | V/Manage | — | V* | V* | CREATE / V OWN |
| Certificate API | Full CRUD | — | V/C/U* | V* | V OWN |
| Support / Ticket API | Full CRUD | V/C/U* | V/C/U* | V/C/U* | C / V OWN |
| Notification API | Full CRUD | V/C/U* | V/C/U* | V/C/U* | V OWN |

`*` = permission is additionally restricted by the user's authorized data scope.

### Important Security Rule

The frontend must never be the final authorization layer.

Even if a page or button is hidden from a user, the backend must independently reject unauthorized API requests.

```text
Frontend Restriction
        +
Backend Authorization
        +
Database Scope Validation
        =
RBAC Enforcement
```

---

## 7. Database Authorization & Data Isolation

This section defines how DEZOLVER enforces database-level authorization after a user's role and permission have been validated.

Platform access determines what a user can do. Database authorization determines which records the user can perform that action on.

### Database Authorization Flow

```text
User Request
    ↓
Authentication
    ↓
Identify User + Role
    ↓
Permission Check
    ↓
Determine Data Scope
    ↓
Validate Requested Record
    ↓
Apply Scope Filter to Database Query
    ↓
Database Operation
    ↓
AuditLog where required
```

### Data Scope Hierarchy

```text
GLOBAL
  ↓
COLLEGE
  ↓
DEPARTMENT
  ↓
SECTION
  ↓
GROUP
  ↓
OWNERSHIP
```

| Role | Maximum data scope |
|---|---|
| SUPER_ADMIN | Global |
| FINANCE_TEAM | Assigned College |
| HOD | Assigned College + Department |
| MANAGER | Assigned College + Department + Section / permitted Group |
| STUDENT | Own records + permitted academic resources |

### Scope Rules by Role

#### 1. SUPER_ADMIN

SUPER_ADMIN operates at the platform level.

```text
SUPER_ADMIN
    ↓
GLOBAL SCOPE
    ↓
All Colleges
    ↓
All Departments
    ↓
All Sections / Groups
```

No college-level tenant restriction is applied to platform-wide administrative operations.

#### 2. FINANCE_TEAM

FINANCE_TEAM is restricted to its assigned college.

```text
FINANCE_TEAM
    ↓
Assigned collegeId
    ↓
Financial / subscription records
    ↓
Only records belonging to that college
```

The backend must ensure:

```text
resource.collegeId = authenticatedUser.collegeId
```

Records belonging to another college must not be returned or modified.

#### 3. HOD

HOD access is restricted to the assigned college and department.

```text
HOD
  ↓
collegeId
  ↓
departmentId
  ↓
Department Resources
```

The backend must validate:

```text
resource.collegeId = authenticatedUser.collegeId
AND
resource.departmentId = authenticatedUser.departmentId
```

This prevents an HOD from accessing another department's restricted academic data.

#### 4. MANAGER

MANAGER access is restricted to the assigned operational scope.

```text
MANAGER
  ↓
collegeId
  ↓
departmentId
  ↓
sectionId / permitted groupId
  ↓
Operational Records
```

The backend must validate the applicable collegeId, departmentId, sectionId, and groupId before allowing the requested operation.

#### 5. STUDENT

Student-owned records must be restricted to the authenticated student.

```text
STUDENT
  ↓
Authenticated userId
  ↓
studentId
  ↓
Own Records
```

Example:

```text
submission.studentId = authenticated user.studentId
```

A student must not be able to access another student's private submissions, assessment attempts, or other ownership-controlled records by changing an ID in the request.

### Tenant Isolation

DEZOLVER must prevent data from one college from being accessed through another college's user account.

Example hierarchy:

```text
College A
├── CSE
│   ├── Section A
│   └── Section B
└── ECE
    ├── Section A
    └── Section B

College B
└── CSE
    └── Section A
```

### Scope Validation Example

Request: `GET /students/{studentId}`

```text
Authenticated User
    ↓
Role = MANAGER
    ↓
Assigned College
    ↓
Assigned Department
    ↓
Assigned Section
    ↓
Query Student
    ↓
Student belongs to permitted scope?
    ├── YES → Return record
    └── NO  → Deny access
```

### Database Query Principle

Authorization must be applied to the database query itself wherever applicable.

**Without scope control:**

```sql
SELECT * FROM Student
WHERE id = requestedStudentId;
```

**With scope control:**

```sql
SELECT * FROM Student
WHERE id = requestedStudentId
  AND collegeId = authenticatedCollegeId
  AND departmentId = authenticatedDepartmentId
  AND sectionId = permittedSectionId;
```

The exact implementation will depend on the backend architecture, but the authorization requirement remains the same.

### Ownership Validation

Some resources are controlled by ownership rather than organizational scope:

- Student code submissions
- Assessment attempts
- Student-specific results
- Student certificates
- Personal profile information

```text
Requested Record
    ↓
Ownership Check
    ↓
Owner = Authenticated User?
    ├── YES → Allow
    └── NO  → Deny
```

### Final Database Authorization Rule

A valid role does not automatically grant access to every record. Every database operation must satisfy both the required permission and the user's authorized data scope.

```text
ROLE
  +
PERMISSION
  +
DATA SCOPE
  +
OWNERSHIP (where applicable)
  =
AUTHORIZED DATABASE ACCESS
```

---

## 8. Authorization Flow & Judge0 Integration

This section defines the end-to-end authorization flow used by DEZOLVER and explains how protected coding operations are handled when code execution is performed through Judge0.

### 8.1 DEZOLVER Authorization Flow

Every protected operation follows the same authorization sequence:

```text
User
  ↓
Authentication
  ↓
Identify User + Role
  ↓
Permission Check
  ↓
Data Scope Check
  ↓
Resource / Ownership Check
  ↓
Database Operation
  ↓
Audit Logging (where required)
```

#### Step 1 — Authentication

The system verifies the identity of the user through the authentication mechanism.

The authenticated user context must provide information required for authorization, such as:

- User ID
- Role
- College
- Department
- Section / Group where applicable

#### Step 2 — Role Check

The backend identifies the user's role:

- SUPER_ADMIN
- FINANCE_TEAM
- HOD
- MANAGER
- STUDENT

The role determines the set of permissions available to the user.

#### Step 3 — Permission Check

The backend verifies whether the role has the required permission for the requested operation.

Examples:

- VIEW
- CREATE
- UPDATE
- DELETE
- SUBMIT
- ATTEMPT
- PARTICIPATE
- SOLVE

If the required permission is not available: **Access Denied**.

#### Step 4 — Data Scope Check

If the permission is valid, the backend checks whether the requested resource belongs to the user's permitted scope.

- SUPER_ADMIN → GLOBAL
- FINANCE_TEAM → COLLEGE
- HOD → COLLEGE + DEPARTMENT
- MANAGER → COLLEGE + DEPARTMENT + SECTION / GROUP
- STUDENT → SELF + PERMITTED ACADEMIC DATA

#### Step 5 — Resource / Ownership Check

For resources that belong to a particular user, the backend additionally verifies ownership.

Example:

```text
Student
  ↓
View Code Submission
  ↓
Does submission.studentId match authenticated student?
  ├── YES → Allow
  └── NO  → Deny
```

#### Step 6 — Database Operation

Only after all authorization checks succeed does the backend perform the requested operation.

```text
Authorization Passed
    ↓
Database Query / Create / Update / Delete
```

#### Step 7 — Audit Logging

Sensitive administrative or system operations should be recorded in AuditLog according to the application's audit requirements.

### 8.2 Coding Submission Flow with Judge0

Judge0 is used as the code execution service. It executes submitted source code and returns the execution result to the DEZOLVER backend.

The user does not directly control the Judge0 execution request.

```text
Student
  ↓
Coding Page
  ↓
Submit Code
  ↓
DEZOLVER Backend
  ↓
Authentication
  ↓
Permission + Scope Check
  ↓
Create CodeSubmission
  ↓
Prepare Judge0 Submission
  ↓
Judge0
  ↓
Code Execution
  ↓
Execution Result
  ↓
DEZOLVER Backend
  ↓
Store / Process Result
  ↓
Lab / Assessment / Contest Result
```

### 8.3 Judge0 Authorization Boundary

Judge0 should be accessed through the DEZOLVER backend, rather than allowing the frontend to directly control execution requests.

```text
Frontend
  ↓
DEZOLVER Backend
  ↓
Authorization
  ↓
Judge0
```

The backend is responsible for determining:

- Whether the user is allowed to submit
- Which problem/lab/assessment/contest the submission belongs to
- Which language and execution limits apply
- What result is stored in DEZOLVER

### 8.4 Coding Submission Authorization

```text
Student
  ↓
Submit Code
  ↓
Is user authenticated?
  ↓ YES
Has SUBMIT permission?
  ↓ YES
Is the problem / lab / assessment / contest accessible?
  ↓ YES
Create CodeSubmission
  ↓
Send execution request to Judge0
```

If any authorization condition fails: **Submission Rejected**.

### 8.5 Judge0 Result Processing

Judge0 returns execution information to the backend.

```text
Judge0
  ↓
Execution Result
  ↓
DEZOLVER Backend
  ↓
Submission Result / Verdict
  ↓
CodeSubmission
  ↓
Relevant Lab / Assessment / Contest Result
```

The backend remains responsible for the application's final interpretation and persistence of the submission result.

### 8.6 Authorization Principle for Judge0

Judge0 performs code execution; DEZOLVER remains responsible for authentication, authorization, data ownership, submission records, and application-level results.

```text
DEZOLVER
├── User Authentication
├── RBAC
├── Scope Validation
├── CodeSubmission
├── Result Processing
└── Application Results
        ↓
      Judge0
        ↓
    Code Execution
```

---

## 9. Audit Logging & Security Rules

This section defines the audit and security requirements for DEZOLVER. It ensures that sensitive administrative and data-changing operations can be tracked and that authorization failures do not result in unauthorized database access.

### 9.1 Audit Logging

DEZOLVER uses the AuditLog table to record important actions performed within the platform.

Audit logging should be applied to sensitive operations, particularly administrative and data-modifying actions.

#### Actions to Consider for Audit Logging

- User creation, update, and deletion
- College creation, update, and deletion
- Department and section changes
- Role or access-related changes
- Subscription changes
- Invoice and payment-related administrative actions
- Course, lab, and academic-resource changes
- Assessment and contest administration
- Group membership or access changes
- Important configuration changes
- Other sensitive administrative operations

### 9.2 Audit Log Information

| Information | Purpose |
|---|---|
| User | Who performed the action |
| Action | What operation was performed |
| Resource / Entity | What was affected |
| Resource ID | Which specific record was affected |
| Timestamp | When the action occurred |
| Result / Status | Whether the operation succeeded or failed |
| Relevant Context | Additional information required for investigation |

The exact fields must follow the existing AuditLog table design.

### 9.3 Security Rules

#### Rule 1 — Backend Authorization is Mandatory

Frontend restrictions must not be treated as the security mechanism.

```text
Frontend Permission
        +
Backend Authorization
        +
Database Scope Validation
```

#### Rule 2 — Role Alone is Not Enough

Having the correct role does not automatically grant access to every record.

```text
Role
  ↓
Permission
  ↓
Data Scope
  ↓
Ownership where applicable
```

All applicable checks must pass before the operation is executed.

#### Rule 3 — Tenant Isolation

Users must not access restricted records belonging to another college.

- College A User → College A Data → Allowed
- College A User → College B Data → Denied

#### Rule 4 — Scope Isolation

Access must also respect department, section, and group boundaries.

- HOD → Assigned Department → Department Data
- Manager → Assigned Section / Group → Permitted Operational Data

#### Rule 5 — Ownership Protection

Student-owned resources must be protected using the authenticated user's identity.

Examples:
- Code submissions
- Assessment attempts
- Personal results
- Personal profile data
- Other student-specific records

```text
Requested Record
    ↓
Ownership Check
    ↓
Authenticated User = Owner?
    ├── YES → Allow
    └── NO  → Deny
```

### 9.4 Unauthorized Request Handling

```text
API Request
  ↓
Authentication
  ↓
Role Check
  ↓
Permission Check
  ↓
Scope / Ownership Check
  ↓
FAILED
  ↓
Access Denied
```

The database operation must not be executed.

Where required, the failed operation may also be recorded for security monitoring/audit purposes.

### 9.5 Sensitive Operations

| Operation | Security concern |
|---|---|
| User / Role changes | Can change who has access |
| College / Department changes | Can change data boundaries |
| Subscription / Payment changes | Financial data |
| Course / Problem administration | Academic content |
| Assessment administration | Student evaluation |
| Contest administration | Participation and scoring |
| Code submissions | Student work and execution |
| Group membership changes | Changes resource access |
| Administrative deletion | Potential data loss |
| Audit log access | Sensitive system information |

### 9.6 Core Security Principle

No database operation should be performed until authentication, role permission, data scope, and ownership requirements have been satisfied.

```text
AUTHENTICATE
    ↓
AUTHORIZE
    ↓
VALIDATE SCOPE
    ↓
VALIDATE OWNERSHIP
    ↓
DATABASE OPERATION
    ↓
AUDIT WHERE REQUIRED
```

---

## 10. RBAC Testing & Finalization Checklist

This section defines the tests required to verify that DEZOLVER RBAC is correctly enforced across roles, pages, APIs, database scopes, and ownership before the RBAC design is finalized.

### 10.1 Role Access Testing

| Test | Expected result |
|---|---|
| SUPER_ADMIN accesses permitted platform modules | ALLOW |
| FINANCE_TEAM accesses permitted finance modules | ALLOW |
| HOD accesses permitted department modules | ALLOW |
| MANAGER accesses permitted operational modules | ALLOW |
| STUDENT accesses permitted student modules | ALLOW |
| User attempts to access a module outside their role | DENY |

### 10.2 Permission Testing

Every CRUD permission must be verified:

- VIEW
- CREATE
- UPDATE
- DELETE

Example: If a role has COURSE → VIEW + UPDATE:

| API operation | Expected result |
|---|---|
| GET Course | ALLOW |
| PATCH Course | ALLOW |
| POST Course | DENY |
| DELETE Course | DENY |

### 10.3 Database Scope Testing

#### College Isolation

```text
User from College A
    ↓
Requests College B record
    ↓
DENY
```

#### Department Isolation

```text
HOD of CSE
    ↓
Requests ECE restricted record
    ↓
DENY
```

#### Section Isolation

```text
Manager of Section A
    ↓
Requests restricted Section B record
    ↓
DENY
```

#### Group Scope

```text
User
  ↓
Requests Group outside permitted scope
  ↓
DENY
```

### 10.4 Ownership Testing

| Scenario | Expected result |
|---|---|
| Student views own submission | ALLOW |
| Student views another student's submission | DENY |
| Student accesses own permitted assessment attempt | ALLOW |
| Student accesses another student's attempt | DENY |
| Student modifies another student's record | DENY |

### 10.5 API Authorization Testing

Each protected API should be tested with:

- Correct Role
- Correct Permission
- Correct Scope
- Correct Ownership where applicable

**Valid request:**

```text
Authenticated User
    ↓
Correct Role
    ↓
Permission Available
    ↓
Resource Within Scope
    ↓
ALLOW
```

**Invalid request:**

```text
Authenticated User
    ↓
Correct Role
    ↓
Permission Available
    ↓
Resource Outside Scope
    ↓
DENY
```

### 10.6 Judge0 / Coding Authorization Testing

| Scenario | Expected result |
|---|---|
| Authorized student submits permitted problem | ALLOW |
| Unauthenticated user submits code | DENY |
| Student submits to inaccessible problem | DENY |
| Student accesses own submission | ALLOW |
| Student accesses another student's private submission | DENY |
| Authorized submission is sent for execution | Judge0 execution allowed |
| Unauthorized submission | Judge0 request must not be created |

### 10.7 Audit Testing

Verify that required sensitive operations generate the appropriate AuditLog entry.

- User Management
- Subscription Changes
- Payment / Invoice Operations
- Academic Administration
- Contest Administration
- Important Configuration Changes

Test both:

- Successful sensitive operation → AuditLog created
- Unauthorized / security-sensitive operation → Rejected, and security event recorded where applicable

### Final RBAC Validation

The RBAC design is ready for implementation when the following statement is true:

> For every protected DEZOLVER operation, we can identify who can perform it, what permission is required, which data scope applies, whether ownership must be verified, and what happens when authorization fails.

```text
ROLE
  +
PERMISSION
  +
DATA SCOPE
  +
OWNERSHIP
  +
API ENFORCEMENT
  +
AUDIT
  ↓
FINALIZED RBAC
```

---

## Source Note

The original source is the 25-page PDF **`RBAC (1).pdf`**. This Markdown version reorganizes its text, tables, and flow diagrams into a searchable Markdown format. It preserves source statements and does not add new RBAC rules or resolve ambiguities in the source.
