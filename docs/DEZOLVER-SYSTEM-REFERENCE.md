# DEZOLVER System Reference

This reference consolidates the technical information contained in the current DEZOLVER PDF documentation set.

## 1. Platform stakeholders and scope

DEZOLVER defines five primary roles:

- `SUPER_ADMIN`: platform-wide administration with global scope.
- `FINANCE_TEAM`: college-level subscription, billing, and related administration; restricted to the assigned college.
- `HOD`: department-level academic administration; restricted to assigned college + department.
- `MANAGER`: section/group operational administration; restricted to assigned college + department + section/permitted group.
- `STUDENT`: self-service academic activity; access to own records and permitted academic resources.

The core authorization model is:

`Authentication → Role Check → Permission Check → Tenant/Data Scope Check → Ownership/Resource Check → Database Operation → Audit Logging where required`

A role determines **what** a user may do. Data scope and ownership determine **which records** the user may do it on.

## 2. Platform-level permissions

The RBAC document defines role permissions across College Management, Department Management, User Management, Subscriptions & Billing, Courses, Course Problems, Labs, Problems/Test Cases, Groups, Assessments, Contests, Contest Ranking, Code Submissions, Audit Logs, Certificates, Support/Tickets, Notifications, and Profile/Settings.

Permission types include standard CRUD plus domain actions:

- `VIEW`
- `CREATE`
- `UPDATE`
- `DELETE`
- `SUBMIT`
- `ATTEMPT`
- `PARTICIPATE`
- `SOLVE`

Permissions marked as scoped must additionally satisfy college, department, section, group, or ownership constraints.

## 3. Data-scope hierarchy

The documented scope hierarchy is:

`GLOBAL → COLLEGE → DEPARTMENT → SECTION → GROUP → OWNERSHIP`

Maximum role scopes:

| Role | Maximum scope |
|---|---|
| SUPER_ADMIN | Global |
| FINANCE_TEAM | Assigned College |
| HOD | Assigned College + Department |
| MANAGER | Assigned College + Department + Section / permitted Group |
| STUDENT | Own records + permitted academic resources |

Important enforcement rules:

- Finance queries must be filtered by the authenticated user's `collegeId`.
- HOD resources must match both `collegeId` and `departmentId`.
- Manager operations must additionally satisfy the relevant `sectionId` and/or permitted `groupId`.
- Student-owned resources must match the authenticated student's identity.
- Frontend visibility is not a security boundary; backend/API authorization and database-scope validation are mandatory.
- Authorization should be included in database queries wherever applicable rather than fetching broadly and filtering later.

## 4. Business flow

The documented end-to-end flow includes:

1. Super Admin performs college onboarding and platform-level setup.
2. Finance Team handles plan selection, subscription, active periods, and renewal.
3. HOD manages department academic structure: sections, managers, students, courses, problems, and groups.
4. Manager operates within the assigned section and manages students and permitted activities.
5. Groups connect collaborative activities such as labs and contests.
6. Students work on courses/problems, labs, and contests and submit code.
7. In the v2 flow, code submission is explicitly routed through the DEZOLVER backend and Judge0 before evaluation/results.
8. Lab submissions produce lab scores; contest submissions feed contest scoring and rank generation.

## 5. Judge0 integration

Judge0 is the code execution service. The documented trust boundary keeps Judge0 behind the DEZOLVER backend.

Submission flow:

`Student → Coding Page → Submit Code → DEZOLVER Backend → Authentication → Permission + Scope Check → Create CodeSubmission → Prepare Judge0 Submission → Judge0 → Code Execution → Execution Result → DEZOLVER Backend → Persist/Process Result → Lab/Assessment/Contest Result`

The backend is responsible for deciding:

- whether the user can submit;
- which problem/lab/assessment/contest the submission belongs to;
- allowed language and execution limits;
- persisted verdict/result;
- all authentication, RBAC, scope, ownership, and application-level result processing.

The frontend should not directly control Judge0 execution requests.

## 6. Database schema domains

### Identity, organization, and audit

- `College`
- `Department`
- `Section`
- `User`
- `RefreshSession`
- `AuditLog`

`College` holds identity/domain/status/seat information. `Department` belongs to a college and tracks codes, allocated/used seats, and HOD. `Section` belongs to a department. `User` includes email, password hash, name, role, status, MFA fields, organization scope (`collegeId`, `departmentId`, `sectionId`), academic batch, and timestamps.

### Commercial / subscription

- `Plan`
- `CollegeOnboarding`
- `PricingQuote`
- `Subscription`
- `Invoice`
- `PaymentTransaction`

These cover pricing plans, onboarding, quotes, purchased seats, subscription periods/renewal/grace state, invoicing, payment gateways, payment status, and transaction payloads.

### Courses, problems, and code execution

- `Course`
- `CourseSection`
- `CourseEnrollment`
- `CourseProblem`
- `Problem`
- `TestCase`
- `CodeSubmission`

Problems include difficulty, tags, time/memory limits, allowed languages, visibility/scope, and creator/tenant information. Test cases may be hidden and ordered. `CodeSubmission` stores the problem/student/language/code plus Judge0 token, verdict, execution time, memory use, failed-test index, stdout/stderr, and lifecycle timestamps.

### Labs

- `LabAssignment`
- `LabSubmission`

A lab assignment links a course and problem and stores scope, maximum score, and due date. Lab submissions link student + code submission and carry score, feedback, and submission time.

### Assessments

- `Assessment`
- `AssessmentProblem`

Assessments are scoped by college/department/group, have creator, title/description, type, duration, passing percentage, time window, and publish state.

### Contests

