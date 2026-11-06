# 4. Course & Enrollment

**Document type:** Functional Specification  
**Version:** 1.0 — Adopted target baseline  
**Status:** Product decisions adopted (COURSE-D01–COURSE-D40); engineering implementation gates remain  
**Scope:** Course creation and management, course scope, section assignment, student enrollment, course visibility, course-problem association, and course completion data

> **Specification principle:** This baseline reconciles Business Flow (original/v2), Page–DB Mapping (original/v2), RBAC, target schema, and reviewed legacy Module 04 documentation. Sections 24–26 and ADOPTED-DECISIONS.md are normative where earlier draft passages retain TBD wording.

---

## 1. Purpose

Define how DEZOLVER organizes academic learning into Courses, makes courses available to Departments and Sections, associates Students with Courses, and tracks enrollment/completion.

Courses connect:
- College and Department
- Sections
- Students through CourseEnrollment
- Problems through CourseProblem
- Group-related course problems through GroupCourseProblem
- Labs and Assessments, which are specified in their own modules

## 2. Scope

### Included
- Course entity and metadata
- Course creation and management
- Course status and publication
- Course scope
- Course-to-Section assignment
- Course enrollment and Student My Courses access
- Course-to-Problem association
- Course completion field and unresolved semantics
- Role-based access and organizational scope
- Source discrepancies and decisions required

### Excluded / owned by other modules

| Topic | Detailed specification owner |
|---|---|
| User identity, roles, and section assignment | Module 1 — Authentication & User Lifecycle |
| College, Department, and Section lifecycle | Module 2 — College / Department / Section Management |
| Subscription and seat capacity enforcement | Module 3 — Subscription & Seat Management |
| Problem creation, publication, and test cases | Module 5 — Problem Management |
| Code submission and Judge0 | Module 6 — Code Submission & Judge0 |
| Test case evaluation | Module 7 — Test Case Evaluation |
| Lab assignment and lab scoring | Module 8 — Labs & Lab Scoring |
| Assessments | Module 9 — Assessments |
| Groups and GroupCourseProblem behavior | Module 10 — Groups |
| Notifications | Module 13 — Notifications |
| Audit event catalogue | Module 15 — Audit Logging |
| Canonical state transitions | Module 17 — Functional State Machines |
| Cross-module edge cases | Module 18 — Edge Cases & Failure Handling |

---

## 3. Source documents and interpretation

### 3.1 Business Flow (original and v2)

The business flow establishes this academic context:

- HOD manages Department, Sections, Managers, Students, Courses, Problems, and Groups.
- Manager operates within their assigned Section and manages students and activities there.
- Student accesses Courses and Problems, writes code, and submits it.
- Course and Problem are connected to the code submission workflow.

This confirms Courses as part of the HOD-managed department academic structure and the Student learning journey. It does **not** define detailed course creation, enrollment, or completion workflows.

### 3.2 Page-to-DB Mapping (original and v2)

| Role | Page | Course-related entities mapped |
|---|---|---|
| HOD | Dashboard | Course |
| HOD | Courses & Labs | Course, CourseSection, CourseEnrollment, CourseProblem, GroupCourseProblem |
| HOD | Students | CourseEnrollment |
| Manager | Dashboard | Course |
| Manager | Courses & Labs | Course, CourseSection, CourseEnrollment, CourseProblem, GroupCourseProblem |
| Manager | Students | CourseEnrollment |
| Student | Dashboard | CourseEnrollment |
| Student | My Courses | Course, CourseSection, CourseEnrollment, CourseProblem, GroupCourseProblem, Problem, Section |

The original and v2 mappings confirm page/entity associations, not complete lifecycle behavior. RBAC platform/page/API matrices were also reviewed; conflicts are resolved in Sections 15 and 24.

### 3.3 Schema reconstruction

The schema identifies Course, CourseSection, CourseEnrollment, CourseProblem, and GroupCourseProblem. It also defines ContentScope and PublishStatus enums, detailed in Section 7.

---

## 4. Roles and access scope

| Role | Source-described Course responsibility | Expected scope |
|---|---|---|
| **SUPER_ADMIN** | Platform-level administration; Course permissions are not explicitly detailed in the Course page mapping. | Platform-wide, subject to final permissions |
| **FINANCE_TEAM** | No Course management responsibility shown. | Course access not defined by these sources |
| **HOD** | Department academic structure includes Courses; HOD Dashboard and Courses & Labs pages are mapped. | Assigned College and Department |
| **MANAGER** | Manager Dashboard and Courses & Labs pages are mapped; Manager operates within assigned Section. | Assigned College, Department, and Section |
| **STUDENT** | Student Dashboard and My Courses pages are mapped. | Own enrollment and authorized academic content |

**Source limitation:** these documents establish role context and page mapping, not complete create/read/update/delete/publish/enroll permissions.

### 4.1 Authorization requirements

- Course operations must validate authenticated identity, role, College, Department, and Section scope where applicable.
- HOD must not access Course records outside the permitted College/Department.
- Manager must not manage or view restricted Course records for unrelated Sections.
- Student must not access another Student's enrollment or progress records.
- Client-supplied IDs must not override server-resolved organization scope.

**TBD:** action-level permissions for creating, editing, publishing, archiving, assigning Sections, enrolling Students, and overriding completion.

---

## 5. Functional requirements

### COURSE-001 — Maintain Course records

**Status:** Source-confirmed schema.

DEZOLVER must represent Courses using the Course entity.

Schema fields:
- id
- collegeId
- departmentId
- createdById
- title
- description
- code
- scope
- status
- createdAt
- updatedAt

A Course is associated with a College, Department, and creator.

**TBD:** required/optional field rules, course duration, academic term/year, and whether additional Course metadata is required.

### COURSE-002 — Create a Course within an organizational scope

**Status:** Source-confirmed hierarchy/schema; workflow TBD.

Course includes collegeId, departmentId, and createdById.

**Derived consistency rule:** the Course Department must belong to the Course College, and the creator must be authorized to create Courses within that scope.

The original and v2 Business Flow identify Courses within HOD's Department academic management; v2 adds backend/Judge0 context but neither defines the automatic enrollment algorithm.

**TBD:**
- Whether HOD is the primary/only Course creator.
- Whether SUPER_ADMIN can create Courses directly.
- Whether Manager may create/edit Courses or only manage course activities.
- Whether a Course may be created without a Department.
- Whether cross-department or college-wide creation is permitted.

### COURSE-003 — Maintain course title, description, and code

**Status:** Source-confirmed schema.

Course stores title, description, and code.

**TBD:** required fields, code uniqueness, case sensitivity, normalization, format restrictions, and uniqueness scope (global, College, or Department).

### COURSE-004 — Support Course scope

**Status:** Source-confirmed enum.

The schema defines ContentScope:
- DEPT_WIDE
- SECTION_ONLY
- GLOBAL

Course contains a scope field.

**TBD:** exact visibility meaning:
- Does DEPT_WIDE make the Course available to every Section in its Department?
- Does SECTION_ONLY require one or more CourseSection assignments?
- Does GLOBAL mean all users, all Colleges, or global within one College?
- Is GLOBAL allowed when the Course also has a specific collegeId and departmentId?
- Can scope be changed after enrollment or student activity exists?

Do not implement scope behavior solely by interpreting enum names.

### COURSE-005 — Assign Courses to Sections

**Status:** Source-confirmed schema and page mapping.

CourseSection contains:
- courseId
- sectionId
- createdAt

The ER model associates Courses and Sections. A Course may be associated with multiple Sections and a Section may receive multiple Courses.

**Derived requirement:** Course-section association must validate the Section's organizational scope, unless an approved cross-scope rule exists.

**TBD:**
- Whether Section assignment is mandatory for SECTION_ONLY Courses.
- Whether a Course may be assigned to multiple Sections.
- Who can assign/remove a Section.
- Whether assignment automatically enrolls every Student in that Section.
- Whether removal from a Section affects existing enrollments/progress.
- Whether duplicate CourseSection links are prohibited.

### COURSE-006 — Course publication status

**Status:** Source-confirmed schema enum; visibility behavior TBD.

Course uses PublishStatus:
- DRAFT
- PUBLISHED
- ARCHIVED

**TBD:**
- Whether DRAFT is visible only to authorized staff.
- Whether PUBLISHED makes a Course visible to eligible Students.
- Whether ARCHIVED Courses remain visible to previously enrolled Students.
- Whether an archived Course allows submissions/progress updates.
- Who may publish/archive.
- Whether publication requires validation or approval.

### COURSE-007 — HOD Course management

**Status:** Source-confirmed page mapping and business flow.

HOD Dashboard maps Course, and HOD Courses & Labs maps Course-related entities.

HOD Course operations must be restricted to the HOD's assigned College and Department.

