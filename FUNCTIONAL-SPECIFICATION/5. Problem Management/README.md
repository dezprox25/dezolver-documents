# 5. Problem Management

**Document type:** Functional Specification  
**Version:** 1.0 — Adopted target baseline  
**Status:** Product decisions adopted (PROB-D01–PROB-D25); schema and engineering gates remain  
**Scope:** Problem authoring, metadata, access scope, publication, and TestCase management. Submission execution and evaluation belong to Modules 6–7.

> This baseline reconciles Business Flow (original/v2), Page–DB Mapping (original/v2), RBAC, target schema, Module 4 adopted decisions, Modules 6–7 interfaces, and reviewed legacy Module 05 documentation. Sections 25–27 and ADOPTED-DECISIONS.md are normative where earlier draft passages retain TBD wording.

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

### 3.1 Business Flow (original and v2)

The source flow establishes:
- HOD manages Problems within the Department academic structure.
- Manager manages Students and activities within the assigned Section.
- Students access Problems from Courses, write code, and submit.
- Problems are used in Course, Lab, and Contest workflows.
- Submission and evaluation occur downstream of Problem access.

It does not describe Problem authoring screens, publication rules, or detailed permissions.

### 3.2 Page-to-DB Mapping (original and v2)

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

The original and v2 mappings confirm entity/page associations, not complete UI behavior or all action permissions. Platform, page, and API RBAC matrices were also reviewed; adopted conflicts are resolved in Sections 16 and 25.

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

## 5. Source-level functional requirements and original open questions

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

## 9. Original source gaps, conflicts, and risks (superseded by adopted decisions)

### 9.1 Problem publication status is absent

Problem has isPublic and scope but no status field in the reconstructed schema; adopted lifecycle and schema delta are in Sections 25.4 and 26.

**Risk:** draft/published/archive behavior could be implemented inconsistently.

**Adopted:** Problem.status uses DRAFT/PUBLISHED/ARCHIVED; see Section 25.4 and schema delta Section 26.

### 9.2 Public visibility and tenant scope may conflict

Problem has both isPublic and scope.

**Risk:** public visibility could unintentionally bypass College/Department restrictions.

**Adopted:** see the normative visibility matrix in Section 25.3.

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

## 10. Adopted product decision register

All PROB-D01–PROB-D25 are adopted as the v1 target product baseline. The complete rationale and implementation implications are recorded in [ADOPTED-DECISIONS.md](./ADOPTED-DECISIONS.md). No PROB-D01–PROB-D25 remains an open product decision. Remaining work is schema migration, cross-module implementation, security verification and QA execution—not product-policy uncertainty.

## 11. Acceptance checklist

**Product policy — adopted**
- [x] Role/action/scope policy, including Finance denial and Manager Section restriction.
- [x] isPublic Practice discovery semantics; no anonymous/cross-College access.
- [x] Problem ContentScope semantics and Section audience relationship.
- [x] Problem DRAFT/PUBLISHED/ARCHIVED lifecycle.
- [x] Required authoring fields, Markdown safety, title/tag/batch rules.
- [x] Language identifiers, empty-list behavior and resource-limit policy.
- [x] TestCase create/edit/remove/order/visibility/evaluation contract.
- [x] Reuse across Courses/Labs/Assessments/Contests under dual authorization.
- [x] Immutable ProblemVersion snapshots and historical submission behavior.
- [x] Archive/no-hard-delete and history retention.
- [x] Search, Practice discovery, submission visibility, notifications and audit.

**Engineering/release — pending**
- [ ] Add Problem.status/publishedAt/current version and ProblemSection relation.
- [ ] Implement ProblemVersion snapshot persistence and CodeSubmission version reference.
- [ ] Update canonical RBAC and API/page matrices.
- [ ] Align language catalogue/resource caps with Module 6 and Module 7.
- [ ] Implement atomic TestCase bulk operations, stable order and hidden-field response DTOs.
- [ ] Integrate activity-version binding in Modules 4/8/9/10/11.
- [ ] Integrate Module 13 notification events and Module 15 audit.
- [ ] Execute 200 QA cases, security review and release sign-off.

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

**Source review:** original/v2 sources and legacy Module 05 docs reviewed. Adopted target decisions are normative in Section 25. Engineering implementation remains pending.

---

# Functional Specification Standard v1.0 compliance expansion

> This expansion supplements source-grounded PROB-001–PROB-020 above. Those IDs are source requirement anchors. The stable function IDs below use PROB-Fxxx. Existing PROB-D01–PROB-D25 are adopted; see ADOPTED-DECISIONS.md.

## 14. Evidence and status labels

- **Source-confirmed:** explicitly in Business Flow, Page-to-DB Mapping, RBAC, or schema.
- **Derived:** logical safeguard from confirmed facts, such as server-side authorization and preventing hidden TestCase exposure.
- **Historical source gap:** absent or conflicting in the sources; resolved for the target baseline by PROB-D01–PROB-D25.
- **Recommendation:** suggestion only; not approved.
- **Legacy documentation reviewed:** legacy Module 05 README/SPECIFICATION were reviewed as historical evidence only; this is not a claim that source code was re-executed or re-verified.
- **Adopted target behavior:** PROB-D01–PROB-D25; runtime implementation remains pending.

No implementation/API behavior is claimed to have been verified. Adopted product policy is recorded in Section 25; implementation evidence remains pending.

## 15. Terminology

