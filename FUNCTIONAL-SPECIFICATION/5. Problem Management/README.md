# 5. Problem Management

**Document type:** Functional Specification  
**Version:** 0.2 — Detailed draft aligned to Functional Specification Standard v1.0  
**Status:** Detailed draft — ready for product review; target behavior and decisions remain unapproved  
**Scope:** Problem authoring, metadata, access scope, publication, and TestCase management. Submission execution and evaluation belong to Modules 6–7.

> This draft is based on Business Flow v2, Page-to-DB Mapping v2, and the schema reconstruction in the repository. Source-confirmed facts, derived safeguards, and unresolved decisions are distinguished. Missing behavior is not silently assumed.

## 1. Purpose

Define how DEZOLVER creates, maintains, organizes, and makes coding Problems available to authorized users.

Problems are reusable content used in:
- Courses and Course Problems
- Labs
- Assessments
- Contests
- Student Practice Problems

## 2. Scope

### Included
- Problem authoring and ownership
- Problem statement and metadata
- Difficulty, tags, and academic batch
- College/Department scope, public visibility, and ContentScope
- Allowed programming languages and resource limits
- Problem publication/lifecycle questions
- TestCase authoring, ordering, and hidden/visible classification
- Problem access from Courses, Labs, Assessments, Contests, and Practice Problems
- Authorization, validation, source gaps, and decisions required

### Owned by other modules

| Topic | Specification owner |
|---|---|
| Course and CourseProblem association | Module 4 — Course & Enrollment |
| Code editor, submissions, Judge0 | Module 6 — Code Submission & Judge0 |
| Test execution, expected-output comparison, verdicts | Module 7 — Test Case Evaluation |
| Lab assignment/scoring | Module 8 — Labs & Lab Scoring |
| Assessment problem assignment | Module 9 — Assessments |
| Group-specific Problem association | Module 10 — Groups |
| Contest-specific Problem assignment | Module 11 — Contests |
| Notifications | Module 13 — Notifications |
| Audit events | Module 15 — Audit Logging |
| State transitions | Module 17 — Functional State Machines |
| Edge cases | Module 18 — Edge Cases & Failure Handling |

## 3. Source documents and interpretation

### 3.1 Business Flow v2

The source flow establishes:
- HOD manages Problems within the Department academic structure.
- Manager manages Students and activities within the assigned Section.
- Students access Problems from Courses, write code, and submit.
- Problems are used in Course, Lab, and Contest workflows.
- Submission and evaluation occur downstream of Problem access.

It does not describe Problem authoring screens, publication rules, or detailed permissions.

### 3.2 Page-to-DB Mapping v2

| Role | Page | Problem-related entities |
|---|---|---|
| HOD | Courses & Labs | Course, CourseProblem, GroupCourseProblem, LabAssignment, LabSubmission, CodeSubmission, Problem |
| HOD | Problems | Problem, TestCase, CodeSubmission, User, Department, College |
| Manager | Courses & Labs | Course, CourseProblem, GroupCourseProblem, LabAssignment, LabSubmission, CodeSubmission, Problem |
| Manager | Problems | Problem, TestCase, CodeSubmission, User, Department, College |
| Student | My Courses | Course, CourseProblem, GroupCourseProblem, Problem |
| Student | Assignments / My Labs | LabAssignment, LabSubmission, CodeSubmission, Problem, Course |
| Student | Practice Problems | Problem, TestCase, CodeSubmission, User |
| Student | Contests | Contest, ContestProblem, ContestParticipant, ContestScoreRank, Problem, Group, GroupMember, GroupContest, CodeSubmission |

The mapping confirms entity/page associations, not complete UI behavior or action permissions.

### 3.3 Schema reconstruction

The schema defines Problem and TestCase, and links Problem to CodeSubmission. Problem is referenced by CourseProblem, LabAssignment, AssessmentProblem, ContestProblem, and GroupCourseProblem-related relationships.

## 4. Roles and access scope

| Role | Source-described responsibility | Scope |
|---|---|---|
| SUPER_ADMIN | Platform administration exists; Problem authoring rights are not explicit in the Problem page mapping. | Platform role; Problem permissions TBD |
| FINANCE_TEAM | No Problem-management responsibility shown. | Problem access not specified |
| HOD | Department academic management includes Problems; HOD Problems page is mapped. | Assigned College and Department |
| MANAGER | Problems page is mapped; Manager operates within assigned Section. | Assigned College, Department, Section |
| STUDENT | Accesses Problems through My Courses, Practice Problems, Labs, Assessments, and Contests. | Authorized Problems only |

### 4.1 Authorization safeguards

These are derived safeguards:
- Problem operations must validate actor role and organization scope server-side.
- HOD operations must be restricted to authorized College and Department.
- Manager access must be limited to assigned Section and authorized activities.
- Students must access only Problems exposed by approved Course, Lab, Assessment, Contest, or practice rules.
- Student requests must not retrieve private Problem content or hidden TestCase data by manipulating identifiers.
- Client-supplied collegeId, departmentId, createdById, or visibility values must not override server-side authorization.

**TBD:** action-level rights for create, edit, publish, archive, delete, clone, TestCase management, and cross-department sharing.

## 5. Functional requirements

### PROB-001 — Create and maintain Problem records

**Source-confirmed schema.**

Problem fields:
- id
- collegeId
- departmentId
- createdById
- academicBatch
- title
- statement
- difficulty
- tags
- timeLimitMs
- memoryLimitMb
- allowedLanguages
- isPublic
- scope
- createdAt
- updatedAt

Problem is associated with a College, Department, and creator in the schema.

**TBD:** whether College/Department association can be absent for global Problems, and whether SUPER_ADMIN may create tenant-independent Problems.

### PROB-002 — Problem title and statement

**Source-confirmed schema:** title (varchar(255)) and statement (text).

The Problem statement is the task content shown to users who have access.

**TBD:**
- Required-field rules and title uniqueness.
- Markdown, rich-text, or HTML support.
- Sanitization and allowed markup.
- Structure for constraints, input/output format, and sample input/output.
- Editorial, hints, explanation, and solution support.
- Whether the statement has structured fields or is one text field.

No editorial/hint fields are shown in the schema reconstruction.

### PROB-003 — Difficulty

**Source-confirmed enum:** ProblemDifficulty = EASY, MEDIUM, HARD.

Problem has a difficulty field.

**TBD:** whether difficulty is mandatory, editable after publication, shown on every user-facing page, or used for scoring/recommendation.

### PROB-004 — Tags

**Source-confirmed schema:** tags is text[].

**TBD:**
- Free-form tags vs controlled catalogue.
- Normalization, casing, and duplicate handling.
- Whether tags are College-specific or platform-wide.
- Student tag filtering and tag edits after publication.

### PROB-005 — College, Department, and academic batch

**Source-confirmed schema:** collegeId, departmentId, academicBatch.

**Derived consistency safeguard:** for tenant-specific Problems, the Department must belong to the specified College and the actor must have permission within that scope.

**TBD:**
- Whether academicBatch is optional and its format.
- Whether a Problem targets one or multiple batches.
- Whether academicBatch affects visibility or only filtering.
- Cross-Department or cross-College reuse rules.

### PROB-006 — Problem scope

**Source-confirmed enum:** ContentScope = DEPT_WIDE, SECTION_ONLY, GLOBAL.

Problem has a scope field.

**TBD:** exact semantics:
- Does DEPT_WIDE mean visible to all eligible users in the Department?
- Does SECTION_ONLY require a Section association or rely on creator/assignment context?
- Does GLOBAL mean all Colleges, all Departments, or global within one College?
- How does GLOBAL interact with collegeId and departmentId?
- Can scope change after Problem has been linked to activities?

Enum names are confirmed; complete access behavior is not.

### PROB-007 — Public/private visibility

**Source-confirmed schema:** isPublic (boolean).

**TBD:**
- Does true mean all platform users, College-wide visibility, or anonymous internet access?
- Is isPublic subordinate to or overriding ContentScope?
- Do public Problems automatically appear in Practice Problems?
- Can public Problems be used in private Courses or Contests?
- Can other users edit public Problems?