**TBD:** exact HOD actions (create, edit, publish, archive, assign Sections, manage enrollment, view progress), and whether HOD can view Courses created by other HODs in the same Department.

### COURSE-008 — Manager Course access

**Status:** Source-confirmed page mapping; action permissions TBD.

Manager Dashboard and Courses & Labs pages map Course and course-related entities. Business Flow v2 describes Manager as operating only within the assigned Section.

**Derived scope requirement:** Manager Course access must be restricted to Courses/Sections within the Manager's authorized scope.

**TBD:** whether Manager can create/edit Courses, assign Problems, enroll Students, or only view/manage students' course activities.

### COURSE-009 — Student My Courses

**Status:** Source-confirmed page mapping.

Student My Courses maps Course, CourseSection, CourseEnrollment, CourseProblem, GroupCourseProblem, Problem, and Section.

The Student-facing course list must respect approved enrollment and scope rules.

**TBD:**
- Whether My Courses lists only CourseEnrollment records or also Section-assigned Courses.
- Whether published but not enrolled Courses can appear as discoverable Courses.
- Whether Students can self-enroll.
- Whether Courses are filtered by academic batch, Section, or Department.
- Whether Students retain access to archived Courses.

### COURSE-010 — Maintain CourseEnrollment records

**Status:** Source-confirmed schema.

CourseEnrollment fields:
- id
- courseId
- studentId
- enrolledAt
- completedAt

The schema relationship is Course 1:N CourseEnrollment and User/Student 1:N CourseEnrollment.

**Derived requirement:** every enrollment must reference a valid Course and Student/User.

**TBD:**
- Enrollment creation trigger (manual staff enrollment, automatic Section enrollment, self-enrollment, or other).
- Whether Student can enroll outside their assigned Section/Department.
- Whether duplicate enrollments are prohibited.
- Whether enrollment can be withdrawn, removed, or transferred.
- Whether Course enrollment capacity exists.
- Whether enrollment is subject to Subscription seat limits (Module 3).

### COURSE-011 — Enrollment access and eligibility

**Status:** TBD / Decision Required.

The source documents do not define eligibility rules for enrolling a Student.

Potential eligibility factors that need approval:
- Student account status
- College/Department/Section assignment
- Course scope
- Course status
- Subscription/seat availability
- Academic batch or term
- Course start/end dates (not shown in the Course schema)

No enrollment eligibility rule should be inferred beyond source-confirmed role and organization scope until approved.

### COURSE-012 — Enrollment lifecycle

**Status:** Source-confirmed enrollment timestamps; lifecycle TBD.

CourseEnrollment has enrolledAt and completedAt. No enrollment status enum or withdrawal timestamp is shown.

**TBD:**
- Whether enrollment is permanent, removable, or reactivatable.
- Whether completion is represented only by completedAt.
- Whether enrollment can be incomplete, withdrawn, failed, or expired.
- Whether Student can repeat a Course and create a new enrollment.
- Whether re-enrollment resets progress.
- Whether historical enrollments are retained.

### COURSE-013 — Course completion tracking

**Status:** Source-confirmed completedAt field; completion rule TBD.

CourseEnrollment includes completedAt, which can record a completion timestamp.

The source documents do not define:
- What constitutes Course completion.
- Whether all CourseProblems must be solved.
- Whether Labs, Assessments, or Contests contribute.
- Whether completion is automatic or staff-approved.
- Whether completion can be reversed.
- Whether completion triggers a Certificate.

**Important:** Course completion and individual Problem/Room progress are not necessarily the same concept. The canonical completion model must be coordinated with Modules 6–9, 12, and 17.

### COURSE-014 — Associate Problems with Courses

**Status:** Source-confirmed schema and page mapping.

CourseProblem contains:
- courseId
- problemId
- createdAt

The ER model associates Course with Problems through CourseProblem.

**Derived requirement:** Course and Problem must be valid and accessible within the permitted organization/scope.

**TBD:**
- Who can add/remove Problems from a Course.
- Whether a Problem may be linked to multiple Courses.
- Whether ordering is required (no order field is shown).
- Whether CourseProblem supports required/optional Problems.
- Whether Problem availability is controlled by Course or Problem status.
- Whether adding/removing a Problem affects existing progress.

### COURSE-015 — GroupCourseProblem relationship

**Status:** Source-confirmed schema/page mapping; behavior belongs primarily to Module 10.

GroupCourseProblem contains:
- groupId
- courseId
- problemId
- createdAt

The Page-to-DB Mapping includes GroupCourseProblem on HOD, Manager, and Student course-related pages.

**TBD:** whether this represents group-specific assignment/visibility, a supplemental course-problem relationship, or another rule. Do not treat it as identical to CourseProblem without confirming its semantics.

### COURSE-016 — Course list, search, and filtering

**Status:** TBD / Decision Required.

The source identifies Course-related pages but does not specify UI behavior.

Confirm:
- Search fields (title, code, creator, Department).
- Filters (status, scope, Department, Section, enrollment state).
- Sorting and pagination.
- Course list columns.
- Whether HOD and Manager have different views.
- Whether Student My Courses supports search, progress filtering, or sorting.

### COURSE-017 — Course updates and historical consistency

**Status:** TBD / Decision Required.

The schema has updatedAt, but the source does not define how edits affect already-enrolled Students.

Confirm:
- Whether title/description/code/scope can change after publication.
- Whether changing scope or Section assignment affects existing enrollment.
- Whether removing a Problem removes it from student history.
- Whether archived Course data remains available for reporting.
- Whether historical progress and submissions remain associated after Course edits.

### COURSE-018 — Course access and subscription/seat constraints

**Status:** Cross-module dependency; policy TBD.

Module 3 contains Subscription and seat-capacity fields. Course source documents do not define what happens when:
- College Subscription expires, is suspended, or is cancelled.
- College/Department seat capacity is full.
- Student seat entitlement is no longer valid.
- Subscription state changes while Student is enrolled.

Course access/enrollment must use approved Module 3 entitlement rules. This module does not independently define them.

---

## 6. High-level workflows

### 6.1 Course creation and publication — source-grounded outline

Business Flow identifies Courses under HOD's Department academic management.

Conceptual outline:
1. Authorized academic staff creates a Course.
2. Course is associated with College and Department.
3. Course metadata is saved.
4. Course scope is set.
5. Course may be associated with Sections through CourseSection.
6. Course status is set to DRAFT, PUBLISHED, or ARCHIVED.
7. Course may be associated with Problems through CourseProblem.
8. Eligible Students access the Course using the approved enrollment/visibility mechanism.

**Exact ordering, required steps, permissions, and publish validation are TBD.**

### 6.2 Student course access — source-grounded outline

1. Student opens My Courses.
2. DEZOLVER resolves Student identity and organization scope.
3. System identifies Courses available under approved enrollment and scope rules.
4. Student opens an authorized Course.
5. Student accesses Course-associated Problems and learning activities.

The source does not define self-enrollment or automatic Section enrollment.

### 6.3 Course enrollment — conceptual record flow

1. An approved enrollment trigger occurs (trigger TBD).
2. CourseEnrollment links Student and Course.
3. enrolledAt records enrollment time.
4. Student course access is evaluated using enrollment, Course status, scope, and organization rules.
5. completedAt is set only after the approved completion rule is satisfied.

**Enrollment trigger, access conditions, and completion criteria require decisions.**

---

## 7. Data model reference

### 7.1 Course

| Field | Schema type | Meaning |
|---|---|---|
| id | varchar | Course identifier |
| collegeId | varchar | College scope |
| departmentId | varchar | Department scope |
| createdById | varchar | Course creator |
| title | varchar(255) | Course title |
| description | text | Course description |
| code | varchar(50) | Course code |
| scope | ContentScope | DEPT_WIDE / SECTION_ONLY / GLOBAL |
| status | PublishStatus | DRAFT / PUBLISHED / ARCHIVED |
| createdAt, updatedAt | timestamp | Record timestamps |

### 7.2 CourseSection

| Field | Schema type | Meaning |
|---|---|---|
| courseId | varchar | Course reference |
| sectionId | varchar | Section reference |
| createdAt | timestamp | Association timestamp |

### 7.3 CourseEnrollment

| Field | Schema type | Meaning |
|---|---|---|
| id | varchar | Enrollment identifier |
| courseId | varchar | Course reference |
| studentId | varchar | Student/User reference |
| enrolledAt | timestamp | Enrollment timestamp |
| completedAt | timestamp | Course completion timestamp; meaning TBD |

### 7.4 CourseProblem

| Field | Schema type | Meaning |
|---|---|---|
| courseId | varchar | Course reference |
| problemId | varchar | Problem reference |
| createdAt | timestamp | Association timestamp |

### 7.5 GroupCourseProblem