| Term | Definition | Status |
|---|---|---|
| Problem | Coding task with metadata and statement, referenced by academic activities and submissions. | Schema/Business Flow |
| TestCase | Input/expected-output record belonging to a Problem. | Schema |
| Visible TestCase | TestCase with isHidden=false. Whether it is a sample or evaluated is TBD. | Schema flag |
| Hidden TestCase | TestCase with isHidden=true. Student payload confidentiality is a derived safeguard. | Schema flag |
| Problem scope | ContentScope enum DEPT_WIDE, SECTION_ONLY, GLOBAL. | Enum confirmed; adopted semantics in Section 25 |
| Public flag | Problem.isPublic boolean. Does not imply anonymous internet publication. | Schema; semantics TBD |
| Activity reference | CourseProblem, LabAssignment, AssessmentProblem, ContestProblem, or GroupCourseProblem relationship. | Schema/page mapping |
| Submission | CodeSubmission referencing a Problem and Student. Module 6 owns submission lifecycle; Module 7 owns evaluation. | Schema/module ownership |

## 16. Role, action, and scope matrix

This is the adopted v1 authorization baseline. Backend authorization is authoritative; page visibility is not a security boundary.

| Role | View Problem | Create | Update metadata/TestCases | Publish/unpublish | Archive/restore | Delete | Solve/submit |
|---|---|---|---|---|---|---|---|
| SUPER_ADMIN | All Colleges | Full | Full | Full | Full | No hard delete; archive only | N/A |
| FINANCE_TEAM | No Problem page/API access in v1 | No | No | No | No | No | N/A |
| HOD | Assigned College + Department | Full in Department | Full in Department | Full in Department | Full in Department | No hard delete; archive only | N/A |
| MANAGER | Assigned Section/activity scope | SECTION_ONLY own Section | Own Section Problems/TestCases | No | No | No | N/A |
| STUDENT | Own authorized Practice/activity Problems | No | No | No | No | No | Solve; submission intake belongs to Module 6 |

- HOD may create/manage SECTION_ONLY and DEPT_WIDE Problems only within assigned College + Department. HOD cannot widen to GLOBAL.
- SUPER_ADMIN alone may create/change GLOBAL. GLOBAL means all eligible active Sections in the owning College, never cross-College or anonymous.
- Manager Create/Update* is restricted to SECTION_ONLY Problems within the assigned Section. Manager may manage TestCases only for those Problems. Manager cannot publish, archive, restore, delete, or reparent.
- Finance platform-level View conflicts with page/API matrices, which omit Problem access. Least-privilege Page/API policy is adopted: no Problem page/API access.
- Student View/Solve does not expose hidden TestCase values or another Student's submissions.
- HOD/Manager submission history is separately scoped: HOD assigned Department; Manager assigned Section. Source code/raw output visibility remains Module 6 policy.
- Every list/search/direct ID/export/API/activity route applies College, Department, Section, status, audience, ownership and context checks server-side.

## 17. Functional inventory

| Function ID | Function | Actor | Entry point | Type | Adopted status |
|---|---|---|---|---|---|
| PROB-F001 | List/view Problem records | Authorized roles | Problems/Practice/activity | Read | Adopted |
| PROB-F002 | Create Problem | SUPER_ADMIN/HOD; Manager scoped | Problems/Create | Create | Adopted |
| PROB-F003 | Update Problem metadata | SUPER_ADMIN/HOD; Manager scoped | Problem edit | Update | Versioned after publish |
| PROB-F004 | Set College/Department/batch | Authorized editor | Problem form | Configuration | Adopted |
| PROB-F005 | Set scope/public visibility | SUPER_ADMIN/HOD; Manager Section-limited | Problem access | Configuration | Adopted |
| PROB-F006 | Publish/unpublish/lifecycle | SUPER_ADMIN/HOD | Problem action | Lifecycle | DRAFT/PUBLISHED/ARCHIVED |
| PROB-F007 | Archive/restore Problem | SUPER_ADMIN/HOD | Problem action | Lifecycle | No hard delete |
| PROB-F008 | Edit statement/content | Authorized editor | Problem editor | Update | Published changes create version |
| PROB-F009 | Manage difficulty/tags | Authorized editor | Problem editor/list | Update | Adopted |
| PROB-F010 | Configure languages/resource limits | Authorized editor | Problem editor | Configuration | Module 6/7 aligned |
| PROB-F011 | Create TestCase | Authorized editor | Problem/TestCases | Create | Draft revision only |
| PROB-F012 | Update TestCase | Authorized editor | TestCase editor | Update | Draft revision only |
| PROB-F013 | Remove TestCase | HOD/SUPER_ADMIN; Manager scoped | TestCase action | Delete | Draft revision only |
| PROB-F014 | Reorder TestCases | Authorized editor | TestCase list | Ordering | Zero-based deterministic order |
| PROB-F015 | Set hidden/visible classification | Authorized editor | TestCase editor | Classification | Hidden payload protected |
| PROB-F016 | Student Practice Problems discovery | Student | Practice Problems | Read/search | isPublic + published + scope |
| PROB-F017 | Access Problem through Course | Eligible Student/staff | My Courses/Course | Integration | Dual authorization |
| PROB-F018 | Access Problem through Lab | Eligible participant/staff | Lab | Integration | Dual authorization |
| PROB-F019 | Access Problem through Assessment | Eligible attempt/staff | Assessment | Integration | Dual authorization |
| PROB-F020 | Access Problem through Contest | Eligible participant/staff | Contest | Integration | Dual authorization |
| PROB-F021 | Search/filter/sort/paginate | Authorized staff/Student | Problem list | Query | Scope-first |
| PROB-F022 | View Problem-linked submissions | HOD/Manager/SUPER_ADMIN; Student own | Problem history | Read/integration | Module 6 field permissions |
| PROB-F023 | Preserve historical submission/evaluation meaning | System | Publish/edit/submit | Integrity | Immutable version snapshot |
| PROB-F024 | Audit and notification integration | System | Mutations/lifecycle | Integration | Module 13/15 contracts |
| PROB-F025 | Enforce visibility across tenant/activity boundaries | Backend | Every request | Authorization | Adopted security policy |

## 18. Detailed function specifications

The common source schema fields are defined in Sections 4 and 7. Adopted target behavior is normative in Sections 16 and 25–27 and in ADOPTED-DECISIONS.md; earlier source-gap/TBD notes are historical analysis, not open product decisions.