**Security note:** do not interpret isPublic as anonymous internet access unless explicitly approved.

### PROB-008 — Allowed programming languages

**Source-confirmed schema:** allowedLanguages (text[]).

This field represents languages allowed for the Problem.

**TBD:**
- Canonical language identifiers and display labels.
- Whether the list must contain at least one language.
- Whether empty/null means all supported languages, none, or unspecified.
- Whether Course/Lab/Assessment/Contest can override Problem languages.
- Whether a language can be removed after submissions exist.
- Mapping to Judge0 language IDs (Module 6).

Do not assume an empty language list means unrestricted access.

### PROB-009 — Time and memory limits

**Source-confirmed schema:** timeLimitMs (int) and memoryLimitMb (int).

These are Problem-level resource-limit fields.

**TBD:**
- Valid ranges and defaults.
- Whether Judge0 consumes the values directly or requires conversion.
- Whether limits vary by language.
- Whether academic activities can override limits.
- Whether edits apply to existing submissions or only future submissions.

Limit enforcement belongs to Modules 6–7.

### PROB-010 — Publication and lifecycle

**Source gap:** no Problem status field is shown in the schema reconstruction. Other entities use PublishStatus, but Problem does not show it.

**Decision required:**
- Does Problem have DRAFT/PUBLISHED/ARCHIVED status?
- If the implementation has a status field absent from the source schema, what is canonical?
- Are visibility rules based only on isPublic, scope, and activity associations?
- Who may publish/archive?
- Can unpublished Problems be attached to Courses, Labs, Assessments, or Contests?
- What happens to linked activities when a Problem is archived?

Do not assume Course PublishStatus applies to Problem.

### PROB-011 — TestCase management

**Source-confirmed schema.**

TestCase fields:
- id
- problemId
- input
- expectedOutput
- isHidden
- orderIndex
- createdAt

Each TestCase belongs to a Problem through problemId.

**Derived requirement:** TestCase must reference a valid Problem.

**TBD:**
- Required/optional input and expectedOutput.
- Empty input/output behavior.
- TestCase count and input/output size limits.
- Add/edit/delete behavior after publication.
- Bulk import support.
- Whether samples and hidden TestCases share one authoring interface.
- Whether orderIndex must be unique within a Problem.

### PROB-012 — TestCase ordering

**Source-confirmed schema:** orderIndex.

**Derived requirement:** TestCases should be returned in deterministic order.

**TBD:**
- Whether orderIndex begins at zero or one.
- Whether duplicate order values are allowed.
- How ordering is changed.
- Whether ordering impacts failedTestCaseIndex from submissions.
- Whether order remains stable after publication.

### PROB-013 — Hidden vs visible TestCases

**Source-confirmed schema:** isHidden (boolean).

**Derived security safeguard:** Student-facing APIs must not expose hidden TestCase input or expectedOutput through page payloads, direct API calls, or error responses.

**TBD:**
- Whether visible cases are samples.
- Whether visible and hidden cases are both evaluated.
- Whether Students see hidden TestCase count or aggregate information.
- Whether staff roles can inspect hidden data.
- Whether partial hiding is supported or only a boolean.

Evaluation semantics belong to Module 7.

### PROB-014 — Reuse across academic activities

**Source-confirmed relationships/page mapping:** Problem is referenced by CourseProblem, LabAssignment, AssessmentProblem, ContestProblem, and GroupCourseProblem-related association.

**Derived requirement:** Problem identity must remain stable so activity records can reference it.

**TBD:**
- Reuse across multiple Courses/Labs/Assessments/Contests.
- Whether edits affect all linked activities immediately.
- Whether activities snapshot Problem statement or TestCases.
- Whether a referenced Problem can be deleted.
- Whether activity-level visibility is independent from Problem visibility.

### PROB-015 — HOD Problem Management page

**Source-confirmed page mapping:** HOD Problems page maps Problem, TestCase, CodeSubmission, User, Department, and College.

**TBD:** HOD actions (create/edit/publish/archive, TestCase management, search/filter, submission history visibility, and whether HOD can manage Problems created by other users within the Department).

### PROB-016 — Manager Problem Management page

**Source-confirmed page mapping:** Manager Problems page maps Problem, TestCase, CodeSubmission, User, Department, and College.

Business Flow v2 states Manager manages only Students and activities within assigned Section.

**Derived safeguard:** Manager Problem access must be restricted to authorized Section/activity scope.

**TBD:** whether Manager can create/edit Problems, manage TestCases, publish/archive, or only access Problems assigned to activities.

### PROB-017 — Student Practice Problems

**Source-confirmed page mapping:** Student Practice Problems maps Problem, TestCase, CodeSubmission, and User.

**Derived safeguard:** only authorized Problem fields and non-hidden TestCase data may be sent to Student interfaces.

**TBD:**
- Whether Practice Problems includes only isPublic=true.
- Whether it also includes College/Department Problems assigned to the Student.
- Search/filter/sort/pagination behavior.
- Bookmark/report features.
- Whether Students see all non-hidden TestCases or designated samples only.

### PROB-018 — Search and filtering

**Source gap:** Problem pages are mapped, but list behavior is not specified.

Confirm:
- Search by title/statement.
- Difficulty and tag filters.
- Language filter.
- College/Department/Section/academicBatch filters.
- Public/private filter for staff.
- Linked Course/Lab/Assessment/Contest filters.
- Sorting and pagination.
- Role-specific filter availability.

### PROB-019 — Edits and historical submission integrity

Problem has updatedAt; CodeSubmission references Problem and records verdict/runtime fields. The schema reconstruction does not show Problem version or TestCase snapshot.

**TBD:**
- Whether title, statement, limits, languages, and tags can change after publication.
- Whether TestCases can change after submissions exist.
- Whether past submissions are interpreted against current or historical TestCases.
- Whether submission records require Problem/TestCase version references.
- Whether archived Problems remain visible in historical activity.
- Soft-delete/archive vs hard-delete.

### PROB-020 — Ownership and creator attribution

**Source-confirmed schema:** createdById.

**Derived requirement:** set creator attribution from authenticated server context, not from a client-provided ID.

**TBD:** ownership transfer, editing after creator's role/department changes, creator retention after archive, and privileged editing/impersonation (coordinate with Module 1 and Audit Logging).

---

## 6. High-level workflows

### 6.1 Problem authoring — conceptual outline

The sources confirm Problem Management pages but not the full authoring sequence.

Proposed outline for review:
1. Authorized staff opens Problem Management.
2. Staff enters title, statement, difficulty, tags, organization scope, language list, and resource limits.
3. Staff creates TestCases and sets order and hidden/visible classification.
4. Staff reviews visibility and scope.
5. Problem becomes available to eligible users/activities under the approved access model.

The detailed actions and status transitions in this outline are proposed/derived, not source-confirmed.

### 6.2 Student Problem access — source-grounded outline

1. Student opens Practice Problems, My Courses, a Lab, Assessment, or Contest.
2. Platform resolves Student identity and activity/organization context.
3. Platform retrieves authorized Problems.
4. Student sees Problem statement and permitted metadata/sample data.
5. Student proceeds to code and submit through Module 6.

### 6.3 TestCase authoring — conceptual outline

1. Authorized staff opens a Problem's TestCase view.
2. Staff adds input and expectedOutput.
3. Staff sets hidden/visible flag and orderIndex.
4. System validates the TestCase and its Problem association.
5. TestCases are made available to evaluation workflow in Module 7.

Detailed validation and execution behavior remain TBD.

---

## 7. Data model reference

### 7.1 Problem

| Field | Type | Meaning |
|---|---|---|
| id | varchar | Problem identifier |
| collegeId | varchar | College association |
| departmentId | varchar | Department association |
| createdById | varchar | Creator |
| academicBatch | varchar | Academic batch |
| title | varchar(255) | Problem title |
| statement | text | Problem statement |
| difficulty | ProblemDifficulty | EASY / MEDIUM / HARD |
| tags | text[] | Problem tags |
| timeLimitMs | int | Time limit in milliseconds |
| memoryLimitMb | int | Memory limit in MB |
| allowedLanguages | text[] | Allowed languages |
| isPublic | boolean | Public visibility flag; semantics TBD |
| scope | ContentScope | DEPT_WIDE / SECTION_ONLY / GLOBAL |
| createdAt, updatedAt | timestamp | Record timestamps |