- `Contest`
- `ContestProblem`
- `ContestParticipant`
- `ContestScoreRank`

Contests store schedule/status, problem ordering/weights, participants, total score, penalty time, rank, and completion timestamps.

### Groups

- `Group`
- `GroupMember`
- `GroupLab`
- `GroupContest`
- `GroupCourseProblem`

Groups belong to college/department and can associate students with labs, contests, and course/problem resources.

### Certificates

- `CertificateTemplate`
- `Certificate`

Certificate templates store HTML/styles. Certificates include verification code, related student/course/contest, QR code URL, PDF URL, and issue time.

### Support and notifications

- `SupportTicket`
- `TicketComment`
- `Notification`

Tickets track requester/assignee, category, priority, status, description, resolution/escalation notes and timestamps. Notifications carry recipient, type, message, optional link, and read state.

## 7. Important relationships

The schema explicitly includes relationships such as:

- college → departments
- department → sections
- user → refresh sessions
- user → audit logs
- onboarding/plan/user → pricing quotes
- plan/college → subscriptions
- subscription/college → invoices and payments
- college/department/user → courses
- course ↔ sections
- course/student → enrollments
- course/problem → lab assignments
- lab/student/code submission → lab submissions
- problem → test cases and code submissions
- college/department/group/user → assessments
- college/user → contests
- contest ↔ problems, participants, scores
- college/user → groups
- group ↔ members, labs, contests
- college/user → certificate templates
- template/college/student → certificates
- college/user → support tickets
- ticket/user → comments
- user → notifications

## 8. Enumerations shown in the schema

The schema PDF includes enums for:

- `Role`: `SUPER_ADMIN`, `HOD`, `MANAGER`, `STUDENT`
- `EntityStatus`: `ACTIVE`, `SUSPENDED`, `DEACTIVATED`, `ARCHIVED`
- `SubscriptionStatus`: `ACTIVE`, `GRACE_PERIOD`, `SUSPENDED`, `CANCELLED`
- `InvoiceStatus`: `PAID`, `OPEN`, `VOID`, `UNCOLLECTIBLE`
- `PaymentStatus`: `SUCCESS`, `FAILED`, `PENDING`, `REFUNDED`
- `PaymentGateway`: `STRIPE`, `RAZORPAY`, `MANUAL_BANK_TRANSFER`
- `ContentScope`: `DEPT_WIDE`, `SECTION_ONLY`, `GLOBAL`
- `PublishStatus`: `DRAFT`, `PUBLISHED`, `ARCHIVED`
- `AssessmentType`: `CODING`
- `ProblemDifficulty`: `EASY`, `MEDIUM`, `HARD`
- `JudgeVerdict`: `PENDING`, `RUNNING`, `ACCEPTED`, `WRONG_ANSWER`, `TIME_LIMIT_EXCEEDED`, `MEMORY_LIMIT_EXCEEDED`, `RUNTIME_ERROR`, `COMPILATION_ERROR`
- `ContestStatus`: `UPCOMING`, `LIVE`, `ENDED`
- `TicketStatus`: `SUBMITTED`, `RESOLVED`, `ESCALATED`, `MORE_INFO_REQUESTED`, `CLOSED`, `REOPENED`
- `TicketPriority`: `LOW`, `MEDIUM`, `HIGH`, `URGENT`
- `TicketCategory`: `TECHNICAL`, `BILLING`, `ACADEMIC`, `ACCOUNT`
- `NotificationType`: `SYSTEM`, `ASSIGNMENT`, `GRADE`, `CONTEST`, `BILLING`, `TICKET`

Note: the RBAC PDF includes `FINANCE_TEAM` as a DEZOLVER role, while the schema PDF's displayed `Role` enum extraction lists `SUPER_ADMIN`, `HOD`, `MANAGER`, and `STUDENT`. This is a source-level discrepancy that should be reconciled before implementation.

## 9. Audit and security requirements

Sensitive administrative/data-changing operations should be auditable, including user/role changes, organization structure changes, subscriptions, invoices/payments, academic-resource administration, assessment/contest administration, group membership, and important configuration changes.

Core security rules documented in RBAC:

- Backend authorization is mandatory.
- Role alone is insufficient.
- Enforce tenant isolation between colleges.
- Enforce department/section/group scope isolation.
- Enforce ownership for student-specific resources.
- Failed authorization must prevent the database operation.
- Security-sensitive failures may be logged where required.

## 10. RBAC testing checklist

The RBAC document requires testing across:

- role access;
- CRUD/action permissions;
- college isolation;
- department isolation;
- section isolation;
- group scope;
- ownership controls;
- API authorization;
- Judge0/coding authorization;
- audit logging.

The final validation criterion is that every protected operation identifies the role, permission, scope, ownership requirements, API enforcement, audit behavior, and denial outcome.

## 11. Page-to-database mapping documents

The two page-to-database mapping PDFs are organized by stakeholder:

- Super Admin
- Finance Team
- HOD
- Manager
- Student

Their stated purpose is to map every Figma/application page to the database entities supporting that page. Most individual page/entity relationships are embedded visually and are not present in machine-extractable text, so the PDFs remain authoritative for the detailed visual mapping.

## 12. Source versions

Current source PDFs:

- `Business Flow.pdf` — original business flow
- `Business Flow v2.pdf` — updated business flow with explicit DEZOLVER Backend/Judge0 paths and richer lab/contest scoring flow
- `Page - DB Mapping.pdf` — original page/database mapping
- `Page - DB Mapping v2.pdf` — updated mapping
- `RBAC (1).pdf` — detailed RBAC/security specification
- `scheema.pdf` — database ER/schema design