### PROB-F001 — List and view Problem records
**Status:** Source page/entity mapping + derived authorization. **Actors:** HOD, Manager, SUPER_ADMIN; Finance conditional; Student in academic/practice context.  
**Purpose:** return only authorized Problems and fields. **Preconditions:** authenticated/active actor; mapped Problems/Practice Problems/Course/Lab/Assessment/Contest route.  
**Inputs:** authenticated role/scope; optional Problem ID; activity context; query filters (contract TBD). Client role/scope are not trusted.  
**UI:** render authorized list/detail; actions shown only if separately authorized; Student response excludes hidden TestCase input/output.  
**Flow:** resolve actor → check role/action → apply College/Department/Section/group/activity scope → apply approved Problem visibility policy → query Problem → return safe fields.  
**Rules/validation:** PROB-V01–V05, V17, V21; public/scope policy remains PROB-D03–PROB-D05.  
**Backend/data:** read Problem and required associations; no mutation.  
**Success/failure/edge:** permitted record returned; foreign/invalid ID denied/not-found without disclosure; empty list state wording TBD; archived/draft behavior TBD.  
**Audit/privacy:** read logging Module 15; no hidden test data or unnecessary Student data.  
**Acceptance:** direct URL/API/list enforce same access; Student cannot fetch hidden TestCase values.  
**QA:** PROB-QA-001–010. **Decisions:** PROB-D01, PROB-D03–PROB-D05, PROB-D20–PROB-D23.

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
**QA:** PROB-QA-011–020. **Decisions:** PROB-D01, PROB-D02, PROB-D06–PROB-D12, PROB-D23–PROB-D25.

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
**QA:** PROB-QA-021–028. **Decisions:** PROB-D01, PROB-D06–PROB-D11, PROB-D17–PROB-D19, PROB-D22, PROB-D25.

### PROB-F004 — Set College, Department, and academic batch
**Status:** schema fields confirmed; requiredness/null/global semantics TBD. **Actors:** authorized editor. **Entry:** create/edit form.  
**Inputs:** collegeId, departmentId, academicBatch.  
**Flow:** select values → validate actor authority and parent relationship → save fields → access policy consumes values only after PROB-D04/PROB-D05/PROB-D09.  
**Rules:** Department must belong to College when both are set (derived); Manager cannot select foreign scope; batch is varchar, not an enum/multi-batch field in schema.  
**Validation/failure:** invalid parent/out-of-scope actor rejects; nullable/global behavior unresolved.  
**Data:** update organization/batch fields; downstream access impact uses F023.  
**Success/edge:** saved consistent hierarchy; changing Department after assignments/submissions is decision-gated.  
**Audit:** Module 15. **Acceptance:** no cross-tenant reparenting; batch is not treated as eligibility boundary absent approval.  
**QA:** PROB-QA-029–034. **Decisions:** PROB-D04, PROB-D05, PROB-D09, PROB-D23.

### PROB-F005 — Set scope and public visibility
**Actors:** SUPER_ADMIN/HOD; Manager may create/update SECTION_ONLY only within assigned Section.
**ContentScope:** SECTION_ONLY requires explicit ProblemSection links; DEPT_WIDE includes active Sections in owning Department; GLOBAL includes active Sections across Departments in owning College and is SUPER_ADMIN-only.
**isPublic:** true means authenticated Student discovery in Practice Problems, only while PUBLISHED and only within the Problem's College/Department/Section/academicBatch scope. It never grants anonymous access, cross-College access or bypasses activity permissions. False excludes the Problem from Practice discovery but does not prevent an authorized activity link from exposing it to eligible participants.
**Flow:** authorize scope change → validate hierarchy/Section links → preview access impact → save draft/new published version → invalidate relevant cache/index → audit.
**QA:** PROB-QA-035–043, 171–174, 199–200. **Decisions:** PROB-D03–PROB-D05, PROB-D21, PROB-D23, PROB-D25.

### PROB-F006 — Publish/unpublish/lifecycle transition
**Actors:** SUPER_ADMIN/HOD within scope. Manager and Student cannot publish/unpublish.
**Schema delta:** add Problem.status using shared PublishStatus values DRAFT/PUBLISHED/ARCHIVED; new Problems default DRAFT.
**Publish gate:** valid title/statement/difficulty/College/Department/scope; non-empty allowedLanguages; at least one valid TestCase; valid SECTION_ONLY audience association; valid resource limits. Publish creates immutable ProblemVersion + TestCase snapshot, sets PUBLISHED and queues Module 13 events.
**Unpublish:** PUBLISHED→DRAFT; keeps current and prior versions/history; removes new Student access while DRAFT. Republish creates a new immutable version.
**Failure:** invalid checklist or stale update rejects without partial state mutation. Module 17 owns canonical transition implementation.
**QA:** PROB-QA-044–049, 171–174, 195–198. **Decisions:** PROB-D02, PROB-D06, PROB-D10–PROB-D14, PROB-D17, PROB-D24–PROB-D25.

### PROB-F007 — Archive/restore Problem
**Actors:** SUPER_ADMIN platform-wide; HOD within assigned Department. Manager/Student cannot archive, restore or delete.
**Rules:** ARCHIVED is soft retirement. No hard-delete endpoint for Problem or TestCase. Archive prevents new Practice discovery, new activity attachment and new submissions, but retains activity references, versions, TestCases, CodeSubmissions, verdicts and audit history. Existing history remains available under activity/module policies.
**Restore:** ARCHIVED→DRAFT, not directly to PUBLISHED. A separate publish activates a version.
**TestCases:** remove/reorder/edit only in a DRAFT revision. Published version snapshots remain immutable.
**QA:** PROB-QA-050–056, 178, 199. **Decisions:** PROB-D01, PROB-D02, PROB-D17–PROB-D19, PROB-D25.

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
**QA:** PROB-QA-057–062. **Decisions:** PROB-D06, PROB-D07, PROB-D17, PROB-D18.