### 7.2 TestCase

| Field | Type | Meaning |
|---|---|---|
| id | varchar | TestCase identifier |
| problemId | varchar | Parent Problem |
| input | text | Test input |
| expectedOutput | text | Expected output |
| isHidden | boolean | Hidden/visible classification |
| orderIndex | int | TestCase ordering |
| createdAt | timestamp | Creation timestamp |

### 7.3 Related entities

- CourseProblem — Course/Problem association
- LabAssignment — Course/Problem association
- AssessmentProblem — Assessment/Problem association
- ContestProblem — Contest/Problem association
- GroupCourseProblem — Group/Course/Problem association
- CodeSubmission — Student submission for a Problem

Their workflow details belong to their modules.

### 7.4 Relevant enums

**ProblemDifficulty:** EASY, MEDIUM, HARD  
**ContentScope:** DEPT_WIDE, SECTION_ONLY, GLOBAL

PublishStatus exists in the shared schema, but Problem does not show a status field in the reconstruction.

---

## 8. Validation and consistency requirements

These are derived safeguards; business-specific ranges/policies remain TBD.

| Rule ID | Validation | Status |
|---|---|---|
| PROB-V01 | Problem must reference valid College/Department where tenant-scoped. | Derived; global/null behavior TBD |
| PROB-V02 | Department must belong to Problem College when both are specified. | Derived |
| PROB-V03 | createdById must come from authenticated actor context. | Derived |
| PROB-V04 | difficulty must be EASY, MEDIUM, or HARD. | Source enum |
| PROB-V05 | scope must be DEPT_WIDE, SECTION_ONLY, or GLOBAL. | Source enum |
| PROB-V06 | timeLimitMs and memoryLimitMb must satisfy approved ranges. | Range TBD |
| PROB-V07 | allowedLanguages must use approved platform language identifiers. | Catalogue TBD |
| PROB-V08 | TestCase must reference an existing Problem. | Derived |
| PROB-V09 | TestCase input/output size and format must satisfy approved limits. | Limits TBD |
| PROB-V10 | TestCases must have deterministic orderIndex ordering. | Derived |
| PROB-V11 | Student-facing APIs must not expose hidden TestCase input/expectedOutput. | Security safeguard |
| PROB-V12 | Problem retrieval must enforce College/Department/Section/scope/public rules server-side. | Derived |
| PROB-V13 | Referenced Problems must not be hard-deleted unless approved retention/cascade rules permit it. | Policy TBD |
| PROB-V14 | Problem/TestCase edits must preserve submission history under an approved versioning policy. | TBD |

---

## 9. Source gaps, conflicts, and risks

### 9.1 Problem publication status is absent

Problem has isPublic and scope but no status field in the reconstructed schema.

**Risk:** draft/published/archive behavior could be implemented inconsistently.

**Decision:** confirm whether Problem uses a lifecycle field or whether visibility is controlled by isPublic, scope, and activity association.

### 9.2 Public visibility and tenant scope may conflict

Problem has both isPublic and scope.

**Risk:** public visibility could unintentionally bypass College/Department restrictions.

**Decision:** define a visibility matrix across role, isPublic, scope, College, Department, academicBatch, and assignment context.

### 9.3 Hidden TestCase confidentiality

isHidden exists but API behavior is unspecified.

**Risk:** hidden inputs/expected outputs could leak through API payloads, client state, or errors.

**Decision:** define Student vs staff response payloads and permissible evaluation feedback.

### 9.4 TestCase edits may invalidate historical results

CodeSubmission stores Problem reference and verdict/runtime fields, but no Problem version/TestCase snapshot is shown.

**Risk:** old verdicts may not match the current Problem/TestCase set.

**Decision:** choose versioning/snapshot strategy and determine whether rejudging is supported.

### 9.5 Language identifiers and resource limits are incomplete

allowedLanguages is text[]; canonical language identifiers are absent. timeLimitMs and memoryLimitMb have no defaults/ranges in source.

**Decision:** coordinate language catalogue and resource-limit configuration with Module 6.

### 9.6 Problem deletion and reuse

Problems are referenced by multiple activity types.

**Risk:** deletion or edits may disrupt Courses, Labs, Assessments, Contests, and submission history.

**Decision:** define archive/delete and linked-activity behavior.

---

## 10. Decisions required before finalization

| ID | Decision | Why it matters |
|---|---|---|
| PROB-D01 | Define create/edit/publish/archive/delete permissions for HOD, Manager, SUPER_ADMIN, and other roles. | RBAC |
| PROB-D02 | Confirm whether Problem has a lifecycle status field. | Publication |
| PROB-D03 | Define isPublic meaning (global public, authenticated platform users, College-wide, etc.). | Access |
| PROB-D04 | Define combined isPublic/scope/College/Department/batch/activity visibility rules. | Tenant security |
| PROB-D05 | Define ContentScope semantics for Problems. | Visibility |
| PROB-D06 | Define required Problem fields and statement editor/format. | Authoring |
| PROB-D07 | Define title uniqueness and validation. | Data quality |
| PROB-D08 | Define tag catalogue/free-form behavior, casing, and duplicate rules. | Filtering |
| PROB-D09 | Define academicBatch format and multi-batch support. | Eligibility |
| PROB-D10 | Define allowedLanguages identifiers and empty-list behavior. | Judge0 |
| PROB-D11 | Define time/memory defaults, min/max, and activity override policy. | Evaluation |
| PROB-D12 | Define TestCase required fields, count/size limits, and import behavior. | TestCase authoring |
| PROB-D13 | Define orderIndex convention, uniqueness, and ordering controls. | Evaluation determinism |
| PROB-D14 | Define isHidden behavior and Student/staff API payload differences. | Confidentiality |
| PROB-D15 | Define visible/sample TestCase behavior and whether they are evaluated. | UX/evaluation |
| PROB-D16 | Define Problem reuse across Courses, Labs, Assessments, Contests, and Groups. | Shared content |
| PROB-D17 | Define whether edits affect linked activities immediately or use snapshots. | Versioning |
| PROB-D18 | Define historical submission behavior after Problem/TestCase edits. | History |
| PROB-D19 | Define archive/delete behavior for referenced Problems and TestCases. | Referential integrity |
| PROB-D20 | Define Problem search/filter/sort/pagination per role/page. | UI |
| PROB-D21 | Define Practice Problems inclusion rules for public, Department, Section, and assigned Problems. | Student UX |
| PROB-D22 | Define HOD/Manager access to CodeSubmission history from Problem pages. | Data access |
| PROB-D23 | Define cross-College/Department Problem sharing. | Tenant model |
| PROB-D24 | Define Problem publication/assignment notifications (Module 13). | Notifications |
| PROB-D25 | Define Problem/TestCase audit events and metadata (Module 15). | Audit |

---

## 11. Acceptance checklist

Problem Management is implementation-ready when:
- [ ] Problem permissions and ownership are explicit.
- [ ] Problem lifecycle and publication semantics are confirmed.
- [ ] isPublic and ContentScope visibility rules are approved.
- [ ] College, Department, and academicBatch rules are defined.
- [ ] Difficulty and tag behavior is finalized.
- [ ] allowedLanguages identifiers and empty-list behavior are defined.
- [ ] Resource-limit defaults and ranges are approved.
- [ ] TestCase creation, ordering, and hidden/visible behavior are finalized.
- [ ] Hidden TestCase confidentiality is verified in API and UI.
- [ ] Problem reuse across academic activities is defined.
- [ ] Problem/TestCase edits preserve submission history under an approved policy.
- [ ] Archive/delete behavior is finalized.
- [ ] HOD, Manager, and Student access is explicitly scoped.
- [ ] Search/filter/list requirements are approved.
- [ ] Module 6–7 interfaces for language mapping and evaluation are agreed.

