# Module 5 — Problem Management: Adopted Decisions

**Document ID:** DEZ-PROB-DECISIONS-005  
**Version:** 1.0  
**Status:** Adopted target product baseline  
**Decision range:** PROB-D01–PROB-D25

Adopted by explicit user direction on 2026-11-06: isPublic option D (authenticated Practice Problems discovery within configured Problem scope and owning College), with recommended defaults applied to remaining decisions. This register reconciles original/v2 Business Flow and Page–DB Mapping, RBAC, target schema, Module 4 adopted decisions, Module 6 submission contract, Module 7 evaluation requirements and reviewed legacy Module 05 documentation. README Sections 25–27 are normative.

| ID | Topic | Adopted decision |
|---|---|---|
| PROB-D01 | Role and action permissions | SUPER_ADMIN has platform-wide full authoring; HOD has full authoring in assigned College+Department; Manager Create/Update only for SECTION_ONLY Problems/TestCases in assigned Section; Students View/Solve only; Finance has no Problem page/API access in v1; Manager cannot publish/archive/restore/delete. |
| PROB-D02 | Problem lifecycle field | Add Problem.status using DRAFT/PUBLISHED/ARCHIVED (shared PublishStatus values), default DRAFT; publish requires complete fields, allowed language, valid limits, valid audience and at least one TestCase; only HOD/SUPER_ADMIN publish/unpublish/archive/restore. |
| PROB-D03 | isPublic meaning | isPublic=true means authenticated Student Practice Problems discovery only within the Problem's existing College, ContentScope, Section and academicBatch eligibility; no anonymous Internet or cross-College access; false excludes Practice discovery but does not block valid explicit activity assignment. |
| PROB-D04 | Visibility intersection | Problem access is the intersection of active identity/membership, College, Department, Section, academicBatch, Problem status, scope, isPublic for Practice discovery and owning activity eligibility; no flag/link bypasses another rule. |
| PROB-D05 | ContentScope semantics | SECTION_ONLY requires explicit ProblemSection rows; DEPT_WIDE includes active Sections in owning Department; GLOBAL includes active Sections across Departments in owning College only; SUPER_ADMIN alone can set/change GLOBAL; HOD remains Department-bound. |
| PROB-D06 | Required fields and statement format | title (3–255), statement (non-empty Markdown), difficulty, collegeId, departmentId and scope are required; statement rendered through safe allowlisted Markdown; no structured input/output/constraints/editorial/hint/solution fields added in v1. |
| PROB-D07 | Title validation/uniqueness | Problem title duplicates are allowed; title is trimmed and length-validated only. No unique title or slug requirement is introduced. |
| PROB-D08 | Tag model and normalization | Use free-form tags; trim, normalize lowercase, case-insensitive dedupe within Problem; reject empty entries; no global controlled tag catalogue required. |
| PROB-D09 | Academic batch behavior | academicBatch is optional single text identifier; if present, Student batch must match; null means no batch restriction beyond ContentScope; multiple batch targeting needs future relation. |
| PROB-D10 | Language identifiers and empty list | allowedLanguages uses canonical DEZOLVER lowercase keys; at least one supported key required to publish; empty does not mean all languages; Module 6 maps keys to Judge0 IDs/version server-side; activities may narrow but never broaden. |
| PROB-D11 | Time/memory defaults and bounds | Suggested defaults: 2000ms and 256MB. Adopt configurable ranges 100–60000ms and 8–512MB subject to stricter deployment ceilings; values are versioned and not retroactive. |
| PROB-D12 | TestCase fields, limits and import | TestCase input and expectedOutput are required text but empty string is valid; configurable payload/count caps; transactional bulk import (no silent partial commit); at least one valid TestCase required for publication. |
| PROB-D13 | TestCase order and reorder | orderIndex is zero-based, unique and contiguous per ProblemVersion; reordering is atomic; evaluation correlates by stable TestCase ID and immutable version. |
| PROB-D14 | Hidden TestCase confidentiality | isHidden=true data is server-only; no hidden input/output, IDs, exact count or order to Students through UI/API/search/errors/logs/analytics; authorized scoped staff can inspect authoring payload. |
| PROB-D15 | Visible/hidden evaluation behavior | isHidden=false is visible/sample; both visible and hidden cases enter formal evaluation; Student receives visible samples only; Module 7 owns verdict, scoring and aggregation; no TestCase weight field added in Module 5. |
| PROB-D16 | Problem reuse across activities | Same-College Problems may be reused in Courses, Labs, Assessments and Contests when both activity and Problem policies allow; GroupCourseProblem remains Module 10 owned; reuse does not modify isPublic/scope. |
| PROB-D17 | Published content versioning | Every publish creates immutable ProblemVersion with Problem metadata, language/limits and complete ordered TestCase snapshot; published content changes only through a new draft revision and republish. |
| PROB-D18 | Historical submissions and rejudge | CodeSubmission binds to the exact ProblemVersion at acceptance; old results/verdicts remain pinned; no automatic rejudge; explicit rejudge is Module 7-owned and audited. |
| PROB-D19 | Archive/delete/retention | No hard-delete endpoint for Problem or TestCase; archive retires the Problem; TestCase changes/removal only in draft revisions; published versions and all activity/submission history remain. |
| PROB-D20 | Search/filter/sort/pagination | Search by title/statement; filters difficulty/tags/language; staff additionally status/scope/Department/Section/batch/isPublic; scope filter applied before search; default createdAt desc then stable ID; page 20 default/max100; invalid query rejected. |
| PROB-D21 | Practice Problems inclusion | Practice Problems includes only PUBLISHED and isPublic=true Problems for authenticated active Students matching College/scope/Section/batch; activity-only Problems do not appear there unless isPublic=true. |
| PROB-D22 | Submission history permissions | Student sees own submissions; HOD sees assigned Department; Manager assigned Section; SUPER_ADMIN global; Finance denied. Source code/stdout/stderr/raw Judge0 artifacts follow Module 6 field-level permissions. |
| PROB-D23 | Cross-College/Department sharing | No cross-College Problem sharing. GLOBAL is College-wide only; Problem collegeId and owning departmentId remain required; Department-wide access does not cross Departments unless GLOBAL is set by SUPER_ADMIN. |
| PROB-D24 | Notifications | Module 13 owns delivery; emit events on publish/new version, unpublish/archive/restore and availability changes; Module 13 resolves recipients/channels/templates/retry. |
| PROB-D25 | Audit events and metadata | Module 15 records Problem/TestCase create/update/publish/unpublish/archive/restore/scope/isPublic/reorder/version events with actor, target/version, tenant scope, changed fields, event key, timestamp and reason; never log hidden contents. |

