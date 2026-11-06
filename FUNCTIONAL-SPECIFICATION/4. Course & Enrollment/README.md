# 4. Course & Enrollment

**Document type:** Functional Specification  
**Version:** 0.1 — Initial source-based draft  
**Status:** Awaiting user review and approval  
**Scope:** Course creation and management, course scope, section assignment, student enrollment, course visibility, course-problem association, and course completion data

> **Specification principle:** This module uses DEZOLVER Business Flow v2, Page-to-DB Mapping v2, and the schema reconstruction stored in the repository. It separates source-confirmed facts from derived safeguards and unresolved decisions. No missing behavior is silently assumed.

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

### 3.1 Business Flow v2

The business flow establishes this academic context:

- HOD manages Department, Sections, Managers, Students, Courses, Problems, and Groups.
- Manager operates within their assigned Section and manages students and activities there.
- Student accesses Courses and Problems, writes code, and submits it.
- Course and Problem are connected to the code submission workflow.

This confirms Courses as part of the HOD-managed department academic structure and the Student learning journey. It does **not** define detailed course creation, enrollment, or completion workflows.

### 3.2 Page-to-DB Mapping v2

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

The mapping confirms page/entity associations, not final UI behavior or all action permissions.

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

The Business Flow identifies Courses within HOD's Department academic management.

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

## 10. Decisions required before finalization

| ID | Decision | Why it matters |
|---|---|---|
| COURSE-D01 | Define who can create Courses (HOD, SUPER_ADMIN, Manager, or other). | Ownership/permissions |
| COURSE-D02 | Define HOD Course CRUD/publish/archive permissions. | RBAC |
| COURSE-D03 | Define Manager Course permissions: view, create, edit, publish, assign, enroll, or activity-only. | Section scope |
| COURSE-D04 | Define whether FINANCE_TEAM and SUPER_ADMIN can view/manage academic Courses. | Platform access |
| COURSE-D05 | Define Course required fields and code format. | Validation |
| COURSE-D06 | Define Course code uniqueness scope and case handling. | Duplicate prevention |
| COURSE-D07 | Define exact meanings of DEPT_WIDE, SECTION_ONLY, and GLOBAL. | Course visibility |
| COURSE-D08 | Define whether GLOBAL Course can cross College/Department boundaries. | Tenant isolation |
| COURSE-D09 | Define whether Course must be linked to one or more Sections. | CourseSection behavior |
| COURSE-D10 | Define who may assign/remove Sections from Course. | Academic administration |
| COURSE-D11 | Define whether CourseSection assignment automatically enrolls all Section Students. | Enrollment mechanics |
| COURSE-D12 | Define enrollment triggers (manual, automatic, self-enrollment, etc.). | Student access |
| COURSE-D13 | Define Student enrollment eligibility and cross-section/department rules. | Authorization |
| COURSE-D14 | Define duplicate enrollment behavior and uniqueness constraints. | Data consistency |
| COURSE-D15 | Define withdrawal, removal, reactivation, and repeat-course behavior. | Enrollment lifecycle |
| COURSE-D16 | Define whether CourseEnrollment needs status beyond enrolledAt/completedAt. | State modeling |
| COURSE-D17 | Define Course completion criteria and when completedAt is set. | Completion tracking |
| COURSE-D18 | Define completion reset/reversal and whether repeats create new enrollment. | History |
| COURSE-D19 | Define CourseProblem ordering, required/optional, weighting, and scheduling. | Course structure |
| COURSE-D20 | Define effect of adding/removing Problems after enrollment. | Progress/history |
| COURSE-D21 | Define My Courses visibility: enrolled-only vs Section-available/discoverable. | Student UX |
| COURSE-D22 | Define DRAFT/PUBLISHED/ARCHIVED visibility and action rules. | Publication |
| COURSE-D23 | Define Course archive/delete behavior and historical retention. | Data integrity |
| COURSE-D24 | Define Course search/filter/sort/pagination and role-specific columns. | UI |
| COURSE-D25 | Define subscription/seat access rules during enrollment and Course access. | Module 3 integration |
| COURSE-D26 | Define notifications for Course publish, assignment, enrollment, completion. | Module 13 |
| COURSE-D27 | Define Course-related audit events and metadata. | Module 15 |