### PROB-F009 — Manage difficulty and tags
**Actors:** authorized Problem editor. **Inputs:** ProblemDifficulty enum, tags text[]. **Entry:** create/edit/list.  
**Flow:** select difficulty/tags → validate enum and approved tag policy → persist → list/search uses values.  
**Rules:** difficulty enum EASY/MEDIUM/HARD confirmed; tag catalogue/free-form/casing/duplicates/scope TBD. Difficulty does not imply score or TestCase count.  
**Validation:** invalid difficulty rejected; tag constraints TBD.  
**Data:** update difficulty/tags only.  
**Failure/edge:** duplicate/case variants, tag edits after publish, filter normalization.  
**Acceptance:** enum enforced; tags conform to approved policy. **QA:** PROB-QA-063–068. **Decisions:** PROB-D07–PROB-D09/PROB-D20.

### PROB-F010 — Configure languages and resource limits
**Actors:** authorized editors; Module 6/7 consumers. **Inputs:** allowedLanguages text[], timeLimitMs int, memoryLimitMb int.  
**Flow:** choose canonical language IDs/limits → validate shared catalogue/ranges → save → Module 6 maps identifiers to Judge0; Module 7 uses evaluation configuration.  
**Rules:** defaults, bounds, language identifiers, empty-list meaning, per-language limits, activity overrides are TBD. Do not rejudge old submissions automatically.  
**Validation:** only approved IDs/ranges accepted; empty list follows PROB-D10.  
**Data:** update fields only.  
**Failure/edge:** language removed after submissions, queued execution while limits change, Judge0 mapping unavailable.  
**Acceptance:** consistent with Module 6/7 policy; no unsupported interpretation. **QA:** PROB-QA-069–076. **Decisions:** PROB-D10, PROB-D11, PROB-D17, PROB-D18.

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
**QA:** PROB-QA-077–084. **Decisions:** PROB-D01, PROB-D12–PROB-D15, PROB-D25.

### PROB-F012 — Update TestCase
**Actors:** HOD/SUPER_ADMIN; Manager conditional. **Entry:** TestCase edit. **Inputs:** TestCase ID and input/output/hidden/order fields; parent reassignment not ordinary edit.  
**Flow:** load → authorize parent → edit → validate → update → return.  
**Rules:** content changes may invalidate historic results; no automatic rejudge or old submission rewrite. Schema has createdAt but no TestCase updatedAt.  
**Validation:** parent, size, boolean/order and concurrency policy.  
**Data:** update allowed fields only.  
**Failure/edge:** foreign Problem, active judge, active contest/assessment, hidden-to-visible change.  
**Acceptance:** scoped update and no implicit history mutation. **QA:** PROB-QA-085–090. **Decisions:** PROB-D12–PROB-D18.

### PROB-F013 — Delete TestCase
**Status:** decision-blocked; TestCase delete semantics not described. **Actors:** HOD/SUPER_ADMIN; Manager delete not granted.  
**Entry/input:** delete action and TestCase ID, confirmation TBD.  
**Flow:** authorize → inspect active/history references → apply approved delete/version policy → report outcome.  
**Rules:** no cascade/rejudge or hard delete inferred; Module 7 owns evaluation dependency.  
**Data:** deletion/retention blocked until PROB-D19.  
**Failure/edge:** active evaluation, historical submission, deleting final TestCase, repeated delete.  
**Acceptance:** no deletion before policy approval; Manager cannot delete by inference. **QA:** PROB-QA-091–095 blocked. **Decisions:** PROB-D01, PROB-D12, PROB-D18, PROB-D19.

### PROB-F014 — Reorder TestCases
**Actors:** authorized staff. **Entry:** TestCase list. **Inputs:** TestCase IDs/orderIndex values; 0/1 origin and uniqueness TBD.  
**UI:** reorder controls/drag behavior TBD. **Flow:** reorder → verify all IDs belong to same Problem → validate indices → save ordering → return deterministic list.  
**Rules:** orderIndex exists; uniqueness/contiguity and failedTestCaseIndex relationship are TBD.  
**Data:** update orderIndex for affected cases; atomicity policy TBD.  
**Failure/edge:** duplicate/negative indices, mixed Problem, partial update, concurrent reorder.  
**Acceptance:** valid ordering is deterministic; invalid reorder leaves no partial mutation. **QA:** PROB-QA-096–100. **Decisions:** PROB-D13/PROB-D18/PROB-D25.

### PROB-F015 — Set hidden/visible classification
**Actors:** authorized Problem/TestCase editors. **Input:** isHidden boolean. **Entry:** TestCase create/edit.  
**Flow:** set flag → authorize → validate → save → Student-facing APIs apply approved response policy.  
**Rules:** schema confirms boolean; whether visible means sample and whether visible cases are evaluated are Module 7 decisions. Hidden input/expectedOutput must not be sent in Student payloads, error details, or routine logs.  
**Data:** update isHidden only; no rejudge. **Failure/edge:** hidden→visible with submissions, active evaluation, direct API fetch.  
**Acceptance:** Student cannot fetch hidden values; staff access follows RBAC. **QA:** PROB-QA-101–108. **Decisions:** PROB-D14/PROB-D15/PROB-D17/PROB-D18.

### PROB-F016 — Student Practice Problems discovery
**Actor:** authenticated Student.
**Eligibility:** list only Problems that are PUBLISHED, isPublic=true, within owning College and configured ContentScope/ProblemSection/academicBatch, and available to the Student's current active membership. Public false excludes Practice discovery. Activity-assigned Problems with isPublic=false may still be available through their explicit eligible activity route.
**Security:** no anonymous catalog/detail access. No cross-College exposure. List and direct detail use identical policy; hidden TestCase values are excluded.
**QA:** PROB-QA-109–116, 171–174, 192–194, 200. **Decisions:** PROB-D03–PROB-D05, PROB-D09, PROB-D20–PROB-D21, PROB-D23.