## Source reconciliation and legacy boundary

- Business Flow original/v2 confirms HOD Department Problems, Manager assigned-Section activity scope, and Student Problems within Course/Lab/Contest flows. It does not define lifecycle, public visibility or versioning.
- Page–DB Mapping original/v2 maps Problem/TestCase/CodeSubmission to HOD and Manager Problems pages and Student Practice Problems; this confirms page/entity relationships, not all permissions.
- RBAC grants SUPER_ADMIN/HOD full CRUD, Manager V/C/U* and Student View/Solve. Manager Delete is not explicitly granted. Finance platform View conflicts with omission from page/API matrices; v1 resolves to no Finance Problem access.
- Target schema has College/Department-owned Problem fields, isPublic, ContentScope, TestCase.isHidden and orderIndex. It does not include status, ProblemSection, version snapshots, TestCase weights or isSample.
- Legacy Module 05 docs/code audit describe a platform-wide content catalog, separate content schema, slug/Markdown columns, isSample/weight TestCases, legacy resource defaults/ranges, publish requiring a TestCase, and no update/delete endpoints. These are historical observations only; target schema and adopted decisions take precedence.

## Required engineering gates

1. Add Problem lifecycle status and Section audience relation.
2. Add immutable ProblemVersion/TestCase snapshots and bind CodeSubmission to version.
3. Reconcile canonical RBAC/page/API policies.
4. Implement safe Markdown rendering and Student response DTO allowlists.
5. Align canonical language keys and runtime ceilings with Module 6.
6. Integrate Module 7 evaluation with immutable version and stable TestCase identity/order.
7. Bind activity versions in Modules 4/8/9/10/11.
8. Integrate Module 13 notification events, Module 15 audit, Module 17 transitions and Module 18 concurrency/failure contracts.
9. Execute all 200 QA cases, migration validation, security review and release sign-off.

**Approval boundary:** product decisions are adopted; schema migration, RBAC changes, runtime implementation, integrations and QA execution are not represented as complete or tested.