---

## 12. Dependencies and interfaces

- **Module 1 — Authentication & User Lifecycle:** identity, role, active status, organization membership.
- **Module 2 — College / Department / Section Management:** tenant and academic scope.
- **Module 4 — Course & Enrollment:** CourseProblem association and Student course access.
- **Module 6 — Code Submission & Judge0:** allowed-language mapping, submission interface, resource limits, Judge0.
- **Module 7 — Test Case Evaluation:** hidden/visible execution, expected-output comparison, verdict behavior.
- **Module 8 — Labs & Lab Scoring:** Problem assignment to Labs.
- **Module 9 — Assessments:** AssessmentProblem association and access.
- **Module 10 — Groups:** group-specific Problem relationships.
- **Module 11 — Contests:** ContestProblem association and contest access.
- **Module 13 — Notifications:** Problem publish/assignment events, if required.
- **Module 15 — Audit Logging:** Problem/TestCase create/edit/archive events.
- **Module 17 — Functional State Machines:** Problem/TestCase lifecycle, if applicable.
- **Module 18 — Edge Cases & Failure Handling:** visibility, concurrent edits, deletion, versioning.

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

**Review gate:** Module 5 only. Do not begin Module 6 until the user reviews this document and explicitly approves proceeding.

---

# Functional Specification Standard v1.0 compliance expansion

> This expansion supplements source-grounded PROB-001–PROB-020 above. Those IDs are source requirement anchors. The stable function IDs below use PROB-Fxxx. Existing PROB-D01–D25 remain open unless the product owner explicitly resolves them.

## 14. Evidence and status labels

- **Source-confirmed:** explicitly in Business Flow, Page-to-DB Mapping, RBAC, or schema.
- **Derived:** logical safeguard from confirmed facts, such as server-side authorization and preventing hidden TestCase exposure.
- **TBD / Decision Required:** absent, conflicting, or requiring approval.
- **Recommendation:** suggestion only; not approved.
- **Legacy-verified:** none; no legacy implementation was inspected.
- **Target requirement:** none approved yet; this document is a review draft.

No UI behavior beyond source-mapped page names is treated as confirmed. No implementation/API behavior is claimed to have been verified.

## 15. Terminology

| Term | Definition | Status |
|---|---|---|
| Problem | Coding task with metadata and statement, referenced by academic activities and submissions. | Schema/Business Flow |
| TestCase | Input/expected-output record belonging to a Problem. | Schema |
| Visible TestCase | TestCase with isHidden=false. Whether it is a sample or evaluated is TBD. | Schema flag |
| Hidden TestCase | TestCase with isHidden=true. Student payload confidentiality is a derived safeguard. | Schema flag |
| Problem scope | ContentScope enum DEPT_WIDE, SECTION_ONLY, GLOBAL. | Enum confirmed; semantics TBD |
| Public flag | Problem.isPublic boolean. Does not imply anonymous internet publication. | Schema; semantics TBD |
| Activity reference | CourseProblem, LabAssignment, AssessmentProblem, ContestProblem, or GroupCourseProblem relationship. | Schema/page mapping |
| Submission | CodeSubmission referencing a Problem and Student. Module 6 owns submission lifecycle; Module 7 owns evaluation. | Schema/module ownership |

## 16. Role, action, and scope matrix

This matrix records source grants and conflicts without reconciling them silently.

| Role | View Problem | Create | Update metadata/TestCases | Delete | Publish/archive | Solve/submit | Scope |
|---|---|---|---|---|---|---|---|
| SUPER_ADMIN | Full CRUD source grant | Allow | Allow | Full CRUD per RBAC | Lifecycle still decision-gated | N/A | Platform/global |
| FINANCE_TEAM | Platform matrix says View; page/API matrices omit Problem | Not granted | Not granted | Not granted | Not granted | N/A | College; no Problem page mapped |
| HOD | Full CRUD | Allow | Allow | Full CRUD per RBAC | Decision-gated | N/A | College + Department |
| MANAGER | Platform Create/Update*; page/API V/C/U* | Conditional grant | Conditional grant | Not explicitly granted | Not explicitly granted | N/A | College + Department + assigned Section/permitted Group |
| STUDENT | View/Solve | Deny | Deny | Deny | Deny | Solve; submission creation belongs to Module 6 | Own + permitted academic resources |

**Conflict notes**
- RBAC platform matrix gives Manager Create/Update* on Problems & Test Cases; page/API matrices give V/C/U*. Delete is not explicitly granted.
- RBAC platform matrix gives Finance Team View; page/API matrices omit Problem access and no Finance Problem page is mapped. PROB-D01 must resolve this.
- HOD/SUPER_ADMIN Full CRUD is source-confirmed, but Problem lifecycle actions remain gated by the missing lifecycle decision.
- Student View/Solve does not permit hidden TestCase contents or other Students' submissions.
- Page visibility is not backend authorization.

## 17. Functional inventory

| Function ID | Function | Actor | Entry point | Type | Status |
|---|---|---|---|---|---|
| PROB-F001 | List/view Problem records | HOD/Manager/SUPER_ADMIN; Finance disputed | Problems page | Read | Draft |
| PROB-F002 | Create Problem | HOD/SUPER_ADMIN; Manager conditional | Problems/Create | Create | Draft |
| PROB-F003 | Update Problem metadata | HOD/SUPER_ADMIN; Manager conditional | Problem edit | Update | Draft |
| PROB-F004 | Set College/Department/batch | Authorized editor | Problem form | Configuration | TBD |
| PROB-F005 | Set scope/public visibility | Authorized editor | Problem form | Access config | TBD |
| PROB-F006 | Publish/unpublish/lifecycle | Staff | Problem action | Workflow | TBD |
| PROB-F007 | Archive/delete/restore Problem | Staff | Problem action | Lifecycle | TBD |
| PROB-F008 | Edit statement/content | Staff | Problem editor | Update | Draft |
| PROB-F009 | Manage difficulty/tags | Staff | Problem editor/list | Update | Draft |
| PROB-F010 | Configure languages/resource limits | Staff | Problem editor | Update | TBD |
| PROB-F011 | Create TestCase | Staff | Problem/TestCases | Create | Draft |
| PROB-F012 | Update TestCase | Staff | TestCase editor | Update | Draft |
| PROB-F013 | Delete TestCase | Staff | TestCase action | Delete | TBD |
| PROB-F014 | Reorder TestCases | Staff | TestCase list | Ordering | TBD |
| PROB-F015 | Set hidden/visible flag | Staff | TestCase editor | Classification | TBD |
| PROB-F016 | Student Practice Problem discovery | Student | Practice Problems | Read/search | TBD |
| PROB-F017 | Access Problem through Course | Student/staff | My Courses/Course | Read/integration | Draft |
| PROB-F018 | Access Problem through Lab | Student/staff | Lab | Read/integration | Draft |
| PROB-F019 | Access Problem through Assessment | Student/staff | Assessment | Read/integration | Draft |
| PROB-F020 | Access Problem through Contest | Student/staff | Contest | Read/integration | Draft |
| PROB-F021 | Search/filter/sort/paginate | Staff/Student | Problem list | Query | TBD |
| PROB-F022 | View Problem-linked submissions | Staff/Student own | Problem detail | Read/integration | TBD |
| PROB-F023 | Preserve history after Problem/TestCase edits | System | Content mutation | Integrity | TBD |
| PROB-F024 | Audit/notification integration | System | Mutation/lifecycle | Integration | Module 13/15 owned |
| PROB-F025 | Enforce visibility across tenant/activity scopes | Backend | Every request | Authorization | Derived |

## 18. Detailed function specifications

The common input fields, source constraints, role scope, and validation IDs are defined in Sections 4, 7, and 8 above. Each function below references those tables rather than duplicating the same field definitions. Unknown fields/defaults/feedback are TBD and must not be implemented as assumptions.