### PROB-F017 — Access Problem through Course
**Actors:** Student/staff permitted by Module 4. **Entry:** My Courses/Course. **Owner:** Module 4 association, Module 5 Problem access.  
**Flow:** Module 4 authorizes Course → resolve CourseProblem → Module 5 checks Problem scope/visibility → return safe content → Module 6 handles submission.  
**Rules:** CourseProblem does not bypass Problem access; Problem view does not grant enrollment/submission; GroupCourseProblem is separate.  
**Inputs:** Course ID/Problem ID/actor; association and eligibility validation.  
**Failure/edge:** stale link, archived Problem, enrollment removed, scope mismatch, hidden testcase request.  
**Acceptance:** Course route and direct Problem route enforce equivalent authorization. **QA:** PROB-QA-117–122. **Decisions:** PROB-D04/PROB-D05/PROB-D16/PROB-D17; Module 4.

### PROB-F018 — Access Problem through Lab
**Actors:** Student/staff under Module 8. **Entry:** Lab assignment. **Inputs:** LabAssignment ID, Problem ID, actor.  
**Flow:** authorize Lab → validate assignment/membership/time → resolve Problem → apply Problem visibility → return statement → submission/scoring remain Modules 6/8.  
**Rules:** Lab assignment does not make Problem public; hidden testcase protection remains; edits/deletion use F023.  
**Failure/edge:** expired assignment, nonmember, archived Problem, invalid relation.  
**Acceptance:** both Lab and Problem permissions pass; no bypass via Problem ID. **QA:** PROB-QA-123–127. **Decisions:** PROB-D04/PROB-D05/PROB-D16–PROB-D18; Module 8.

### PROB-F019 — Access Problem through Assessment
**Actors:** Student/staff under Module 9. **Entry:** Assessment. **Inputs:** Assessment ID, Problem ID, actor.  
**Flow:** authorize Assessment/attempt → validate AssessmentProblem → apply Problem access → return content → Module 6/7 handle code/evaluation.  
**Rules:** Assessment assignment does not bypass Problem scope; assessment timing/attempt rules remain Module 9; snapshot policy unresolved.  
**Failure/edge:** not in window, no attempt permission, Problem edit during attempt, stale link.  
**Acceptance:** only eligible assessment actor receives Problem content. **QA:** PROB-QA-128–132. **Decisions:** PROB-D04/PROB-D05/PROB-D16–PROB-D18; Module 9.

### PROB-F020 — Access Problem through Contest
**Actors:** participants/staff under Module 11. **Entry:** Contest problem page. **Inputs:** Contest ID, Problem ID, actor.  
**Flow:** authorize Contest/participant → validate ContestProblem → apply Problem access/version policy → return statement → submission/scoring remains Modules 6/7/11.  
**Rules:** contest access does not make Problem globally public; contest assignment does not grant edit permission; snapshot/version policy unresolved.  
**Failure/edge:** nonparticipant, ended contest, foreign College, archived/changed Problem.  
**Acceptance:** contest and Problem authorization both pass; guessed Problem ID cannot bypass participation. **QA:** PROB-QA-133–137. **Decisions:** PROB-D04/PROB-D05/PROB-D16–PROB-D18; Module 11.

### PROB-F021 — Search/filter/sort/paginate
**Actors:** staff/Student within allowed page. **Inputs:** search term, difficulty, tags, language, batch, organization/activity, sort/page/size; exact list TBD.  
**Flow:** authenticate → scope filter first → validate query → search only authorized content → return safe results/page metadata.  
**Rules:** filters cannot widen scope; Student cannot use staff-only public/private or tenant filters; snippets must not disclose hidden/private statement content.  
**Validation:** unsupported filters/sort, invalid page/size, oversized query → reject/normalize per API contract TBD.  
**Data:** read only. **Acceptance:** all pages/search results retain scope; no foreign data leakage. **QA:** PROB-QA-138–145. **Decisions:** PROB-D20/PROB-D21/PROB-D23.

### PROB-F022 — View Problem-linked submission history
**Status:** Problem page maps CodeSubmission; RBAC Code Submission matrix: HOD View, Manager View*, Student Create/View own; Module 6 owns lifecycle.  
**Actors:** HOD/Manager/SUPER_ADMIN within scope; Student own submissions; Finance not mapped. **Entry:** Problem detail/history tab if approved.  
**Flow:** authorize Problem → separately authorize submission view → scope/ownership filter → return permitted fields.  
**Rules:** Problem view alone does not grant submission access; Student sees own records only; Manager Section scope; code/stdout/stderr visibility remains Module 6/7 policy.  
**Data:** read CodeSubmission only. **Failure/edge:** foreign Student, scope changes, private code/output.  
**Acceptance:** no submission/source-code/result disclosure outside permission. **QA:** PROB-QA-146–151. **Decisions:** PROB-D22/PROB-D25; Module 6.

### PROB-F023 — Preserve historical submission/evaluation meaning
**Policy:** immutable ProblemVersion snapshot for every published revision. The snapshot includes Problem statement, difficulty, tags, language whitelist, time/memory limits, and the ordered complete TestCase set with hidden flags.
**Submission contract:** Module 6 binds each accepted CodeSubmission to the ProblemVersion used at acceptance. Module 7 evaluates that immutable snapshot, not mutable draft/current pointer. Existing verdicts/results never silently change after edits or publication of a new version. Rejudge is an explicit Module 7 operation against a specified version with audit and downstream score reconciliation; Module 5 does not trigger it automatically.
**Schema delta:** ProblemVersion and snapshot data; CodeSubmission.problemVersionId or equivalent immutable reference.
**QA:** PROB-QA-152–158, 175–177, 198–199. **Decisions:** PROB-D17–PROB-D19.