---

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
- **TBD / Decision Required:** missing or conflicting behavior.
- **Recommendation:** suggestion only; not approved.
- No legacy implementation was inspected; no legacy-verified behavior is claimed.

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

This preserves source permissions; it does not settle conflicting entries.

| Role | Course view | Create | Update | Delete/archive | CourseProblem | Enrollment | Scope |
|---|---|---|---|---|---|---|---|
| SUPER_ADMIN | Allow | Full CRUD | Full CRUD | Full CRUD | Full CRUD | Global view; action policy TBD | Global |
| FINANCE_TEAM | View per platform matrix | Not granted | Not granted | Not granted | View per platform matrix | Not mapped | Assigned College |
| HOD | Full CRUD | Full CRUD | Full CRUD | Full CRUD | Full CRUD | CourseEnrollment mapped; action TBD | College + Department |
| MANAGER | V/C/U* in platform/page/API matrices | Create* | Update* | Delete not explicitly granted | Create/Update* in platform matrix; detail TBD | Section student activity; enrollment action TBD | College + Department + assigned Section/permitted Group |
| STUDENT | View/Enroll in platform matrix; View in page/API matrix | No | No | No | View | Self-enrollment conflict | Own + permitted academic resources |

- * means additional organization scope must be checked.
- CourseSection and CourseEnrollment lack separate detailed permission matrices in the sources.
- Student “Enroll” conflict must be resolved before self-enrollment is implemented.
- Page mapping is not authorization.
- Backend permission, scope, ownership, and relationship checks are mandatory.

## 16. Functional inventory

| ID | Function | Actor | Entry point | Type | Status |
|---|---|---|---|---|---|
| COURSE-F001 | List/view Courses | Staff/Student | Dashboard, Courses & Labs, My Courses | Read | Draft |
| COURSE-F002 | Create Course | HOD/SUPER_ADMIN; Manager conditional | Courses & Labs | Create | Decision required |
| COURSE-F003 | Update Course metadata | HOD/SUPER_ADMIN; Manager conditional | Course edit | Update | Decision required |
| COURSE-F004 | Archive/delete/restore Course | HOD/SUPER_ADMIN | Course management | Lifecycle | Decision required |
| COURSE-F005 | Publish/change Course status | HOD/SUPER_ADMIN; Manager conditional | Course status action | Lifecycle | Decision required |
| COURSE-F006 | Set Course scope | Authorized Course admin | Course form/details | Configuration | Decision required |
| COURSE-F007 | Assign Course to Section | HOD/SUPER_ADMIN; Manager TBD | Section assignments | Association | Decision required |
| COURSE-F008 | Remove Course-Section link | HOD/SUPER_ADMIN; Manager TBD | Section assignments | Association | Decision required |
| COURSE-F009 | View Section assignments | Staff/Student permitted | Course details | Read | Draft |
| COURSE-F010 | Attach Problem to Course | HOD/SUPER_ADMIN/Manager per RBAC | Course Problems | Association | Decision required |
| COURSE-F011 | Remove Problem from Course | HOD/SUPER_ADMIN/Manager per RBAC | Course Problems | Association | Decision required |
| COURSE-F012 | View Course Problems | Staff/Student permitted | Course/My Courses | Read | Draft |
| COURSE-F013 | View My Courses | Student | My Courses/Dashboard | Read | Decision required |
| COURSE-F014 | Open Course details/content | Staff/Student | Course list | Read/access | Decision required |
| COURSE-F015 | View Course enrollments | HOD/Manager/Student own | Students/Course | Read | Decision required |
| COURSE-F016 | Create/assign enrollment | HOD/SUPER_ADMIN; Manager TBD | Course/Students | Create | Decision required |
| COURSE-F017 | Automatic Section enrollment | System | CourseSection event | System workflow | Not source-confirmed |
| COURSE-F018 | Student self-enrollment | Student | My Courses | Workflow | RBAC conflict |
| COURSE-F019 | Withdraw/remove/repeat enrollment | Student/staff/system | Enrollment action | Lifecycle | Decision required |
| COURSE-F020 | View enrollment/completion state | Student/staff | My Courses/Students | Read | Draft |
| COURSE-F021 | Determine Course completion | System/staff | Completion trigger | System workflow | Decision required |
| COURSE-F022 | Search/filter/sort Courses | Staff/Student | Course list | Query/read | Decision required |
| COURSE-F023 | Subscription/seat eligibility | System | Enrollment/course access | Authorization | Module 3 dependency |
| COURSE-F024 | Read GroupCourseProblem in Course context | Staff/Student permitted | Course/Group page | Integration/read | Module 10 owner |
| COURSE-F025 | Preserve history during Course changes | System | Course/enrollment/content changes | Data integrity | Decision required |

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
**Evidence/status:** CourseEnrollment schema confirmed; enrollment trigger/permission TBD. **Actors:** HOD/SUPER_ADMIN; Manager permission TBD. **Inputs:** courseId/studentId; enrolledAt system; completedAt null initially unless approved. **Preconditions:** Course/Student valid, actor scope, eligibility and Module 3 seat policy approved. **Flow:** select Student/Course → verify permission/hierarchy/status → check Module 3 seat rule → apply duplicate policy → create CourseEnrollment → return record. **Rules:** Section assignment does not auto-enroll absent decision D11. **Failure:** wrong scope, full seat capacity, duplicate, unavailable Course → reject per policy. **Acceptance:** only approved trigger creates enrollment and seat check is authoritative. **QA:** COURSE-QA-098–106. **Decisions:** D11–D16/D25/D29/D30/D35.