### PROB-F001 — List and view Problem records
**Status:** Source page/entity mapping + derived authorization. **Actors:** HOD, Manager, SUPER_ADMIN; Finance conditional; Student in academic/practice context.  
**Purpose:** return only authorized Problems and fields. **Preconditions:** authenticated/active actor; mapped Problems/Practice Problems/Course/Lab/Assessment/Contest route.  
**Inputs:** authenticated role/scope; optional Problem ID; activity context; query filters (contract TBD). Client role/scope are not trusted.  
**UI:** render authorized list/detail; actions shown only if separately authorized; Student response excludes hidden TestCase input/output.  
**Flow:** resolve actor → check role/action → apply College/Department/Section/group/activity scope → apply approved Problem visibility policy → query Problem → return safe fields.  
**Rules/validation:** PROB-V01–V05, V17, V21; public/scope policy remains PROB-D03–D05.  
**Backend/data:** read Problem and required associations; no mutation.  
**Success/failure/edge:** permitted record returned; foreign/invalid ID denied/not-found without disclosure; empty list state wording TBD; archived/draft behavior TBD.  
**Audit/privacy:** read logging Module 15; no hidden test data or unnecessary Student data.  
**Acceptance:** direct URL/API/list enforce same access; Student cannot fetch hidden TestCase values.  
**QA:** PROB-QA-001–010. **Decisions:** D01, D03–D05, D20–D23.

### PROB-F002 — Create Problem
**Status:** HOD/SUPER_ADMIN Full CRUD and Manager Create* source-confirmed; detailed form TBD.  
**Actors:** HOD/SUPER_ADMIN; Manager conditional. **Entry:** Problems/Create. **Preconditions:** role and scope pass; parent organization valid.  
**Inputs:** all Problem fields in Section 7; createdById is server-derived; requiredness/defaults TBD.  
**UI:** Problem form; field presentation/defaults/cancel destination TBD.  
**Flow:** open form → submit fields → backend derives creator → check role/scope/College-Department hierarchy → validate schema/enums/approved rules → persist Problem/timestamps → return result.  
**Rules:** no implicit publish, TestCase creation, activity association, or notification. Client cannot forge creator.  
**Validation:** PROB-V03–V13 and role scope.  
**Data:** insert Problem only; TestCase/association rows not implicitly created.  
**Success/failure/edge:** saved record returned; invalid/out-of-scope values create no record; global/null tenant and repeated submit behavior TBD.  
**Audit/privacy:** Module 15 Problem-create event; creator attribution server-side.  
**Acceptance:** HOD stays within Department; Manager create only after permission/scope rule confirmed; no unapproved side effects.  
**QA:** PROB-QA-011–020. **Decisions:** D01, D02, D06–D12, D23–D25.

### PROB-F003 — Update Problem metadata
**Actors:** HOD/SUPER_ADMIN; Manager conditional. **Entry:** authorized Problem edit. **Preconditions:** Problem exists and is in scope.  
**Inputs:** Problem ID and allowed fields from schema; id/createdAt/createdById are protected unless decision says otherwise. Scope/public/lifecycle fields use F005/F006.  
**UI:** load current values, edit allowed fields, save/cancel; exact editable field list TBD.  
**Flow:** load → authorize parent Problem → submit changes → re-check scope → validate → update approved fields/updatedAt → return updated record.  
**Rules:** no silent reparenting or widening visibility; edits after submissions follow F023; no implicit TestCase/association mutation.  
**Validation:** field/enums/parent/limits; stale update and uniqueness policy TBD.  
**Data:** update only allowlisted fields. **Failure:** foreign/missing record or invalid fields rejected without partial update.  
**Edge:** concurrent edits, actor scope change, active activity/submissions.  
**Audit/privacy:** Module 15 changed-field event; do not log full statement by default.  
**Acceptance:** scope cannot be widened by client IDs; historical submissions remain unchanged.  
**QA:** PROB-QA-021–028. **Decisions:** D01, D06–D11, D17–D19, D22, D25.

### PROB-F004 — Set College, Department, and academic batch
**Status:** schema fields confirmed; requiredness/null/global semantics TBD. **Actors:** authorized editor. **Entry:** create/edit form.  
**Inputs:** collegeId, departmentId, academicBatch.  
**Flow:** select values → validate actor authority and parent relationship → save fields → access policy consumes values only after D04/D05/D09.  
**Rules:** Department must belong to College when both are set (derived); Manager cannot select foreign scope; batch is varchar, not an enum/multi-batch field in schema.  
**Validation/failure:** invalid parent/out-of-scope actor rejects; nullable/global behavior unresolved.  
**Data:** update organization/batch fields; downstream access impact uses F023.  
**Success/edge:** saved consistent hierarchy; changing Department after assignments/submissions is decision-gated.  
**Audit:** Module 15. **Acceptance:** no cross-tenant reparenting; batch is not treated as eligibility boundary absent approval.  
**QA:** PROB-QA-029–034. **Decisions:** D04, D05, D09, D23.

### PROB-F005 — Set scope and public visibility
**Status:** enum/boolean source-confirmed, semantics TBD. **Actors:** authorized editor. **Inputs:** ContentScope and isPublic. **Entry:** Problem access configuration.  
**UI:** scope choices and explanatory labels TBD; do not label public as “internet-wide” without approval.  
**Flow:** choose values → check role → validate enum/boolean → save → apply approved visibility matrix.  
**Rules:** public and scope are distinct; GLOBAL may not mean cross-College; isPublic does not bypass tenant/ownership; changing visibility must not silently widen access.  
**Validation:** invalid enum/unauthorized scope change rejects; combinations follow D03–D05/D23.  
**Data:** update scope/isPublic only; cache/index side effects TBD.  
**Failure/edge:** public+SECTION_ONLY, null organization, linked Course/Lab/Assessment/Contest, active submissions.  
**Audit/privacy:** security-sensitive change; Module 15.  
**Acceptance:** one approved access matrix governs list/detail/direct URL/activity.  
**QA:** PROB-QA-035–043. **Decisions:** D03–D05, D17, D23, D25.

### PROB-F006 — Publish/unpublish/lifecycle transition
**Status:** blocked; Problem has no status field in reconstructed schema. **Actors:** HOD/SUPER_ADMIN; Manager TBD.  
**Entry/input:** status action and target state only if approved.  
**UI/flow:** no publish action may be finalized until D02 defines status field, state transitions, and publish requirements. A possible sequence (authorize → validate content → transition → audit) is illustrative, not approved.  
**Rules:** generic PublishStatus enum for other entities does not prove Problem uses it; isPublic is not a lifecycle status. Module 17 owns canonical transition.  
**Data:** no status mutation until schema/state decision.  
**Failure/edge:** incomplete content, unpublish while assigned, archived Problem used in active activity, Manager publish.  
**Acceptance:** decision-blocked; no lifecycle implementation before D02.  
**QA:** PROB-QA-044–049 blocked. **Decision:** D02.

### PROB-F007 — Archive/delete/restore Problem
**Status:** SUPER_ADMIN/HOD Full CRUD source grant; Manager delete not granted; schema has no Problem archive/deleted field.  
**Actors:** HOD/SUPER_ADMIN. **Entry:** Problem actions, UI details TBD. **Input:** Problem ID/action/confirmation TBD.  
**Flow:** authorize → inspect Course/Lab/Assessment/Contest/Group/CodeSubmission references → apply approved retention policy → return outcome.  
**Rules:** no cascade deletion or soft-delete field is assumed; Manager cannot delete without explicit grant.  
**Data:** blocked pending D19; no delete of linked entities by assumption.  
**Failure/edge:** linked submissions, active activities, repeated deletion, restore, foreign scope.  
**Audit:** destructive event Module 15.  
**Acceptance:** no hard delete/cascade until policy approved.  
**QA:** PROB-QA-050–056 blocked. **Decisions:** D01, D02, D16, D19, D25.