### PROB-F024 — Audit and notification integration
**Actors:** system; Module 13 notification delivery, Module 15 audit catalog. **Triggers:** create/update/scope/public/lifecycle/TestCase mutation.  
**Flow:** approved mutation → emit audit event per Module 15 → notify only if Module 13 trigger/recipient is approved.  
**Rules:** no default notification assumption; avoid source code, full statement, hidden TestCase values, or sensitive Student data in routine audit payloads.  
**Inputs:** actor/action/target/scope/changed fields/outcome; schema owned by Module 15.  
**Failure:** audit/notification failure follows Modules 13/15/18. **Acceptance:** approved mutations are auditable; no unapproved notifications.  
**QA:** PROB-QA-159–163. **Decisions:** PROB-D24/PROB-D25.

### PROB-F025 — Enforce visibility across tenant/activity boundaries
**Actor:** backend authorization layer; applies on every read/write.
**Order:** authenticate → active identity/membership → role/action → College boundary → Department/Section/audience/batch → Problem status and isPublic where Practice discovery → activity eligibility → ownership → database operation.
**Rules:** Problem.collegeId is a hard tenant boundary. GLOBAL is College-wide only and SUPER_ADMIN controlled. HOD is Department-bounded. Manager is assigned-Section bounded. Student receives only eligible content and own submissions. isPublic never bypasses ContentScope or activity authorization.
**Failure:** deny before protected read/write; safe response without foreign record existence leakage; audit security-relevant denials per Module 15 policy.
**QA:** PROB-QA-164–170, 171–174, 194, 200. **Decisions:** PROB-D01–PROB-D05, PROB-D09, PROB-D14, PROB-D21, PROB-D23, PROB-D25.

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
| PROB-V01 | No action permission | Deny | PROB-D01 |
| PROB-V02 | Missing/invalid Problem ID | Safe not-found | API policy TBD |
| PROB-V03 | College/Department mismatch | Reject | Derived |
| PROB-V04 | Actor supplies foreign organization/Section | Deny | RBAC |
| PROB-V05 | title >255 chars | Reject | Schema |
| PROB-V06 | invalid difficulty | Reject | Schema |
| PROB-V07 | invalid scope | Reject | Schema |
| PROB-V08 | isPublic not boolean | Reject | Schema |
| PROB-V09 | invalid academicBatch | Apply approved format | PROB-D09 |
| PROB-V10 | invalid/duplicate tag | Apply approved tag policy | PROB-D08 |
| PROB-V11 | unsupported language | Reject after catalogue is approved | PROB-D10 |
| PROB-V12 | empty allowedLanguages | Apply approved empty-list semantics | PROB-D10 |
| PROB-V13 | resource limit outside range | Reject | PROB-D11 |
| PROB-V14 | TestCase missing/foreign Problem | Reject | Derived |
| PROB-V15 | input/output exceeds limit | Reject | PROB-D12 |
| PROB-V16 | invalid orderIndex | Reject/normalize per approved policy | PROB-D13 |
| PROB-V17 | Student requests hidden TestCase | Deny/omit hidden values | PROB-D14 |
| PROB-V18 | lifecycle unknown | Block pending decision | PROB-D02 |
| PROB-V19 | deletion with history/references | Apply retention decision; no cascade assumption | PROB-D19 |
| PROB-V20 | edit after submissions | Apply version/history policy | PROB-D17/PROB-D18 |
| PROB-V21 | search/direct URL widens scope | Filter/deny server-side | RBAC |
| PROB-V22 | Manager delete/publish | Deny unless explicitly approved | PROB-D01 |
| PROB-V23 | Finance Problem access | Follow resolved matrix | PROB-D01 |
| PROB-V24 | activity membership invalid | Deny | Modules 4/8/9/11 |
| PROB-V25 | audit/notification failure | Follow Modules 13/15/18 | PROB-D24/PROB-D25 |

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

**PROB-D01–PROB-D25 are adopted.** There are no open product decisions in this module. The full register is ADOPTED-DECISIONS.md.

| Workstream | Status | Remaining work |
|---|---|---|
| Product decisions | Adopted | None |
| Target schema | Proposed delta documented | Migration and backfill |
| RBAC | Target policy adopted | Canonical matrix/API updates |
| Runtime behavior | Specified | Implementation |
| Cross-module contracts | Defined | Integration with Modules 2, 4, 6–11, 13, 15, 17 and 18 |
| QA | 200 cases authored | Execute and attach evidence |

## 24. Review and readiness checklist

**Product policy — complete**
- [x] PROB-F001–PROB-F025 target behavior defined.
- [x] Role/action/College/Department/Section scope defined.
- [x] isPublic and ContentScope visibility rules adopted.
- [x] Problem lifecycle, archive and restore behavior defined.
- [x] Authoring, language, resource and TestCase behavior defined.
- [x] Hidden TestCase confidentiality and versioned evaluation defined.
- [x] Course/Lab/Assessment/Contest access and version binding defined.
- [x] Submission history, audit and notification contracts defined.

**Engineering and release — pending**
- [ ] Problem.status and ProblemSection migrations implemented.
- [ ] ProblemVersion/TestCase snapshot and CodeSubmission version binding implemented.
- [ ] Canonical RBAC and API/page matrices reconciled.
- [ ] Language catalogue/Judge0 mapping and resource ceilings verified.
- [ ] Activity version binding and Module 7 evaluation verified.
- [ ] Notification/audit/concurrency/failure integrations implemented.
- [ ] All 200 QA cases executed with evidence; security review and release sign-off complete.

**Review gate:** Module 5 product baseline adopted; implementation readiness depends on Section 27 engineering gates.

**Review gate:** Module 5 only. GitHub inclusion versions this detailed draft; it does not approve target behavior or authorize implementation of unresolved decisions.


## 25. Adopted target behavior — normative v1 baseline

This section supersedes earlier source-gap/TBD wording where the two differ. Existing schema fields remain as documented in Section 20; additions are in Section 26.