### COURSE-F017 — Automatic Section enrollment
**Evidence/status:** Not source-confirmed; only related entities exist. **Actor:** system if approved. **Trigger:** CourseSection change, if selected by product. **Candidate flow:** event → resolve eligible Section students → apply course eligibility/seat checks → create enrollments idempotently. **Important:** candidate only; do not implement before D11. **Failure:** partial enrollment, duplicate event, roster changes, seat shortage → policy TBD. **Acceptance:** blocked pending decision. **QA:** COURSE-QA-107–110 (blocked). **Decisions:** D11/D13/D14/D25/D29.

### COURSE-F018 — Student self-enrollment
**Evidence/status:** RBAC platform matrix says View/Enroll; page/API matrix says View; Business Flow silent. **Actor:** Student if approved. **Entry:** My Courses/discovery action TBD. **Input:** Course ID; studentId derived from auth. **Candidate flow:** submit enrollment request → check permission, Course scope/status, seat eligibility → create enrollment → return result. **Rules:** no self-enrollment until RBAC conflict resolved; no arbitrary studentId. **Failure:** disabled enrollment, foreign Course, seat full, already enrolled → decision-based. **Acceptance:** blocked until D12/D13/D14/D25/D35. **QA:** COURSE-QA-111–115 (blocked).

### COURSE-F019 — Withdraw/remove/re-enroll
**Evidence/status:** CourseEnrollment lacks status/withdrawnAt/repeat field. **Actors:** Student/staff/system according to future decision. **Input:** enrollmentId and action. **Flow:** authorize/ownership check → inspect history → apply approved lifecycle model → return result. **Rules:** hard-delete/soft-delete/status/repeat behavior not inferred; preserve submissions/progress. **Data:** update/delete/recreate only under approved model. **Failure:** completed course, prior submissions, seat changes, duplicate request, concurrent update → TBD. **Acceptance:** no historical work lost. **QA:** COURSE-QA-116–122. **Decisions:** D15/D16/D18/D25/D29/D30.

### COURSE-F020 — View enrollment/completion state
**Actors:** Student own; HOD/Manager scoped; SUPER_ADMIN. **Entry:** My Courses/Students. **Input:** Course/enrollment ID. **Flow:** authorize record → read enrolledAt/completedAt → display approved fields. **Rules:** timestamps do not define percentage progress or activity completion; Modules 6–9 own activity progress. **Failure:** no/foreign enrollment → deny/empty. **Acceptance:** no fabricated progress metrics. **QA:** COURSE-QA-123–127. **Decisions:** D16/D17/D18/D24/D37.

### COURSE-F021 — Determine completion and set completedAt
**Evidence/status:** completedAt field exists; criterion/trigger absent. **Actor:** system; manual override only if approved. **Preconditions:** completion formula and state machine approved. **Flow:** approved event → evaluate criteria → set completedAt → notify/certificate/report only if explicitly approved. **Rules:** do not assume all CourseProblems/Labs/Assessments/Contests count. **Failure:** criteria unmet, duplicate trigger, missing activity result, unauthorized override → no completion. **Acceptance:** only approved criterion sets timestamp; idempotent. **QA:** COURSE-QA-128–134 (decision blocked). **Decisions:** D17/D18/D37/D38.