### PROB-F008 — Edit Problem statement/content
**Actors:** HOD/SUPER_ADMIN; Manager conditional. **Entry:** Problem editor. **Input:** title and statement text.  
**UI:** editor, preview, Markdown/HTML/rich text, autosave and sanitization are TBD.  
**Flow:** load → edit → server validates title/content policy → save → render safely to authorized viewers.  
**Rules:** statement format is not specified; no structured constraints/sample/editorial fields are assumed. Changes after submission use F023.  
**Validation:** title ≤ varchar(255); statement length/markup/sanitization rules TBD.  
**Data:** update title/statement/updatedAt only.  
**Failure/edge:** malicious markup, concurrent edits, active Assessment/Contest, old submissions.  
**Audit/privacy:** Module 15; avoid logging full statement.  
**Acceptance:** safe render and authorized editing; old submissions remain traceable.  
**QA:** PROB-QA-057–062. **Decisions:** D06, D07, D17, D18.

### PROB-F009 — Manage difficulty and tags
**Actors:** authorized Problem editor. **Inputs:** ProblemDifficulty enum, tags text[]. **Entry:** create/edit/list.  
**Flow:** select difficulty/tags → validate enum and approved tag policy → persist → list/search uses values.  
**Rules:** difficulty enum EASY/MEDIUM/HARD confirmed; tag catalogue/free-form/casing/duplicates/scope TBD. Difficulty does not imply score or TestCase count.  
**Validation:** invalid difficulty rejected; tag constraints TBD.  
**Data:** update difficulty/tags only.  
**Failure/edge:** duplicate/case variants, tag edits after publish, filter normalization.  
**Acceptance:** enum enforced; tags conform to approved policy. **QA:** PROB-QA-063–068. **Decisions:** D07–D09/D20.

### PROB-F010 — Configure languages and resource limits
**Actors:** authorized editors; Module 6/7 consumers. **Inputs:** allowedLanguages text[], timeLimitMs int, memoryLimitMb int.  
**Flow:** choose canonical language IDs/limits → validate shared catalogue/ranges → save → Module 6 maps identifiers to Judge0; Module 7 uses evaluation configuration.  
**Rules:** defaults, bounds, language identifiers, empty-list meaning, per-language limits, activity overrides are TBD. Do not rejudge old submissions automatically.  
**Validation:** only approved IDs/ranges accepted; empty list follows D10.  
**Data:** update fields only.  
**Failure/edge:** language removed after submissions, queued execution while limits change, Judge0 mapping unavailable.  
**Acceptance:** consistent with Module 6/7 policy; no unsupported interpretation. **QA:** PROB-QA-069–076. **Decisions:** D10, D11, D17, D18.

### PROB-F011 — Create TestCase
**Actors:** HOD/SUPER_ADMIN; Manager conditional. **Entry:** Problem → TestCases. **Precondition:** parent Problem exists and is editable.  
**Inputs:** problemId (required), input, expectedOutput, isHidden, orderIndex; requiredness/defaults/size limits TBD.  
**UI:** TestCase editor and list; import/bulk behavior TBD.  
**Flow:** choose Problem → enter input/output → set hidden/order if allowed → server checks parent scope/content/size/order → create TestCase with createdAt → refresh list.  
**Rules:** no execution or submission occurs during authoring; hidden payload is confidential to Student.  
**Validation:** valid parent, input/output size, boolean, ordering.  
**Data:** insert TestCase only. **Failure:** invalid/foreign parent or invalid payload creates no row.  
**Audit/privacy:** Module 15 create event; protect hidden content.  
**Acceptance:** valid TestCase attaches to correct Problem and hidden values never appear in Student payload.  
**QA:** PROB-QA-077–084. **Decisions:** D01, D12–D15, D25.

### PROB-F012 — Update TestCase
**Actors:** HOD/SUPER_ADMIN; Manager conditional. **Entry:** TestCase edit. **Inputs:** TestCase ID and input/output/hidden/order fields; parent reassignment not ordinary edit.  
**Flow:** load → authorize parent → edit → validate → update → return.  
**Rules:** content changes may invalidate historic results; no automatic rejudge or old submission rewrite. Schema has createdAt but no TestCase updatedAt.  
**Validation:** parent, size, boolean/order and concurrency policy.  
**Data:** update allowed fields only.  
**Failure/edge:** foreign Problem, active judge, active contest/assessment, hidden-to-visible change.  
**Acceptance:** scoped update and no implicit history mutation. **QA:** PROB-QA-085–090. **Decisions:** D12–D18.

### PROB-F013 — Delete TestCase
**Status:** decision-blocked; TestCase delete semantics not described. **Actors:** HOD/SUPER_ADMIN; Manager delete not granted.  
**Entry/input:** delete action and TestCase ID, confirmation TBD.  
**Flow:** authorize → inspect active/history references → apply approved delete/version policy → report outcome.  
**Rules:** no cascade/rejudge or hard delete inferred; Module 7 owns evaluation dependency.  
**Data:** deletion/retention blocked until D19.  
**Failure/edge:** active evaluation, historical submission, deleting final TestCase, repeated delete.  
**Acceptance:** no deletion before policy approval; Manager cannot delete by inference. **QA:** PROB-QA-091–095 blocked. **Decisions:** D01, D12, D18, D19.

### PROB-F014 — Reorder TestCases
**Actors:** authorized staff. **Entry:** TestCase list. **Inputs:** TestCase IDs/orderIndex values; 0/1 origin and uniqueness TBD.  
**UI:** reorder controls/drag behavior TBD. **Flow:** reorder → verify all IDs belong to same Problem → validate indices → save ordering → return deterministic list.  
**Rules:** orderIndex exists; uniqueness/contiguity and failedTestCaseIndex relationship are TBD.  
**Data:** update orderIndex for affected cases; atomicity policy TBD.  
**Failure/edge:** duplicate/negative indices, mixed Problem, partial update, concurrent reorder.  
**Acceptance:** valid ordering is deterministic; invalid reorder leaves no partial mutation. **QA:** PROB-QA-096–100. **Decisions:** D13/D18/D25.

### PROB-F015 — Set hidden/visible classification
**Actors:** authorized Problem/TestCase editors. **Input:** isHidden boolean. **Entry:** TestCase create/edit.  
**Flow:** set flag → authorize → validate → save → Student-facing APIs apply approved response policy.  
**Rules:** schema confirms boolean; whether visible means sample and whether visible cases are evaluated are Module 7 decisions. Hidden input/expectedOutput must not be sent in Student payloads, error details, or routine logs.  
**Data:** update isHidden only; no rejudge. **Failure/edge:** hidden→visible with submissions, active evaluation, direct API fetch.  
**Acceptance:** Student cannot fetch hidden values; staff access follows RBAC. **QA:** PROB-QA-101–108. **Decisions:** D14/D15/D17/D18.

### PROB-F016 — Student Practice Problems discovery
**Actor:** Student. **Entry:** mapped Practice Problems page. **Inputs:** authenticated Student; query/filter from F021.  
**Flow:** derive Student identity → resolve approved public/Department/Section/batch/assigned eligibility → apply scope → return safe Problem list → open selected Problem.  
**Rules:** page mapping confirms page/entities but not inclusion criteria. Public/scope semantics remain open; hidden TestCase values never included.  
**Data:** read Problem/approved relations only. **Failure/edge:** empty results, changed batch/Section, archive status, subscription/College status, public-scope conflict.  
**Acceptance:** only eligible Problems are listed; direct URL/API checks match list. **QA:** PROB-QA-109–116. **Decisions:** D03–D05, D09, D20–D21.

### PROB-F017 — Access Problem through Course
**Actors:** Student/staff permitted by Module 4. **Entry:** My Courses/Course. **Owner:** Module 4 association, Module 5 Problem access.  
**Flow:** Module 4 authorizes Course → resolve CourseProblem → Module 5 checks Problem scope/visibility → return safe content → Module 6 handles submission.  
**Rules:** CourseProblem does not bypass Problem access; Problem view does not grant enrollment/submission; GroupCourseProblem is separate.  
**Inputs:** Course ID/Problem ID/actor; association and eligibility validation.  
**Failure/edge:** stale link, archived Problem, enrollment removed, scope mismatch, hidden testcase request.  
**Acceptance:** Course route and direct Problem route enforce equivalent authorization. **QA:** PROB-QA-117–122. **Decisions:** D04/D05/D16/D17; Module 4.