| Field | Schema type | Meaning |
|---|---|---|
| groupId | varchar | Group reference |
| courseId | varchar | Course reference |
| problemId | varchar | Problem reference |
| createdAt | timestamp | Association timestamp |

### 7.6 Relevant enums

**ContentScope**
- DEPT_WIDE
- SECTION_ONLY
- GLOBAL

**PublishStatus**
- DRAFT
- PUBLISHED
- ARCHIVED

The schema provides these values but does not fully define their transition/access semantics.

---

## 8. Validation and consistency requirements

These are derived safeguards to preserve schema and organization consistency. Their business policy details remain subject to approval.

| Rule ID | Validation | Status |
|---|---|---|
| COURSE-V01 | Course must reference a valid College and Department under approved scope rules. | Derived |
| COURSE-V02 | Course Department must belong to Course College. | Derived |
| COURSE-V03 | CourseSection must reference an existing Course and Section. | Derived |
| COURSE-V04 | CourseSection must not cross organization scope unless explicitly allowed by approved GLOBAL rules. | Derived; global behavior TBD |
| COURSE-V05 | CourseEnrollment must reference a valid Course and eligible Student. | Derived |
| COURSE-V06 | Student must not access another Student's enrollment/progress record. | Derived from RBAC scope |
| COURSE-V07 | CourseProblem must reference a valid Course and Problem. | Derived |
| COURSE-V08 | GroupCourseProblem must reference a valid Group, Course, and Problem. | Derived |
| COURSE-V09 | Course visibility must be evaluated server-side using status, scope, Section, and enrollment rules. | Derived |
| COURSE-V10 | Completion timestamp must be set only according to the approved completion rule. | TBD |
| COURSE-V11 | Duplicate enrollment and association behavior must be defined and enforced. | TBD |
| COURSE-V12 | Enrollment must respect approved Subscription/seat limits. | Cross-module; Module 3 |

---

## 9. Source gaps, conflicts, and risks

### 9.1 Course scope semantics are underspecified

Schema includes DEPT_WIDE, SECTION_ONLY, and GLOBAL, but no definitions.

**Risk:** incorrect visibility could expose academic content across Sections or Colleges.

**Decision required:** define the exact access matrix and clarify GLOBAL scope's relationship with College/Department.

### 9.2 Enrollment trigger is not documented

CourseEnrollment exists, but source does not define how it is created.

**Decision required:** manual staff enrollment, automatic Section enrollment, self-enrollment, or multiple mechanisms.

### 9.3 CourseSection vs CourseEnrollment distinction

CourseSection associates Course with Section; CourseEnrollment associates Course with Student.

Source does not say whether Section assignment automatically creates CourseEnrollment records.

**Decision required:** determine whether Section assignment controls eligibility, creates enrollments, or is itself the canonical course access assignment.

### 9.4 CourseEnrollment has no status field

Only enrolledAt and completedAt are shown.

**Risk:** withdrawal, suspension, failed completion, repeat enrollment, and historical states may be difficult to represent.

**Decision required:** confirm whether timestamps are sufficient or schema needs enrollment status/withdrawal fields.

### 9.5 Course completion semantics are not defined

completedAt exists, but no completion formula or trigger is documented.

**Decision required:** determine whether completion depends on Problems, Labs, Assessments, milestones, or staff approval.

### 9.6 CourseProblem ordering and weighting are not represented

CourseProblem has Course, Problem, and createdAt only in the supplied schema reconstruction.

**Decision required:** confirm whether Courses require problem ordering, required/optional status, points/weight, due dates, or scheduled availability. Schema changes may be required if so.

### 9.7 Manager course permissions are unclear

Manager Courses & Labs is mapped, but Manager Course CRUD permissions are not defined.

**Decision required:** Manager view-only vs course management vs student activity management.

### 9.8 Course status behavior is not specified

DRAFT, PUBLISHED, and ARCHIVED exist, but visibility, editability, enrollment, and submission behavior per status are not defined.

**Decision required:** define status behavior and transitions.

### 9.9 Subscription/seat constraints must be coordinated

Course enrollment may consume seats, but the canonical seat-counting trigger is still open in Module 3.

**Decision required:** align Course enrollment with Module 3's approved seat-counting policy.

---

## 10. Adopted product decision register

All product decisions COURSE-D01–COURSE-D40 are adopted for Module 4 v1. Full rationale, source reconciliation, and implementation implications are recorded in ADOPTED-DECISIONS.md. No COURSE-D01–D40 remains an open product decision. Remaining work is engineering/schema/RBAC integration and QA execution, listed in Sections 25–26.

## 11. Acceptance checklist

Module 4 is implementation-ready when:
- [ ] Course creation permissions and ownership are explicit.
- [ ] Course metadata validation and code uniqueness are defined.
- [ ] Scope enum semantics are approved.
- [ ] Course status transitions and visibility are defined.
- [ ] Course-to-Section assignment rules are finalized.
- [ ] Enrollment creation, eligibility, duplicate handling, and withdrawal are defined.
- [ ] CourseSection vs CourseEnrollment behavior is explicit.
- [ ] Student My Courses visibility is defined.
- [ ] Course completion criteria and completedAt updates are finalized.
- [ ] CourseProblem association and ordering/weight requirements are approved.
- [ ] HOD and Manager permissions are explicitly scoped.
- [ ] Subscription/seat checks are coordinated with Module 3.
- [ ] Course audit and notification requirements are assigned to their modules.

---

## 12. Dependencies and interfaces

- **Module 1 — Authentication & User Lifecycle:** role, user status, and College/Department/Section assignments.
- **Module 2 — College / Department / Section Management:** course organizational hierarchy and Section assignment.
- **Module 3 — Subscription & Seat Management:** enrollment seat eligibility and subscription access.
- **Module 5 — Problem Management:** CourseProblem links and Problem lifecycle.
- **Module 8 — Labs & Lab Scoring:** course-linked LabAssignment and course completion contribution, if any.
- **Module 9 — Assessments:** assessment participation and completion contribution, if any.
- **Module 10 — Groups:** GroupCourseProblem behavior.
- **Module 12 — Certificates:** whether Course completion can issue a certificate.
- **Module 13 — Notifications:** Course/enrollment notifications.
- **Module 15 — Audit Logging:** Course and enrollment audit events.
- **Module 17 — Functional State Machines:** Course and CourseEnrollment transitions.
- **Module 18 — Edge Cases & Failure Handling:** concurrent enrollment, Course changes, scope changes, and completion edge cases.

---

## 13. Source references

Repository source-text documents:
- [Business Flow v2.md](../../docs/source-text/Business%20Flow%20v2.md)
- [Page - DB Mapping v2.md](../../docs/source-text/Page%20-%20DB%20Mapping%20v2.md)
- [scheema.md](../../docs/source-text/scheema.md)