### COURSE-F022 — Search/filter/sort/paginate Courses
**Actors:** permitted staff/Student. **Inputs:** query, status/scope/Department/Section/enrollment filter, sort/page (exact list TBD). **Flow:** validate parameters → apply trusted scope first → filter/sort/page → return results/count. **Rules:** query cannot widen scope or reveal hidden Course metadata. **Failure:** invalid filter/page/sort → API policy TBD. **Acceptance:** search is tenant-safe. **QA:** COURSE-QA-135–140. **Decision:** D24/D28.

### COURSE-F023 — Subscription/seat eligibility
**Evidence/status:** Module 3 dependency; course policy absent. **Actor:** system. **Inputs:** College/Department/Student/subscription/seat context. **Preconditions:** Module 3 canonical rule approved. **Flow:** enrollment/access attempt → consult Module 3 entitlement → allow/deny according to authoritative response. **Rules:** do not calculate seat availability independently from overlapping College/Department/Subscription fields; subscription status meanings are not inferred. **Failure:** seat drift, entitlement outage, concurrent subscription change → Module 3/18 policy. **Acceptance:** single authoritative entitlement model. **QA:** COURSE-QA-141–146. **Decision:** COURSE-D25.

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
COURSE-D01–D27 in the existing Section 10 remain open. Additional decisions:

| ID | Question | Impact | Owner | Status |
|---|---|---|---|---|
| COURSE-D28 | Define Course status transition/access table. | Publication | Product/Security | Open |
| COURSE-D29 | Decide Section assignment auto-enrollment/removal and idempotency. | Enrollment | Product/Architecture | Open |
| COURSE-D30 | Define role/method enrollment permission matrix. | RBAC | Product/Security | Open |
| COURSE-D31 | Define DEPT_WIDE/SECTION_ONLY/GLOBAL including College boundaries. | Tenant isolation | Product/Security | Open |
| COURSE-D32 | Define scope/Department/Section change effects after enrollment/activity. | Access/history | Product/Architecture | Open |
| COURSE-D33 | Define archive/delete/restore/retention/cascade. | Data integrity | Product/Architecture | Open |
| COURSE-D34 | Define Problem association eligibility/order/requiredness/scheduling/history effects. | Course content | Product/Academic | Open |
| COURSE-D35 | Resolve Student View/Enroll vs View-only RBAC conflict. | Student enrollment | Product/Security | Open |
| COURSE-D36 | Define enrollment fields/export access by HOD/Manager/Student. | Privacy | Product/Security | Open |
| COURSE-D37 | Define progress representation; schema only has enrolledAt/completedAt. | UX/schema | Product/Architecture | Open |
| COURSE-D38 | Define completion criteria, automatic/manual, reversal/repeat/certificate effect. | Completion | Product/Academic | Open |
| COURSE-D39 | Define GroupCourseProblem ownership and relation to CourseProblem. | Group integration | Product/Architecture | Open |
| COURSE-D40 | Define concurrent Course/Section/Problem edits during enrollment/activity. | Integrity | Architecture/QA | Open |

## 23. Review checklist
- [ ] COURSE-F001–F025 reviewed and source-traced.
- [ ] HOD/SUPER_ADMIN/Manager permissions approved.
- [ ] Finance Team Course view route resolved/deferred.
- [ ] Student self-enrollment conflict resolved.
- [ ] Scope meanings and cross-College behavior approved.
- [ ] CourseSection vs CourseEnrollment relationship explicit.
- [ ] Status transitions/visibility approved.
- [ ] CourseProblem add/remove and historical effects defined.
- [ ] Enrollment, withdrawal, repeat and seat triggers defined.
- [ ] Completion formula and completedAt behavior approved.
- [ ] Module 3 entitlement and Module 10 group integration aligned.
- [ ] QA includes expected results; decision-blocked tests marked.
- [ ] No source conflict/recommendation silently becomes approved behavior.

**Review gate:** Module 4 only. GitHub commit means document versioned, not target behavior approved or implementation-ready.