### PROB-F018 — Access Problem through Lab
**Actors:** Student/staff under Module 8. **Entry:** Lab assignment. **Inputs:** LabAssignment ID, Problem ID, actor.  
**Flow:** authorize Lab → validate assignment/membership/time → resolve Problem → apply Problem visibility → return statement → submission/scoring remain Modules 6/8.  
**Rules:** Lab assignment does not make Problem public; hidden testcase protection remains; edits/deletion use F023.  
**Failure/edge:** expired assignment, nonmember, archived Problem, invalid relation.  
**Acceptance:** both Lab and Problem permissions pass; no bypass via Problem ID. **QA:** PROB-QA-123–127. **Decisions:** D04/D05/D16–D18; Module 8.

### PROB-F019 — Access Problem through Assessment
**Actors:** Student/staff under Module 9. **Entry:** Assessment. **Inputs:** Assessment ID, Problem ID, actor.  
**Flow:** authorize Assessment/attempt → validate AssessmentProblem → apply Problem access → return content → Module 6/7 handle code/evaluation.  
**Rules:** Assessment assignment does not bypass Problem scope; assessment timing/attempt rules remain Module 9; snapshot policy unresolved.  
**Failure/edge:** not in window, no attempt permission, Problem edit during attempt, stale link.  
**Acceptance:** only eligible assessment actor receives Problem content. **QA:** PROB-QA-128–132. **Decisions:** D04/D05/D16–D18; Module 9.

### PROB-F020 — Access Problem through Contest
**Actors:** participants/staff under Module 11. **Entry:** Contest problem page. **Inputs:** Contest ID, Problem ID, actor.  
**Flow:** authorize Contest/participant → validate ContestProblem → apply Problem access/version policy → return statement → submission/scoring remains Modules 6/7/11.  
**Rules:** contest access does not make Problem globally public; contest assignment does not grant edit permission; snapshot/version policy unresolved.  
**Failure/edge:** nonparticipant, ended contest, foreign College, archived/changed Problem.  
**Acceptance:** contest and Problem authorization both pass; guessed Problem ID cannot bypass participation. **QA:** PROB-QA-133–137. **Decisions:** D04/D05/D16–D18; Module 11.

### PROB-F021 — Search/filter/sort/paginate
**Actors:** staff/Student within allowed page. **Inputs:** search term, difficulty, tags, language, batch, organization/activity, sort/page/size; exact list TBD.  
**Flow:** authenticate → scope filter first → validate query → search only authorized content → return safe results/page metadata.  
**Rules:** filters cannot widen scope; Student cannot use staff-only public/private or tenant filters; snippets must not disclose hidden/private statement content.  
**Validation:** unsupported filters/sort, invalid page/size, oversized query → reject/normalize per API contract TBD.  
**Data:** read only. **Acceptance:** all pages/search results retain scope; no foreign data leakage. **QA:** PROB-QA-138–145. **Decisions:** D20/D21/D23.

### PROB-F022 — View Problem-linked submission history
**Status:** Problem page maps CodeSubmission; RBAC Code Submission matrix: HOD View, Manager View*, Student Create/View own; Module 6 owns lifecycle.  
**Actors:** HOD/Manager/SUPER_ADMIN within scope; Student own submissions; Finance not mapped. **Entry:** Problem detail/history tab if approved.  
**Flow:** authorize Problem → separately authorize submission view → scope/ownership filter → return permitted fields.  
**Rules:** Problem view alone does not grant submission access; Student sees own records only; Manager Section scope; code/stdout/stderr visibility remains Module 6/7 policy.  
**Data:** read CodeSubmission only. **Failure/edge:** foreign Student, scope changes, private code/output.  
**Acceptance:** no submission/source-code/result disclosure outside permission. **QA:** PROB-QA-146–151. **Decisions:** D22/D25; Module 6.

### PROB-F023 — Preserve historical submission/evaluation meaning
**Status:** schema has Problem reference but no version/snapshot fields; policy TBD. **Actors:** system/editor/Modules 6–7. **Trigger:** Problem/TestCase edits, ordering, language/limit/visibility change, archive/delete.  
**Flow (decision-gated):** detect references/history → apply approved snapshot/version/immutable-content policy → retain traceable old results → serve current content under current policy.  
**Rules:** no automatic rejudge or old verdict reinterpretation; no version/snapshot assumed.  
**Data:** version fields require schema/architecture approval. **Failure/edge:** edit while Judge0 job runs, contest/assessment active, TestCase reorder changes failed index, delete referenced content.  
**Acceptance:** historical verdicts remain traceable and are not silently overwritten. **QA:** PROB-QA-152–158. **Decisions:** D17–D19; Modules 6/7/17/18.

### PROB-F024 — Audit and notification integration
**Actors:** system; Module 13 notification delivery, Module 15 audit catalog. **Triggers:** create/update/scope/public/lifecycle/TestCase mutation.  
**Flow:** approved mutation → emit audit event per Module 15 → notify only if Module 13 trigger/recipient is approved.  
**Rules:** no default notification assumption; avoid source code, full statement, hidden TestCase values, or sensitive Student data in routine audit payloads.  
**Inputs:** actor/action/target/scope/changed fields/outcome; schema owned by Module 15.  
**Failure:** audit/notification failure follows Modules 13/15/18. **Acceptance:** approved mutations are auditable; no unapproved notifications.  
**QA:** PROB-QA-159–163. **Decisions:** D24/D25.

### PROB-F025 — Enforce visibility across tenant/activity boundaries
**Status:** derived from RBAC and Problem fields. **Actor:** backend. **Trigger:** every read/write.  
**Flow:** authenticate → permission → College/Department/Section/group/activity scope → ownership/membership → approved Problem visibility → database operation.  
**Rules:** permission + scope + ownership required; isPublic/ContentScope/activity/batch use one approved matrix; client IDs/filters cannot widen scope.  
**Failure:** deny before protected mutation; safe response and audit per RBAC/Module 15; avoid revealing existence of foreign Problems.  
**Acceptance:** list/search/direct URL/API/activity context have consistent authorization and tenant isolation. **QA:** PROB-QA-164–170. **Decisions:** D01, D03–D05, D09, D23, D25.

## 19. Business rules and validation matrix

| Rule ID | Rule | Evidence/status |
|---|---|---|
| PROB-R01 | difficulty ∈ EASY/MEDIUM/HARD | Schema enum |
| PROB-R02 | scope ∈ DEPT_WIDE/SECTION_ONLY/GLOBAL | Schema enum; semantics TBD |
| PROB-R03 | Problem references College/Department/creator | Schema |
| PROB-R04 | Department belongs to College when both are assigned | Derived |
| PROB-R05 | createdById is server-derived on create | Derived |
| PROB-R06 | Backend enforces role, scope, ownership | RBAC |
| PROB-R07 | Student Problem visibility requires approved practice/activity eligibility | Source gap |
| PROB-R08 | isPublic does not automatically mean anonymous Internet access | Semantics TBD |
| PROB-R09 | scope/public flags do not bypass tenant boundaries | Derived security |
| PROB-R10 | Hidden TestCase input/expectedOutput are withheld from Student payloads | Derived security |
| PROB-R11 | Problem view does not grant Course/Lab/Assessment/Contest/submission permissions | Cross-module |
| PROB-R12 | CourseProblem/LabAssignment/AssessmentProblem/ContestProblem/GroupCourseProblem are distinct links | Schema |
| PROB-R13 | Problem/TestCase edits do not automatically rejudge old submissions | TBD |
| PROB-R14 | No cascade deletion without approved policy | Derived |
| PROB-R15 | Manager create/update are scoped; delete not explicitly granted | RBAC |
| PROB-R16 | Finance Team Problem View is conflicting across matrices | Source conflict |
| PROB-R17 | Problem lifecycle cannot be inferred from PublishStatus enum | Schema gap |
| PROB-R18 | language/resource configuration aligns with Modules 6–7 | Dependency |