Original Drive PDFs:
- [Business Flow v2.pdf](https://drive.google.com/file/d/1rE9q1h0u9FTVLJmlP2Wl7WWfKFqsETgL/view)
- [Page - DB Mapping v2.pdf](https://drive.google.com/file/d/1UdzowkDPP85PNI0-GQ2k4cEmwIIAZYcw/view)
- [scheema.pdf](https://drive.google.com/file/d/10_LGfTsQ2eGmbJM5tQE9Pkmlwhft1lAI/view)

**Review gate:** Module 4 only. Do not begin Module 5 until the user reviews this document and explicitly approves proceeding.


---

# Functional Specification Standard v1.0 — compliance supplement

This supplement expands the existing source-based draft into function-level contracts. The original source summary, COURSE-D01–D27 decisions, and gaps remain active. Source silence is not converted into approved behavior.

## 14. Evidence labels and terminology

- **Source-confirmed:** explicit in Business Flow v2/original, Page-to-DB Mapping v2/original, RBAC, or schema.
- **Derived:** logical consistency/security behavior based on source facts.
- **Adopted target behavior:** v1 policy recorded as COURSE-D01–COURSE-D40.
- **Engineering gate:** schema migration, canonical RBAC, events, implementation, and QA verification remain pending.
- **Legacy documentation reviewed:** legacy Module 04 README/SPECIFICATION were reviewed as historical evidence; this is not a claim that legacy source code was re-executed or re-verified.
- Legacy-only CourseRoom/Room ordering, Syllabus/overlays, Learning Paths/PathProgress, and legacy API contracts are not imported into target Module 4.

| Term | Meaning | Evidence |
|---|---|---|
| Course | Academic resource represented by Course | Schema |
| CourseSection | Course-to-Section association | Schema |
| CourseEnrollment | Student-to-Course association | Schema |
| CourseProblem | Course-to-Problem association | Schema |
| GroupCourseProblem | Group + Course + Problem association; Module 10 owns group semantics | Schema |
| Scope | ContentScope value on Course | Schema; meaning TBD |
| Publication status | DRAFT / PUBLISHED / ARCHIVED | Schema; transitions TBD |
| Completion | completedAt timestamp on CourseEnrollment | Schema; criteria TBD |

### 14.1 Source version conflict matrix

| Topic | Business Flow v2/original | Page mapping v2 | Original page mapping | RBAC/schema | Interpretation |
|---|---|---|---|---|---|
| HOD Course role | HOD manages Courses in Department academic structure | HOD Dashboard/Courses & Labs map Course | Similar role context | HOD Course Full CRUD | HOD academic ownership is supported; detailed workflow TBD |
| Manager Course role | Manager manages assigned Section activities | Manager Dashboard/Courses & Labs map Course | Similar | Platform matrix Create/Update*; page/API matrix V/C/U* | Manager create/update supported by matrices, but scoped action policy needs decision |
| Manager delete | Not defined | Page mapping is not action permission | Not defined | Manager delete not explicitly granted | Do not grant delete/archive by inference |
| Student enrollment | Course-to-Problem learning flow only | My Courses maps CourseEnrollment/CourseSection/CourseProblem/GroupCourseProblem | Mapping is narrower | Platform matrix View/Enroll; page/API matrix View | Self-enrollment conflict; not approved |
| Finance Team Course view | Not mentioned | No Finance Course page | No Finance Course page | Platform matrix View | View permission is source-confirmed; route/UI TBD |
| SUPER_ADMIN Course access | Platform administration | No Course-specific page | No Course-specific page | Full CRUD | Global permission source-confirmed; route TBD |
| Scope | Not defined | Course mapped | Course mapped | DEPT_WIDE/SECTION_ONLY/GLOBAL enum | Enum exists; semantics TBD |
| CourseProblem | Course/Problem connection | v2 mapping includes CourseProblem and GroupCourseProblem | Original Courses & Labs mapping omits them | Schema has both relations | v2 expands mapped entities |
| Course status | Not defined | Course mapped | Course mapped | DRAFT/PUBLISHED/ARCHIVED | Enum exists; access/transition TBD |
| Enrollment lifecycle | Not defined | CourseEnrollment mapped | CourseEnrollment mapped | only enrolledAt/completedAt fields | Trigger, withdrawal, repeat, completion TBD |

## 15. Role, action, and scope matrix

This is the adopted v1 authorization baseline. Backend authorization is authoritative.

| Role | Course view | Create/update | Publish/archive/restore | Assign Sections | CourseProblem | Enrollment access |
|---|---|---|---|---|---|---|
| SUPER_ADMIN | All Colleges | Full | Full | All Colleges | Full | Global read and controlled lifecycle |
| FINANCE_TEAM | Assigned College metadata only | No | No | No | No | No CourseEnrollment/Student detail |
| HOD | Assigned College + Department | Full in Department | Full in Department | Any permitted Section in Department | Full in Department | Department read/withdraw; verify completion |
| MANAGER | Assigned Section Courses | SECTION_ONLY own Section | No | Own Section only | Own Section only | Read assigned Section; no completion override |
| STUDENT | Own enrolled Courses | No | No | No | View permitted content | Own record only; no self-enrollment/withdrawal |

Manager Create/Update* is constrained to SECTION_ONLY Courses and the Manager's assigned Section. Student permission is View only: the platform View/Enroll conflict is reconciled to the more restrictive Page/API matrices. Finance metadata View does not include enrollment or academic progress. HOD is College + Department scoped; no role crosses Course.collegeId. Direct IDs, search, APIs, exports, and UI routes enforce identical scope/ownership checks.

## 16. Functional inventory

| ID | Function | Actor | Type | Adopted status |
|---|---|---|---|---|
| COURSE-F001 | List/view Courses | Authorized roles | Read | Adopted |
| COURSE-F002 | Create Course | SUPER_ADMIN/HOD; Manager SECTION_ONLY own Section | Create | Adopted |
| COURSE-F003 | Update Course metadata | SUPER_ADMIN/HOD; Manager own Section | Update | Adopted |
| COURSE-F004 | Archive/restore Course | SUPER_ADMIN/HOD | Lifecycle | Archive only; no hard delete |
| COURSE-F005 | Publish/change Course status | SUPER_ADMIN/HOD | Lifecycle | Adopted |
| COURSE-F006 | Set Course scope | SUPER_ADMIN/HOD; Manager SECTION_ONLY | Configuration | Adopted |
| COURSE-F007 | Assign Course to Section | SUPER_ADMIN/HOD; Manager own Section | Association | Adopted |
| COURSE-F008 | Remove Course-Section link | SUPER_ADMIN/HOD; Manager own Section | Association | Adopted |
| COURSE-F009 | View Section assignments | Permitted staff/Student | Read | Adopted |
| COURSE-F010 | Attach Problem to Course | SUPER_ADMIN/HOD; Manager own Section | Association | Adopted |
| COURSE-F011 | Remove Problem from Course | SUPER_ADMIN/HOD; Manager own Section | Association | Adopted |
| COURSE-F012 | View Course Problems | Permitted staff/Student | Read | Adopted |
| COURSE-F013 | Student My Courses | Student | Read | Enrolled-only |
| COURSE-F014 | Open Course details/content | Authorized staff/enrolled Student | Read/access | Adopted |
| COURSE-F015 | View Course enrollments | HOD/Manager/SUPER_ADMIN; Student own | Read | Adopted |
| COURSE-F016 | Create/assign enrollment | System automation | Create | Automatic only |
| COURSE-F017 | Automatic Section enrollment | System | Workflow | Option A adopted |
| COURSE-F018 | Student self-enrollment | Student | Workflow | Disabled in v1 |
| COURSE-F019 | Withdraw/re-enroll | Authorized staff/system | Lifecycle | Staff withdrawal; repeat disabled |
| COURSE-F020 | View enrollment/completion | Student/staff in scope | Read | Adopted |
| COURSE-F021 | Course completion | HOD verification | Lifecycle | HOD-verified; no auto formula |
| COURSE-F022 | Search/filter/sort | Authorized roles | Query | Adopted |
| COURSE-F023 | Subscription/seat eligibility | Module 3 | Authorization | Module 3 authoritative |
| COURSE-F024 | Read GroupCourseProblem | Group-authorized roles | Integration | Module 10 owns writes |
| COURSE-F025 | Preserve history | System/authorized staff | Integrity | Adopted |

## 17. Detailed function specifications

### COURSE-F001 — List and view Courses
**Evidence/status:** Page/entity mapping and RBAC are source-confirmed; list behavior is draft. **Actors:** SUPER_ADMIN, Finance Team (View per RBAC), HOD, Manager, Student.

- **Purpose:** return only Course records visible to the authenticated actor.
- **Scope:** SUPER_ADMIN global; Finance Team assigned College; HOD College+Department; Manager assigned Section/Group; Student own enrolled/assigned/permitted academic resources.
- **Preconditions/entry:** authenticated user; Dashboard/Courses & Labs/My Courses or approved admin route. Finance/SUPER_ADMIN route is TBD.
- **Inputs:** identity/role/scope from trusted auth; optional query/filter/sort/page parameters (exact fields TBD).
- **Main flow:** resolve identity → authorize Course view → apply server scope → apply approved Student visibility rule → read Course/association data → return permitted fields.
- **Rules:** client College/Department/Section IDs cannot widen scope; Finance View is not edit; Student access requires enrollment/assignment/permission; enum scope names are not interpreted by assumption.
- **Data:** read Course and only necessary CourseSection/CourseEnrollment/CourseProblem/GroupCourseProblem/Section.
- **Success:** safe list with permitted actions. **Failure:** empty state, unauthorized ID, query failure use TBD messages/behavior; no foreign data is returned.
- **Audit/privacy:** Module 15 governs read logging; no private enrollment data beyond scope.
- **Acceptance:** UI/API enforce the same scope; Student and Manager never receive foreign records.
- **QA:** COURSE-QA-001–010. **Decisions:** D04, D07–D08, D21, D24, D28.

### COURSE-F002 — Create Course
**Evidence/status:** Course schema, HOD academic ownership, HOD/SUPER_ADMIN Full CRUD source-confirmed; Manager Create* exists in RBAC but scope needs decision. **Actors:** HOD/SUPER_ADMIN; Manager conditional.

- **Purpose:** create Course linked to College, Department and creator.
- **Preconditions/entry:** valid role; Courses & Labs → Create; College/Department exist.
- **Inputs:** collegeId, departmentId, createdById (system), title varchar(255), description text, code varchar(50), scope enum, status enum. Requiredness/default/uniqueness TBD.
- **Flow:** show form → submit → check role/scope → validate College-Department hierarchy and fields/enums → derive creator from auth → persist Course/timestamps → return record.
- **Rules:** Department must belong to College (derived); client cannot set another creator; GLOBAL and initial status semantics TBD; creation does not auto-enroll or assign Section by default.
- **Data:** create Course only; no CourseSection/CourseEnrollment side effects assumed.
- **Success:** Course is saved and visible to authorized actors. **Failure:** invalid parent/enum/field rejects; duplicate code behavior TBD; Manager out-of-scope denied; repeat submit policy TBD.
- **Audit/privacy:** Module 15 Course create event.
- **Acceptance:** only approved roles create; trusted creator; hierarchy validated; no implicit enrollment.
- **QA:** COURSE-QA-011–020. **Decisions:** D01–D05, D07–D09, D22, D28–D30.

### COURSE-F003 — Update Course metadata
**Evidence/status:** updatedAt and RBAC update permission source-confirmed; field allowlist TBD. **Actors:** HOD/SUPER_ADMIN; Manager only within approved scope.

- **Entry/preconditions:** Course edit route; record exists; Update permission.
- **Inputs:** Course ID and changed title/description/code/scope fields. Status handled by F005. College/Department/creator are protected unless transfer is approved.
- **Flow:** load → scope check → show current data → submit → validate field allowlist and uniqueness → update approved fields + updatedAt → return updated record.
- **Rules:** scope/Department transfer after enrollment is not assumed; metadata edits must not silently alter CourseProblem, CourseEnrollment, submissions, Lab or Assessment history.
- **Data:** Course update only.
- **Success:** allowed edits persist. **Failure:** stale edit, invalid code/scope, unsupported field or unauthorized scope → reject/conflict with no partial mutation.
- **Audit/privacy:** actor and changed fields per Module 15.
- **Acceptance:** only approved fields change; scope and history remain consistent.
- **QA:** COURSE-QA-021–027. **Decisions:** D02–D06, D20, D22, D32.

### COURSE-F004 — Archive/delete/restore Course
**Evidence/status:** ARCHIVED enum and HOD/SUPER_ADMIN CRUD source-confirmed; delete/restore/cascade semantics absent.

- **Actors:** HOD/SUPER_ADMIN; Manager delete not explicitly granted.
- **Entry/input:** Course management action; Course ID, action, optional reason/confirmation TBD.
- **Flow:** authorize → inspect enrollment/Problem/Section/activity dependencies → apply approved archive/delete/restore rule → show outcome.
- **Rules:** ARCHIVED is not hard delete; restore is not source-confirmed; no cascade is inferred; history must not be destroyed by assumption.
- **Data:** update status or delete only under approved retention rule.
- **Failure:** linked activity, active enrollment, unauthorized action, already retired record → decision-gated.
- **Audit:** retirement/deletion actor/target per Module 15.
- **Acceptance:** no unapproved cascade; Manager cannot delete without explicit grant; history remains traceable.
- **QA:** COURSE-QA-028–033. **Decisions:** D22/D23/D33.

### COURSE-F005 — Publish/change Course status
**Evidence/status:** DRAFT/PUBLISHED/ARCHIVED enum source-confirmed; transitions/visibility TBD.

- **Actors:** HOD/SUPER_ADMIN; Manager permission TBD. **Input:** Course ID and target enum. **Entry:** status control.
- **Flow:** authorize → validate current/target state against Module 17 → apply approved publish checklist → update status/updatedAt → return result.
- **Rules:** enum names do not define access. DRAFT visibility, Student PUBLISHED access, ARCHIVED behavior, unpublish, and submission access are unresolved.
- **Data:** Course.status; audit event; notification only if Module 13 approves.
- **Failure:** invalid transition, incomplete Course, stale state, unauthorized actor → reject/no change.
- **Acceptance:** approved transition table governs all entry points and access.
- **QA:** COURSE-QA-034–039. **Decisions:** D22/D28/D33.


### COURSE-F006 — Set Course scope
**Evidence/status:** ContentScope values DEPT_WIDE/SECTION_ONLY/GLOBAL exist; semantics TBD. **Actor:** authorized Course admin. **Input:** Course ID, scope enum. **Entry:** Course create/edit. **Preconditions:** record and role scope valid. **Flow:** authorize → validate enum and actor's permitted audience → save scope/updatedAt → access checks consume approved scope policy. **Rules:** GLOBAL is not assumed cross-College; changes after enrollment/submissions require approved impact handling. **Data:** Course.scope. **Failure:** invalid enum or Manager requests wider scope → deny. **Success:** saved scope. **Acceptance:** tenant isolation preserved and one consistent interpretation used across API/UI. **QA:** COURSE-QA-040–046. **Decisions:** D07/D08/D31/D32.

### COURSE-F007 — Assign Course to Section
**Evidence/status:** CourseSection(courseId, sectionId, createdAt) source-confirmed. **Actor:** HOD/SUPER_ADMIN; Manager permission TBD. **Inputs:** courseId, sectionId. **Entry:** Course → Section assignment. **Preconditions:** both records exist and actor is in scope. **Flow:** select Section → backend checks hierarchy/role → create association → display updated assignment. **Rules:** multi-Section, duplicate, SECTION_ONLY requirement and auto-enrollment are TBD. CourseSection is not itself CourseEnrollment. **Data:** create CourseSection only. **Failure:** foreign Section, missing record, duplicate → reject or approved handling. **Acceptance:** valid College/Department relationship; no implicit enrollment. **QA:** COURSE-QA-047–053. **Decisions:** D09–D11/D29.

### COURSE-F008 — Remove Course-Section assignment
**Evidence/status:** relationship exists; removal effects TBD. **Actor:** HOD/SUPER_ADMIN; Manager TBD. **Input:** Course ID + Section ID. **Flow:** select link → authorize → inspect active enrollments/access → remove according to approved policy → refresh assignment list. **Rules:** no automatic withdrawal or deletion of CourseEnrollment, submissions, or progress. **Data:** unlink association; schema has no status/removedAt. **Failure:** missing link, dependency, foreign Section, stale update → decision-gated. **Acceptance:** historical enrollment remains intact. **QA:** COURSE-QA-054–057. **Decisions:** D10/D11/D15/D20/D32.

### COURSE-F009 — View Course Section assignments
**Actors:** HOD/Manager/SUPER_ADMIN; Student only within permitted context. **Entry:** Course detail. **Input:** Course ID. **Flow:** authorize Course → read CourseSection and Section → scope-filter → return approved Section fields. **Rules:** Manager sees assigned Section(s) only; student roster is not exposed by this view. **Data:** read only. **Failure:** unauthorized Course/Section → deny/filter. **Acceptance:** all results satisfy scope. **QA:** COURSE-QA-058–060. **Decisions:** D07–D10/D24.

### COURSE-F010 — Attach Problem to Course
**Evidence/status:** CourseProblem relation and Page Mapping v2 source-confirmed; Manager Create* in RBAC. **Actor:** HOD/SUPER_ADMIN, Manager if permitted. **Inputs:** courseId, problemId; createdAt system. **Entry:** Course → Problems → Add. **Flow:** authorize Course → verify Problem exists and is visible under Module 5 → validate scope → create CourseProblem → refresh list. **Rules:** no ordering, weighting, requiredness, schedule or due-date behavior inferred; no Problem copy created. **Failure:** invalid IDs, foreign Problem, duplicate, unavailable Problem → reject/decision-gated. **Data:** CourseProblem insert. **Acceptance:** no cross-scope linking. **QA:** COURSE-QA-061–067. **Decisions:** D19/D20/D34.

### COURSE-F011 — Remove Problem from Course
**Actor:** HOD/SUPER_ADMIN; Manager update permission conditional. **Input:** courseId/problemId. **Flow:** authorize → verify association → unlink → refresh. **Rules:** removing CourseProblem does not delete Problem or historical submissions; effect on existing progress remains TBD. **Data:** association removal only, if approved. **Failure:** missing link, active dependency, stale data or no permission → reject/decision required. **Acceptance:** Problem and historical records remain. **QA:** COURSE-QA-068–071. **Decisions:** D19/D20/D33.

### COURSE-F012 — View Course Problems
**Actors:** HOD/Manager/Student/SUPER_ADMIN as permitted. **Entry:** Course detail/My Courses. **Flow:** authorize Course → query CourseProblem → resolve Problem through Module 5 access rules → filter visibility → return metadata. **Rules:** Course membership does not override Problem status/scope; GroupCourseProblem is distinct. **Data:** read CourseProblem/Problem. **Failure:** inaccessible Problem omitted/denied according to approved policy. **Acceptance:** only allowed Problems appear. **QA:** COURSE-QA-072–076. **Decisions:** D19/D20/D21/D34.

### COURSE-F013 — Student My Courses
**Actor:** Student. **Entry:** My Courses/Dashboard. **Input:** authenticated Student ID; optional filters TBD. **Preconditions:** authenticated and active under Module 1 rules. **Flow:** derive studentId from auth → apply approved enrolled/Section/discovery policy → query CourseEnrollment/Course/CourseSection/Section → return Courses and allowed enrollment data. **Rules:** client cannot specify another studentId; enrolled-only vs assigned/discoverable Course model unresolved. **Data:** read only. **Failure:** no Courses, archived Course, subscription restriction → policy-dependent. **Acceptance:** only own and permitted academic records. **QA:** COURSE-QA-077–084. **Decisions:** D13/D21/D22/D25/D35.

### COURSE-F014 — Open Course details/content
**Actors:** Student/staff. **Input:** Course ID; direct URL and UI both supported subject to permission. **Flow:** resolve actor → authorize Course/scope → apply status, enrollment/Section/group and Module 3 entitlement rules → return metadata/content. **Rules:** Course view does not grant submission/Lab/Assessment permissions; Modules 5–9 own those. **Failure:** DRAFT/ARCHIVED, missing enrollment, changed Section, suspended subscription → TBD until policy approval. **Acceptance:** API/direct route enforce same access as UI. **QA:** COURSE-QA-085–091. **Decisions:** D07/D08/D21/D22/D25.

### COURSE-F015 — View Course enrollment list/details
**Actors:** HOD/Manager/SUPER_ADMIN; Student own enrollment. **Scope:** HOD College+Department; Manager assigned Section; Student own record. **Entry:** Students/Course enrollment view. **Input:** Course ID and Student ID only if allowed. **Flow:** authorize Course → query CourseEnrollment/User → apply scope/ownership → return permitted fields. **Rules:** Manager cannot see other Section students; Student cannot access another studentId; Finance Team enrollment access not mapped. **Failure:** foreign Course/Student → deny/no data leak. **Acceptance:** scope and ownership enforced. **QA:** COURSE-QA-092–097. **Decisions:** D01/D03/D13/D24/D36.

### COURSE-F016 — Create/assign enrollment
**Actor:** system automation only for ordinary v1 operation. No manual HOD/Manager enrollment endpoint and no Student self-enrollment endpoint.

- Triggers: published Course receives an audience Section; a Course is published; a Student joins/becomes active in an included Section; or Module 2 activates an included Section.
- Flow: durable event/outbox → re-read Course, Section, Student membership and College state → validate same-College scope, active Student membership and Module 3 entitlement → insert with server timestamp/source → emit Module 13/15 events.
- Student ID, enrollmentSource, enrolledAt, and initial status are server generated; client cannot set identity, creator, status, or timestamps.
- Unique Course + Student key makes retry idempotent. Existing ENROLLED/COMPLETED records are never reset. Module 4 must not increment/decrement usedSeats.
- Entitlement denial creates no row. Transient worker/storage failure retries missing pairs without reporting partial success.
- **QA:** COURSE-QA-098–106, 158–165, 171–173. **Decisions:** COURSE-D11–D16, D25, D29, D40.

### COURSE-F017 — Automatic Section enrollment (Option A adopted)
**Evidence/status:** adopted target behavior, not a source claim of existing implementation. **Actor:** system automation.

- Triggers: (1) PUBLISHED Course assigned to Section; (2) Course with audience links becomes PUBLISHED; (3) Student joins/becomes active in included Section; (4) Module 2 activates a Section inside DEPT_WIDE/GLOBAL.
- Worker reconciles eligible Student-Course pairs and re-reads Course/Section/Student/College state before insertion.
- Eligibility: Course PUBLISHED; scope includes active Section; active Student membership; same College; College ACTIVE; Module 3 entitlement positive.
- Enforce unique (courseId, studentId). Replayed events/duplicate assignments are safe. Existing ENROLLED/COMPLETED records are not reset.
- Later entrants automatically receive one CourseEnrollment. Section removal/transfer preserves enrollment/history; where Student leaves audience, retain history-only Course and stop new activity access.
- No CourseEnrollment action changes Module 3 seat counts. Use durable transaction/outbox/retry and report unresolved failures.
- **QA:** COURSE-QA-107–110, 158–165, 171–174. **Decisions:** COURSE-D11–D16, D25, D29, D40.

### COURSE-F018 — Student self-enrollment
**Status:** disabled in v1.

- Student Course permission is View only. The platform View/Enroll conflict is reconciled to Page/API matrices, which grant View only.
- My Courses is populated through Option A automatic Section enrollment, not a self-service control or public catalogue.
- Student enrollment requests, guessed IDs, forged studentId values, and cross-College attempts are denied without a CourseEnrollment write.
- **QA:** COURSE-QA-111–115, 169–170. **Decisions:** COURSE-D12/D35.

### COURSE-F019 — Withdraw/remove/re-enroll
**Actors:** HOD within assigned Department; SUPER_ADMIN platform-wide; Manager only for Students in the assigned Section where the action is exposed. Student cannot self-withdraw.

- Staff withdrawal transitions ENROLLED to WITHDRAWN, requires a reason, sets server withdrawnAt, preserves the row and all academic history, and emits audit/notification events.
- Repeated withdrawal is idempotent. COMPLETED is not reset; correction requires SUPER_ADMIN, reason and audit.
- Repeat-course attempts are disabled in v1. Unique (courseId, studentId) prevents a second row; roster sync never resets completion.
- Section changes/unlink do not delete or reset enrollment. Students leaving audience retain a history-only Course entry.
- **QA:** COURSE-QA-116–122, 178. **Decisions:** COURSE-D15/D16/D18/D32/D33/D40.

### COURSE-F020 — View enrollment/completion state
**Actors:** Student own; HOD/Manager scoped; SUPER_ADMIN. **Entry:** My Courses/Students. **Input:** Course/enrollment ID. **Flow:** authorize record → read enrolledAt/completedAt → display approved fields. **Rules:** timestamps do not define percentage progress or activity completion; Modules 6–9 own activity progress. **Failure:** no/foreign enrollment → deny/empty. **Acceptance:** no fabricated progress metrics. **QA:** COURSE-QA-123–127. **Decisions:** D16/D17/D18/D24/D37.

### COURSE-F021 — Determine Course completion and set completedAt
**Policy:** HOD-verified completion in v1; no automatic completion formula.

- HOD assigned to Course Department reviews the record and confirms completion with a reason. SUPER_ADMIN may correct with reason and audit. Manager/Student cannot mark completion.
- On confirmation of an ENROLLED row, set status COMPLETED and completedAt using server time; emit Module 13/15 events.
- Solving CourseProblems or completing Labs, Assessments, Contests, or CodeSubmissions does not automatically complete a Course. Modules 6–10 remain independent until a future cross-module policy exists.
- Duplicate confirmation is idempotent and retains original completedAt.
- **QA:** COURSE-QA-128–134, 175–177. **Decisions:** COURSE-D17/D18/D37/D38.

### COURSE-F022 — Search/filter/sort/paginate Courses
**Actors:** permitted staff/Student. **Inputs:** query, status/scope/Department/Section/enrollment filter, sort/page (exact list TBD). **Flow:** validate parameters → apply trusted scope first → filter/sort/page → return results/count. **Rules:** query cannot widen scope or reveal hidden Course metadata. **Failure:** invalid filter/page/sort → API policy TBD. **Acceptance:** search is tenant-safe. **QA:** COURSE-QA-135–140. **Decision:** D24/D28.

### COURSE-F023 — Subscription/seat eligibility
**Owner:** Module 3 is authoritative for subscription entitlement and seat capacity.

- Before automatic enrollment and active Student Course access, validate active Student membership, College ACTIVE state, and current Module 3 entitlement.
- Active Student membership remains the seat trigger. CourseEnrollment must not consume, reserve, release, or recalculate seats or update usedSeats/purchasedSeats.
- If entitlement is denied, no new enrollment or active access is granted. If unavailable, fail closed for new enrollment/access and retry reconciliation.
- Existing enrollment history remains; access follows Module 3 policy, not a Module 4-invented grace rule.
- **QA:** COURSE-QA-141–146, 171–173. **Decision:** COURSE-D25.

### COURSE-F024 — Read GroupCourseProblem
**Evidence/status:** relation source-confirmed; Module 10 owns group semantics. **Actors:** staff/Student in permitted group; SUPER_ADMIN. **Input:** Group/Course context. **Flow:** authorize Course → validate Group membership/scope via Module 10 → read GroupCourseProblem → return permitted data. **Rules:** not same as CourseProblem; membership does not imply CourseEnrollment. **Failure:** foreign Group/non-member/invalid link → deny/filter. **Acceptance:** no cross-group disclosure. **QA:** COURSE-QA-147–151. **Decision:** COURSE-D39.

### COURSE-F025 — Preserve Course/enrollment history
**Evidence/status:** derived integrity safeguard; cascade/retention unspecified. **Actors:** system/authorized staff. **Preconditions:** linked enrollment/activity exists. **Flow:** mutation request → inspect dependent records → apply approved retention/impact policy → preserve historical references → report outcome. **Rules:** no hard delete/cascade into enrollment/submissions/Labs/Assessments without approval. **Failure:** dependency conflict, partial update, stale link → reject/recover per policy. **Acceptance:** historical activity remains traceable. **QA:** COURSE-QA-152–157. **Decisions:** D20/D23/D33/D40.

## 18. Business rules and validation matrix

| Rule | Requirement | Evidence |
|---|---|---|
| COURSE-R01 | Course College and Department must be hierarchically consistent. | Derived |
| COURSE-R02 | createdById is derived from trusted actor. | Derived |
| COURSE-R03 | HOD Course operations are College+Department scoped. | RBAC |
| COURSE-R04 | Manager operations are College+Department+Section/Group scoped. | RBAC |
| COURSE-R05 | Student resources require enrollment, assignment, or permission. | RBAC |
| COURSE-R06 | CourseSection is not automatically CourseEnrollment. | Derived; D11 open |
| COURSE-R07 | CourseProblem and GroupCourseProblem are distinct. | Schema |
| COURSE-R08 | ContentScope labels do not define access. | Source gap |
| COURSE-R09 | PublishStatus labels do not define transitions. | Source gap |
| COURSE-R10 | Completion is not calculated from unapproved formula. | Source gap |
| COURSE-R11 | Seat entitlement is owned by Module 3. | Dependency |
| COURSE-R12 | Frontend hiding is not backend authorization. | RBAC |
| COURSE-R13 | Course changes must not silently destroy student history. | Derived |
| COURSE-R14 | Student enrollment/progress is ownership-scoped. | RBAC |
| COURSE-R15 | Manager action and Student self-enroll conflicts remain unresolved. | Source conflict |

| Validation ID | Condition | Expected behavior |
|---|---|---|
| COURSE-V01 | No Course view permission | Deny |
| COURSE-V02 | Course Department/College mismatch | Reject |
| COURSE-V03 | Missing Course ID | Not found, no data leak |
| COURSE-V04 | title exceeds 255 chars | Reject |
| COURSE-V05 | code exceeds 50 chars | Reject |
| COURSE-V06 | invalid ContentScope/PublishStatus | Reject |
| COURSE-V07 | Manager outside assigned Section | Deny |
| COURSE-V08 | Student requests another student's enrollment | Deny |
| COURSE-V09 | CourseSection references missing Course/Section | Reject |
| COURSE-V10 | CourseSection crosses organization scope | Reject unless approved exception |
| COURSE-V11 | duplicate CourseSection | Enforce approved uniqueness rule |
| COURSE-V12 | CourseEnrollment references missing Course/Student | Reject |
| COURSE-V13 | duplicate enrollment | Apply approved repeat policy |
| COURSE-V14 | Course unavailable under status/scope | Apply approved eligibility rule |
| COURSE-V15 | seat capacity exceeded | Module 3 decision/response |
| COURSE-V16 | CourseProblem references missing Course/Problem | Reject |
| COURSE-V17 | Problem outside permitted scope | Reject/filter |
| COURSE-V18 | invalid Course status transition | Reject |
| COURSE-V19 | completion criteria unmet | Do not set completedAt |
| COURSE-V20 | client supplies creator/student identity for another user | Reject/ignore; derive identity from auth |
| COURSE-V21 | scope/status change affects existing access | Approved impact policy only |
| COURSE-V22 | GroupCourseProblem without Group permission | Deny/filter |
| COURSE-V23 | search/filter widens scope | Apply trusted scope or reject |
| COURSE-V24 | delete/archive with linked history | Approved retention rule; no implicit cascade |

## 19. Data and state reference

### Course
| Field | Type | Business meaning / unresolved |
|---|---|---|
| id | varchar | ID generation TBD |
| collegeId | varchar | College scope |
| departmentId | varchar | Department scope; null/global behavior TBD |
| createdById | varchar | Creator; server-derived |
| title | varchar(255) | Requiredness/format TBD |
| description | text | Requiredness/format TBD |
| code | varchar(50) | uniqueness/case/format TBD |
| scope | ContentScope | DEPT_WIDE/SECTION_ONLY/GLOBAL; semantics TBD |
| status | PublishStatus | DRAFT/PUBLISHED/ARCHIVED; default/transitions TBD |
| createdAt/updatedAt | timestamp | System timestamps |

CourseSection fields: courseId, sectionId, createdAt. Composite uniqueness/unlink behavior TBD; no status/removedAt shown.

CourseEnrollment fields: id, courseId, studentId, enrolledAt, completedAt. No enrollment status, withdrawal timestamp, repeat identifier, or progress field is shown.

CourseProblem fields: courseId, problemId, createdAt. No order, requiredness, weight, due date, or availability field is shown.

GroupCourseProblem fields: groupId, courseId, problemId, createdAt. Module 10 owns write behavior.

### Draft state model (not approved)
| Entity | Source state/fields | Trigger | Next state | Status |
|---|---|---|---|---|
| Course | DRAFT/PUBLISHED/ARCHIVED | create/publish/archive/restore | Transition matrix TBD | enum only |
| CourseEnrollment | enrolledAt/completedAt | enroll/complete/withdraw/repeat | No status enum | Decision required |
| CourseSection | association present/absent | assign/remove | link create/delete | side effects TBD |
| CourseProblem | association present/absent | attach/remove | link create/delete | historical effects TBD |

Module 17 owns canonical state transitions.

## 20. Audit, notifications, privacy and integration
- Course notification triggers/recipients are not source-defined. Module 13 owns delivery. Candidate publish/Section assignment/enrollment/completion/archive events remain TBD.
- Module 15 owns Course create/update/publish/archive, CourseSection, CourseProblem, enrollment, completion, and override audit events.
- Student enrollment/completion are private academic records.
- Direct IDs, URLs, search, export, and APIs must enforce UI-equivalent scope/ownership.
- Course visibility does not grant CodeSubmission/LabSubmission/Assessment attempt access; Modules 5–9 own activity authorization.
- Do not log unnecessary Student private data.

## 21. Cross-module dependencies
| Module | Dependency |
|---|---|
| 1 | role/status, user identity, College/Department/Section assignment |
| 2 | hierarchy and Section lifecycle |
| 3 | seat counting, capacity, Subscription entitlement |
| 5 | Problem status/scope/metadata |
| 6 | code submission and execution |
| 7 | verdicts/results |
| 8 | Labs and possible completion contribution |
| 9 | Assessments and possible completion contribution |
| 10 | GroupCourseProblem/membership/scope |
| 12 | certificate completion relationship |
| 13 | notifications |
| 15 | audit events |
| 17 | state transitions |
| 18 | concurrency/partial failure/scope-change edge cases |

## 22. Decision log supplement

The complete adopted decision set is COURSE-D01–COURSE-D40 in ADOPTED-DECISIONS.md. Historical open/TBD wording retained in source-reconciliation passages is superseded by the normative baseline in Section 24 and does not represent pending product choices.

## 23. Review checklist

- [x] COURSE-F001–F025 have adopted behavior and traceable QA coverage.
- [x] Role and College/Department/Section boundaries are explicit.
- [x] Student self-enrollment conflict resolved to View only in v1.
- [x] DEPT_WIDE, SECTION_ONLY, and GLOBAL scopes are defined and College-isolated.
- [x] CourseSection is audience; CourseEnrollment is individual Student participation.
- [x] Option A includes later Section entrants and idempotent replay.
- [x] Publication, archive, restoration, and history retention are explicit.
- [x] CourseProblem and GroupCourseProblem remain distinct.
- [x] Enrollment lifecycle, repeat prohibition, and HOD-verified completion are explicit.
- [x] Module 3 seat trigger remains active Student membership only.
- [x] Module 13 notification and Module 15 audit integration are assigned.
- [x] QA suite has 180 unique target cases; execution remains pending.
- [ ] Schema migration and canonical RBAC changes implemented.
- [ ] Module 2 Section/roster events and audience reconciliation implemented.
- [ ] Enrollment outbox/retry/idempotency and Module 3 entitlement integrated.
- [ ] Modules 5–10, 13, 15, 17, and 18 contracts integrated.
- [ ] QA execution, security review, and product sign-off complete.

**Review gate:** Module 4 target product baseline adopted; implementation-ready only after engineering gates in Section 26 are closed.

**Review gate:** Module 4 only. GitHub commit means document versioned, not target behavior approved or implementation-ready.

## 24. Adopted target behavior — normative v1 baseline

This section is authoritative where earlier source-based passages retain TBD/conditional language.

### 24.1 Roles and authorization
SUPER_ADMIN has platform-wide Course administration. HOD has full administration within assigned College + Department, including publish/archive/restore, Section/Problem association, Department enrollment management and completion verification. Manager may create/update SECTION_ONLY Courses and manage only its assigned Section's audience/Problem links; it cannot widen scope, publish, archive, restore or mark completion. FINANCE_TEAM has assigned-College Course metadata View only, with no enrollment/progress data and no dedicated page requirement. STUDENT has View only, own enrolled Courses and own enrollment; no self-enrollment or self-withdrawal. Every API/search/export/direct ID enforces server-resolved scope; Course.collegeId is the hard tenant boundary.

### 24.2 Metadata
title required, trimmed, non-empty, max 255. code required, trimmed, uppercase-normalized, max 50, unique case-insensitively within College; archived codes stay reserved. description optional. Creator, College, Department and timestamps are server controlled. GLOBAL Courses retain the owning Department for audit but broaden audience only within their College. No duration, semester, term, prerequisites, credits, instructor or public catalog fields are assumed.

### 24.3 Course scope and audience
| Scope | Meaning | Audience |
|---|---|---|
| SECTION_ONLY | Explicit Section(s) only | CourseSection links required; at least one before publish |
| DEPT_WIDE | All active Sections in owning Department | Materialize current links and reconcile new/activated Sections |
| GLOBAL | All active Sections in owning College across Departments | Materialize current links and reconcile new/activated Sections |

GLOBAL is never public or cross-College. DRAFT may have audience links prepared but creates no Student enrollments until published. Module 2 Section creation/activation must reconcile DEPT_WIDE/GLOBAL audiences. Scope/Department/Section changes require HOD/SUPER_ADMIN, impact preview and audit. Existing enrollments are retained; Students outside the changed audience receive history-only Course access.

### 24.4 Option A — automatic CourseEnrollment
**Adopted model:** assigning a PUBLISHED Course to a Section automatically creates CourseEnrollment for every eligible active Student in that Section. If assigned while DRAFT, enrollment occurs when it becomes PUBLISHED. A Student who later joins/becomes active in an included Section is automatically enrolled in every applicable PUBLISHED Course.

Eligibility is rechecked immediately before insert: Course PUBLISHED; Section inside scope and active; active Student membership; same College; College ACTIVE; positive Module 3 entitlement. No ordinary manual staff enrollment or Student self-enrollment route exists in v1. Student identity, enrollmentSource and timestamps are server-generated. Unique (courseId, studentId) makes replays safe; existing ENROLLED/COMPLETED rows are not reset. WITHDRAWN is not silently reactivated. Use durable outbox/retry and revalidate authoritative data. Section unlink stops future enrollment but never deletes existing enrollment/history.

### 24.5 Enrollment lifecycle
v1 statuses: ENROLLED, COMPLETED, WITHDRAWN. One row per Student + Course; repeats disabled. Student cannot self-withdraw. HOD can withdraw within Department, SUPER_ADMIN globally, and Manager only within its assigned Section where enabled. Withdrawal requires reason, sets withdrawnAt, preserves history and is audited. COMPLETED is not reset; corrections require SUPER_ADMIN, reason and audit. Course archive, unlink, transfer or scope changes never cascade-delete enrollments.

### 24.6 Publication, archive, restore and access
| Status | Staff | Student | Rule |
|---|---|---|---|
| DRAFT | Authorized staff | Hidden | No Student enrollment |
| PUBLISHED | Authorized staff | Eligible enrolled Students | Option A reconciliation |
| ARCHIVED | Authorized staff | Existing enrollments as read-only history | No new enrollment/activity |

Transitions: create as DRAFT; DRAFT→PUBLISHED; PUBLISHED→DRAFT; DRAFT/PUBLISHED→ARCHIVED; ARCHIVED→DRAFT. Only HOD/SUPER_ADMIN. Publish reconciles enrollment; unpublish retains enrollment but removes active content access; archive retains history and blocks new enrollment/activity; restore returns to DRAFT and does not auto-publish. No hard delete or cascade.

### 24.7 CourseProblem and GroupCourseProblem
CourseProblem is unordered; no order, weight, required/optional, due date, schedule or completion contribution in v1. Attach requires same-College Problem and Module 5 scope/status permission. Unlink removes association only and preserves Problem/submissions. Module 10 owns GroupCourseProblem writes and Group permission; Module 4 reads only after Module 10 authorization. Group-only links are not CourseProblem and do not create CourseEnrollment.

### 24.8 Completion and progress
HOD manually verifies completion; SUPER_ADMIN corrects with reason/audit; Manager and Student cannot mark completion. Confirmation sets status COMPLETED and server completedAt. Duplicate confirmation is idempotent. No automatic all-Problems/Labs/Assessments/Contests formula. Display statuses and timestamps only; never fabricate percentage progress.

### 24.9 Lists, notifications and audit
Staff search title/code and filter status/scope/Department/Section. Student sees only own enrolled Courses. Default sort createdAt descending then stable ID; default page 20/max 100. Module 13 owns delivery; emit Course lifecycle, audience assignment/removal, automatic enrollment, withdrawal and completion events. Module 15 owns audit; capture actor/service, Course/Student/Section, tenant scope, event/idempotency key, timestamp and reason where relevant.

### 24.10 Module 3 seat contract
Module 3 solely owns entitlement and seat capacity. Active Student membership is the seat trigger. CourseEnrollment never consumes/reserves/releases/recalculates seats or updates usedSeats/purchasedSeats. Validate entitlement and College ACTIVE before enrollment/active Course access. Denial creates no enrollment/access; entitlement outage fails closed and retries. Keep existing history.

## 25. Target schema delta

The following are required proposed changes; they are not present in the reconstructed schema as fetched.

### 25.1 Course
Keep current fields. Keep departmentId required as owning Department even for GLOBAL; audience spans Departments only within collegeId. Add case-insensitive unique (collegeId, normalizedCode) or normalized uppercase code + unique (collegeId, code) index.

### 25.2 CourseSection
Keep courseId, sectionId, createdAt. Add unique (courseId, sectionId) and audience reconciliation indexes. Treat links as Course audience/availability, not individual Student enrollment. Removal stops future auto-enrollment and is audited.

### 25.3 CourseEnrollment
Current fields are id, courseId, studentId, enrolledAt, completedAt. Add:
| Proposed field/constraint | Purpose |
|---|---|
| status enum ENROLLED/COMPLETED/WITHDRAWN | Explicit lifecycle |
| withdrawnAt nullable timestamp | Withdrawal time |
| withdrawalReason nullable text | Required for WITHDRAWN |
| enrollmentSource enum/string | SECTION_ASSIGNMENT, COURSE_PUBLISHED, STUDENT_JOINED_SECTION, SECTION_ACTIVATED, RECONCILIATION |
| createdById nullable actor/service reference | Human/system traceability |
| unique (courseId, studentId) | One v1 record per Student + Course |
| indexes (studentId, status), (courseId, status) | My Courses and roster reads |

COMPLETED and completedAt update atomically. WITHDRAWN requires withdrawnAt and reason. ENROLLED requires completedAt null. No attemptNumber because repeats are disabled.

### 25.4 CourseProblem / GroupCourseProblem
Add unique (courseId, problemId) for CourseProblem. Module 10 owns GroupCourseProblem write/uniqueness contract. No order/weight/requiredness/scheduling fields in v1.

### 25.5 Cross-module event requirements
Module 2 provides Section and roster lifecycle events/reconciliation hook. Module 1 provides trusted active Student membership. Module 3 provides entitlement validation without seat increments. Enrollment automation requires transactional outbox or equivalent durable retry and Student-Course idempotency. Database uniqueness is required despite application checks.

## 26. Engineering implementation gates (not product decisions)
1. Migrate CourseEnrollment lifecycle fields/enums, unique constraints and indexes; backfill old rows to ENROLLED unless completedAt is reliably set, then COMPLETED.
2. Update canonical RBAC/page/API matrices: Student View only; Manager SECTION_ONLY own-Section Create/Update; Finance metadata-only View; HOD/SUPER_ADMIN scope.
3. Implement ContentScope resolver and audience materialization; integrate Module 2 Section/roster events.
4. Implement Option A worker, outbox, idempotent retry/reconciliation and Module 3 entitlement.
5. Implement Course lifecycle/history-only visibility, CourseProblem rules, and Modules 5–10 permissions.
6. Integrate Module 13 notifications, Module 15 audit, Module 17 transitions and Module 18 concurrency/failure handling.
7. Execute 180 QA cases, migration validation and security review before release.

**Legacy boundary:** reviewed legacy Module 04 documents describe CourseRoom/ordered Rooms, Syllabus/overlays, Learning Paths/PathProgress and no CourseEnrollment. Those are historical only; they are not imported into the target module.