### 25.1 Roles and ownership
- SUPER_ADMIN: platform-wide Problem administration; may set GLOBAL within the owning College boundary.
- HOD: full Problem authoring/maintenance within assigned College + Department; may create SECTION_ONLY and DEPT_WIDE, not GLOBAL.
- MANAGER: view/create/update Problems and TestCases only for SECTION_ONLY within assigned Section. No publish, archive, restore, delete or reparent.
- STUDENT: view/solve Problems after Practice/activity access checks; own submission data only.
- FINANCE_TEAM: no Problem page/API access in v1 because Page/API matrices omit it despite platform-level View.
- Creator attribution and tenant/Section identity are server-derived; never trust client-supplied ownership fields.

### 25.2 Required fields and statement format
- title is required, trimmed, non-empty, 3–255 characters. Duplicate titles are allowed; no title uniqueness or slug is introduced.
- statement is required non-empty Markdown text. Render through an allowlisted/sanitized Markdown renderer; strip/reject executable HTML, scripts, event handlers, unsafe links and active content. No separate inputFormat/outputFormat/constraints/editorial/hint/solution columns are assumed in v1.
- difficulty is required and must be EASY, MEDIUM or HARD.
- tags are free-form strings, trimmed, normalized to lowercase and de-duplicated case-insensitively within the Problem; empty tags are rejected. No global tag catalogue is required in v1.
- academicBatch is optional single text identifier. If present, Student batch must match; null means no batch restriction within other access scope. Multi-batch targeting needs a future relation.
- College and owning Department are required; Department must belong to College. createdById and timestamps are server-controlled.

### 25.3 ContentScope, ProblemSection and isPublic
| ContentScope | Meaning | Eligible audience |
|---|---|---|
| SECTION_ONLY | Explicit Section audience | Active Students in linked ProblemSection rows only |
| DEPT_WIDE | Department-wide | Active Sections in owning Department |
| GLOBAL | College-wide across Departments | Active Sections in owning College; SUPER_ADMIN may set/change |

GLOBAL is never platform-public, anonymous or cross-College. SECTION_ONLY requires at least one valid ProblemSection link before publication. DEPT_WIDE/GLOBAL audience derives from the active Section hierarchy. Scope changes require HOD within Department or SUPER_ADMIN, impact preview and audit; HOD cannot widen to GLOBAL.

isPublic=true means **authenticated Student discovery in Practice Problems**, and only for PUBLISHED Problems whose College, ContentScope, ProblemSection (where relevant), academicBatch and membership rules match the Student. It does not grant anonymous internet access, cross-College sharing or an override of ContentScope. isPublic=false excludes the Problem from Practice discovery but an explicit Course/Lab/Assessment/Contest link may expose it to a participant who independently satisfies activity and Problem access. All routes use the same authorization policy.

### 25.4 Lifecycle and visibility
Problem uses DRAFT/PUBLISHED/ARCHIVED lifecycle (shared PublishStatus values). New Problems default to DRAFT.

| Status | Staff | Student | Rules |
|---|---|---|---|
| DRAFT | Authorized editors | Hidden | Editable; no Student access/submission |
| PUBLISHED | Authorized scoped staff | In-scope Practice or explicitly assigned activity participants | Immutable published version; eligible submissions allowed |
| ARCHIVED | Authorized staff | Existing history only | No discovery, new assignments or submissions |

Allowed transitions: create→DRAFT; DRAFT→PUBLISHED; PUBLISHED→DRAFT (unpublish); DRAFT/PUBLISHED→ARCHIVED; ARCHIVED→DRAFT (restore). Only SUPER_ADMIN/HOD within scope. Restore does not auto-publish. Publish requires complete metadata, at least one valid TestCase, at least one allowed language, valid resource limits and valid SECTION_ONLY links. Archive is soft retirement; no hard delete. Existing history remains available under owning-module access policy.

### 25.5 Language and resource-limit contract
- allowedLanguages contains one or more stable canonical DEZOLVER language keys; Judge0 IDs/versions are mapped server-side by Module 6.
- Empty allowedLanguages does not mean unrestricted. DRAFT may be incomplete; publication requires at least one supported key.
- Activity modules may narrow the allowed language set but cannot expand beyond Problem.allowedLanguages.
- Suggested defaults: timeLimitMs = 2000 and memoryLimitMb = 256. Adopt configurable validation bounds of 100–60,000 ms and 8–512 MB, subject to stricter deployed Judge0/worker ceilings. Legacy values are reference only, not proof of target defaults.
- Limits are captured in each published ProblemVersion. Edits do not rewrite queued or historical submissions.

### 25.6 TestCase authoring, ordering and visibility
- input and expectedOutput are required text values; empty string is valid for intentional empty-input/empty-output cases.
- isHidden=false is the visible/sample classification. isHidden=true is hidden evaluation material. Student payloads include visible sample input/output only; omit hidden inputs, expected outputs, identifiers, exact count and order metadata. Full hidden values are available only to authorized authoring staff and server-side evaluation.
- Both visible and hidden cases are included in formal evaluation. Module 7 owns verdict aggregation, scoring, comparison and any score weights. Target TestCase has no weight field; no legacy weight is imported.
- orderIndex is zero-based, non-negative, unique and contiguous within a ProblemVersion (0..N-1). Reorders are atomic. Stable TestCase IDs plus ProblemVersion are canonical; failed-case index is derived from pinned version order.
- TestCase create/edit/remove/reorder and bulk import operate only on DRAFT revisions. Bulk import is transactional; partial success is not silently reported as complete. A Problem requires at least one TestCase to publish.
- Hidden-test content must not appear in Student API payloads, browser state, logs, errors, analytics or notification content. Student responses do not expose exact hidden TestCase count.