| Validation ID | Condition | Expected behavior | Decision |
|---|---|---|---|
| PROB-V01 | No action permission | Deny | D01 |
| PROB-V02 | Missing/invalid Problem ID | Safe not-found | API policy TBD |
| PROB-V03 | College/Department mismatch | Reject | Derived |
| PROB-V04 | Actor supplies foreign organization/Section | Deny | RBAC |
| PROB-V05 | title >255 chars | Reject | Schema |
| PROB-V06 | invalid difficulty | Reject | Schema |
| PROB-V07 | invalid scope | Reject | Schema |
| PROB-V08 | isPublic not boolean | Reject | Schema |
| PROB-V09 | invalid academicBatch | Apply approved format | D09 |
| PROB-V10 | invalid/duplicate tag | Apply approved tag policy | D08 |
| PROB-V11 | unsupported language | Reject after catalogue is approved | D10 |
| PROB-V12 | empty allowedLanguages | Apply approved empty-list semantics | D10 |
| PROB-V13 | resource limit outside range | Reject | D11 |
| PROB-V14 | TestCase missing/foreign Problem | Reject | Derived |
| PROB-V15 | input/output exceeds limit | Reject | D12 |
| PROB-V16 | invalid orderIndex | Reject/normalize per approved policy | D13 |
| PROB-V17 | Student requests hidden TestCase | Deny/omit hidden values | D14 |
| PROB-V18 | lifecycle unknown | Block pending decision | D02 |
| PROB-V19 | deletion with history/references | Apply retention decision; no cascade assumption | D19 |
| PROB-V20 | edit after submissions | Apply version/history policy | D17/D18 |
| PROB-V21 | search/direct URL widens scope | Filter/deny server-side | RBAC |
| PROB-V22 | Manager delete/publish | Deny unless explicitly approved | D01 |
| PROB-V23 | Finance Problem access | Follow resolved matrix | D01 |
| PROB-V24 | activity membership invalid | Deny | Modules 4/8/9/11 |
| PROB-V25 | audit/notification failure | Follow Modules 13/15/18 | D24/D25 |

## 20. Data and state reference

### Problem schema

| Field | Type | Meaning | Status |
|---|---|---|---|
| id | varchar | Problem ID | generation TBD |
| collegeId | varchar | College association | nullability/global TBD |
| departmentId | varchar | Department association | nullability/global TBD |
| createdById | varchar | creator | server-derived at create |
| academicBatch | varchar | academic batch | format/requiredness TBD |
| title | varchar(255) | title | requiredness/uniqueness TBD |
| statement | text | content | editor/sanitization TBD |
| difficulty | ProblemDifficulty | EASY/MEDIUM/HARD | enum confirmed |
| tags | text[] | tags | catalogue/normalization TBD |
| timeLimitMs | int | time limit milliseconds | default/range TBD |
| memoryLimitMb | int | memory limit MB | default/range TBD |
| allowedLanguages | text[] | language IDs | catalogue/empty semantics TBD |
| isPublic | boolean | public flag | meaning/default TBD |
| scope | ContentScope | DEPT_WIDE/SECTION_ONLY/GLOBAL | semantics TBD |
| createdAt/updatedAt | timestamp | timestamps | system |

### TestCase schema

| Field | Type | Meaning | Status |
|---|---|---|---|
| id | varchar | TestCase ID | generation TBD |
| problemId | varchar | parent Problem | required relation |
| input | text | input | size/empty rules TBD |
| expectedOutput | text | expected result | size/empty rules TBD |
| isHidden | boolean | hidden classification | default/semantics TBD |
| orderIndex | int | order | uniqueness/origin TBD |
| createdAt | timestamp | creation time | system; no updatedAt shown |

### Related entities and ownership

| Relation | Purpose | Owner |
|---|---|---|
| CourseProblem | Course–Problem link | Module 4 |
| LabAssignment | Lab/Problem assignment | Module 8 |
| AssessmentProblem | Assessment–Problem link | Module 9 |
| ContestProblem | Contest–Problem link | Module 11 |
| GroupCourseProblem | Group/Course/Problem relationship | Module 10 |
| CodeSubmission | Problem + Student submission | Module 6 |
| Evaluation/TestCase execution | run and compare test data | Module 7 |

### State reference

| Entity | Source state | Transition | Status |
|---|---|---|---|
| Problem | isPublic + ContentScope; no lifecycle status shown | publish/archive/delete | TBD; no canonical state machine |
| TestCase | isHidden; no lifecycle status | add/edit/delete/reorder | TBD |
| CodeSubmission | verdict fields | submit/evaluate | Modules 6–7 |
| activity association | association record | attach/detach | respective module |

PublishStatus exists in the schema for other entities; do not assume it applies to Problem without a decision.

## 21. Audit, notifications, privacy, and integration

- Module 13 owns notification delivery; PROB-D24 must identify triggers and recipients. No notification is assumed.
- Module 15 owns audit event catalog; PROB-D25 must define Problem/TestCase mutations, visibility/lifecycle changes, and ordering events.
- Hidden TestCase input/expectedOutput are confidential evaluation material and must not be returned to Student clients.
- Student submissions are governed by Module 6 ownership/access. Problem page cannot bypass that permission.
- Module 6 consumes language/resource fields; Module 7 consumes TestCases/hidden/order semantics; Modules 4/8/9/10/11 consume Problem associations.
- Avoid logging source code, full statements, hidden inputs/outputs, or sensitive Student data by default.

## 22. Cross-module dependencies

| Module | Dependency |
|---|---|
| 1 | authenticated user, role, active status |
| 2 | College/Department/Section hierarchy |
| 4 | CourseProblem and Course eligibility |
| 6 | language mapping, limits, submissions, Judge0 |
| 7 | hidden/visible execution, order, expected-output comparison, verdict |
| 8 | Lab assignment/access/scoring |
| 9 | AssessmentProblem/access/attempts |
| 10 | GroupCourseProblem/group membership |
| 11 | ContestProblem/participant/window |
| 13 | notification triggers/recipients |
| 15 | audit event catalog |
| 17 | lifecycle/state transitions |
| 18 | concurrency, partial failure, history/versioning |

## 23. Decision log status

PROB-D01–PROB-D25 from Section 10 remain **Open** unless explicitly approved by the product owner. No decision is resolved by this documentation commit.

| Decision | Owner | Status | Impact |
|---|---|---|---|
| D01 | Product/Security | Open | roles and action permissions |
| D02 | Product/Architecture | Open | lifecycle field/state |
| D03–D05 | Product/Security | Open | public/scope/tenant visibility |
| D06–D09 | Product/Academic | Open | statement/title/tags/batch |
| D10–D11 | Architecture/Judge0 owner | Open | languages and resource limits |
| D12–D15 | Product/QA/Evaluation | Open | TestCase authoring/order/hidden |
| D16–D19 | Product/Architecture | Open | reuse/version/history/delete |
| D20–D22 | Product/Security | Open | search/practice/submission visibility |
| D23 | Product/Security | Open | cross-tenant sharing |
| D24–D25 | Product/Ops | Open | notifications/audit |

## 24. Review and readiness checklist

- [ ] PROB-F001–F025 reviewed by Product/Security/Architecture.
- [ ] HOD/SUPER_ADMIN/Manager/Finance/Student permissions approved.
- [ ] Finance Team Problem View conflict resolved.
- [ ] Manager delete/publish/archive permissions resolved.
- [ ] isPublic + ContentScope + College/Department/Section/batch visibility approved.
- [ ] Problem lifecycle field and transitions approved (Module 17).
- [ ] Authoring fields/editor/sanitization approved.
- [ ] Tag and academicBatch rules approved.
- [ ] Language IDs and time/memory bounds aligned with Module 6.
- [ ] TestCase size/order/hidden behavior approved and tested.
- [ ] Student/staff TestCase payloads verified.
- [ ] Problem reuse/version/rejudge/delete policy approved.
- [ ] Activity access aligned with Modules 4/8/9/10/11.
- [ ] Audit/notification events aligned with Modules 13/15.
- [ ] QA test cases include expected results and decision-blocked flags.
- [ ] Source facts, derived safeguards, recommendations, and TBDs remain distinct.

**Review gate:** Module 5 only. GitHub inclusion versions this detailed draft; it does not approve target behavior or authorize implementation of unresolved decisions.