### 25.7 Reuse and activity access
Problem IDs may be reused in same-College Courses, Labs, Assessments and Contests when each owning module authorizes the relationship and Problem audience permits it. No default cross-College sharing. Activity assignment does not change isPublic or ContentScope. Activity routes require both activity eligibility and Problem status/scope checks. GroupCourseProblem remains Module 10-owned; group-only association is not a CourseProblem.

### 25.8 Versioning and historical result integrity
- Every publish produces immutable ProblemVersion containing Problem statement/metadata/evaluation configuration and the complete ordered TestCase set, including hidden flags.
- CodeSubmission is bound to the exact ProblemVersion at submission time. Module 7 evaluates that pinned version even when a new draft/current pointer exists.
- Published version content is never edited in place. Change requires a new draft revision and publish transition. Active Course/Lab/Assessment/Contest references keep the version they selected; they change only through an explicit action by the owning module.
- Old verdicts/scores are not recalculated automatically. Rejudge is a separate Module 7 operation, bound to an explicit version and audited. No implicit rejudge on Problem/TestCase edits.
- Draft TestCases may be removed/reordered; published version snapshots remain preserved.

### 25.9 Search and submission visibility
- Search is scope-first. Staff may search title/statement and filter difficulty, tags, language, status, College/Department/Section, academicBatch and isPublic.
- Students search only PUBLISHED Practice-eligible Problems or Problems in a currently authorized activity. Never expose hidden test data.
- Default sort createdAt descending then stable Problem ID; page size 20 default, 100 max. Invalid filters/sort/page values are rejected.
- Student sees own submissions only. HOD sees authorized Department records; Manager sees assigned Section; SUPER_ADMIN platform-wide. FINANCE_TEAM has no Problem submission access. Source code, raw stdout/stderr, compiler diagnostics and raw Judge0 artifacts remain Module 6 field-level permissions.

### 25.10 Notifications and audit
- Module 13 owns notification delivery. Emit events on publish/new version, unpublish/archive/restore and availability changes; Module 13 defines recipient resolution, channel, content and retry.
- Module 15 owns audit. Record actor/service, action, Problem/TestCase/version IDs, College/Department/Section scope, changed field names, event/idempotency key, timestamp and reason where relevant.
- Do not copy hidden input/output, source code, full statement or sensitive Student data into routine audit or notification payloads.

## 26. Target schema delta — proposed implementation changes

These fields/relations are not present in the reconstructed schema as fetched. They require migration/review; the specification does not claim they already exist.

### 26.1 Problem
- Add Problem.status using shared PublishStatus, NOT NULL DEFAULT DRAFT.
- Add publishedAt timestamp and currentPublishedVersionId reference.
- Keep College and owning Department required, including GLOBAL records.
- Keep existing isPublic, scope, content and resource fields.

### 26.2 ProblemSection
Add ProblemSection(problemId, sectionId, createdAt), with unique(problemId, sectionId). Required for SECTION_ONLY audiences; validate Section belongs to owning College and Department. Global/Department-wide scope is derived from hierarchy and need not materialize one link per Section in v1.

### 26.3 ProblemVersion
Add immutable version entity or equivalent immutable snapshot:
- id, problemId, versionNumber, publishedAt, title, statement, difficulty, tags, academicBatch, scope, isPublic, allowedLanguages, timeLimitMs, memoryLimitMb, createdAt, createdById.
- Version-owned TestCase snapshot rows with stable testCaseId, input, expectedOutput, isHidden, orderIndex.
- Unique(problemId, versionNumber); unique(problemVersionId, orderIndex); immutable after publish.
- Problem.currentPublishedVersionId points to latest published version; older versions remain addressable by existing submissions/activity bindings.

### 26.4 CodeSubmission and activity binding
- Add CodeSubmission.problemVersionId or equivalent FK to immutable ProblemVersion.
- Modules 4/8/9/10/11 bind their activity Problem relation to selected published ProblemVersion. Latest pointer must not mutate active attempt content.
- Module 7 consumes immutable snapshot and correlates results by TestCase ID/version.

### 26.5 Language and limit configuration
A platform language catalogue/identifier map is required in Module 6. Deployment-specific maximum time/memory ceilings are configuration, not free-form client values. Activity modules may narrow but not expand Problem configuration.

## 27. Engineering implementation gates (not product decisions)
1. Add Problem.status/publishedAt/currentPublishedVersionId and migrate existing records to a safe initial status; review per-row, do not assume all existing Problems are published.
2. Add ProblemSection and scope resolver for SECTION_ONLY, DEPT_WIDE and College-bounded GLOBAL; integrate Module 2 active Section/membership authority.
3. Add immutable ProblemVersion/TestCase snapshot persistence and CodeSubmission version reference; settle version FK placement in activity association models.
4. Update canonical RBAC/page/API matrices for Finance denial, Manager SECTION_ONLY Create/Update only, HOD Department scope and SUPER_ADMIN GLOBAL authority.
5. Align canonical language keys/Judge0 mapping with Module 6; confirm deployment ceilings and TestCase size/count limits.
6. Implement safe Markdown rendering and Student response allowlists; verify hidden fields cannot leak through direct ID/search/errors/logs/client state.
7. Integrate Module 7 evaluation against pinned versions/stable TestCase IDs/order; no automatic rejudge.
8. Integrate activity version selection with Modules 4/8/9/10/11; preserve history on archive and relationship removal.
9. Integrate Module 13 notifications, Module 15 audit, Module 17 transitions and Module 18 concurrency/retry.
10. Execute 200 QA cases, security review, migration backfill validation and release sign-off.

**Legacy boundary:** legacy Module 05 documentation describes a platform-wide catalog, separate content schema, slug/Markdown-specific columns, isSample and weight TestCase fields, old resource defaults/ranges, publish requiring a TestCase, and no update/delete endpoints. These are historical observations only. The target schema uses College/Department ownership, isPublic, ContentScope, isHidden and orderIndex; no legacy field/API contract is silently imported.
