# Module 4 — Course & Enrollment: Adopted Decisions

**Document ID:** DEZ-COURSE-DECISIONS-004  
**Version:** 1.0  
**Status:** Adopted target product baseline  
**Decision range:** COURSE-D01–COURSE-D40

Adopted by explicit user direction on 2026-11-06: use Option A and recommended Module 4 defaults. This register reconciles original/v2 Business Flow and Page–DB Mapping, RBAC, target schema, and reviewed legacy Module 04 documentation. README.md Sections 24–26 are normative.

| ID | Topic | Adopted decision |
|---|---|---|
| COURSE-D01 | Role/create policy | SUPER_ADMIN/HOD/scoped Manager may create; Manager only SECTION_ONLY in assigned Section. |
| COURSE-D02 | HOD actions | HOD has full Course admin in assigned College + Department; no hard delete. |
| COURSE-D03 | Manager actions | Manager can create/update own SECTION_ONLY Course and own Section links; no scope widening/publish/archive/restore/completion. |
| COURSE-D04 | Finance/SUPER_ADMIN | SUPER_ADMIN platform-wide; FINANCE_TEAM metadata-only View in assigned College; no academic detail or mapped-page requirement. |
| COURSE-D05 | Field validation | title required/non-empty/max255; code required/trimmed/uppercase/max50; description optional. |
| COURSE-D06 | Code uniqueness | code unique case-insensitively within College; archived code remains reserved. |
| COURSE-D07 | ContentScope semantics | SECTION_ONLY explicit Sections; DEPT_WIDE all active Sections in owning Department; GLOBAL all active Sections in owning College; only SUPER_ADMIN may create/change GLOBAL and HOD remains Department-bounded. |
| COURSE-D08 | College boundary | GLOBAL never public or cross-College; Course retains owning College and Department. |
| COURSE-D09 | Audience materialization | SECTION_ONLY needs CourseSection before publish; DEPT_WIDE/GLOBAL materialize present and future active Sections. |
| COURSE-D10 | Assignment authority | SUPER_ADMIN/HOD assign/remove in scope; Manager own Section for SECTION_ONLY only. |
| COURSE-D11 | Option A automatic enrollment | OPTION A: published Course assignment auto-creates enrollment for eligible active Students; DRAFT waits for publish. |
| COURSE-D12 | Enrollment creation methods | Ordinary v1 enrollment is automation-only; no manual HOD/Manager path and no Student self-enrollment. |
| COURSE-D13 | Eligibility | Eligibility: active Student membership, active included Section, same College, PUBLISHED Course, College ACTIVE, Module 3 entitlement positive. |
| COURSE-D14 | Duplicate/idempotency | One row per (courseId, studentId); duplicate events/retries idempotent; ENROLLED/COMPLETED not reset. |
| COURSE-D15 | Withdrawal/repeat | Student cannot self-withdraw; HOD/SUPER_ADMIN and scoped Manager may staff-withdraw with reason; preserve history. |
| COURSE-D16 | Enrollment schema delta | Add CourseEnrollment status ENROLLED/COMPLETED/WITHDRAWN, withdrawnAt, withdrawalReason, enrollmentSource, createdById, unique/index constraints. |
| COURSE-D17 | Progress display | Display enrollment status/timestamps only; no percentage/inferred learning progress. |
| COURSE-D18 | Repeat/correction | Repeat disabled; COMPLETED not reset by roster sync; correction requires SUPER_ADMIN, reason and audit. |
| COURSE-D19 | CourseProblem semantics | CourseProblem unordered; no required/optional, weight, schedule, due date, or completion contribution. |
| COURSE-D20 | History retention | Removing audience/Problem links never deletes enrollment, Problem, submission/result, Lab, Assessment or audit history. |
| COURSE-D21 | My Courses visibility | Student My Courses enrolled-only; no public/discoverable catalogue in v1. |
| COURSE-D22 | Status visibility | DRAFT staff-only; PUBLISHED eligible enrolled Student access; ARCHIVED read-only history/no new enrollment/activity. |
| COURSE-D23 | Archive/restore | Hard delete not exposed; archive retires; restore returns DRAFT; no cascade. |
| COURSE-D24 | Search/pagination | Search title/code; filter status/scope/Department/Section; stable createdAt-desc sort; page 20 default/max100. |
| COURSE-D25 | Subscription/seat | Module 3 alone owns entitlement/seats; active Student membership consumes seat; CourseEnrollment never alters usedSeats/purchasedSeats. |
| COURSE-D26 | Notifications | Module 13 owns delivery; emit Course lifecycle, audience, enrollment, withdrawal and completion events. |
| COURSE-D27 | Audit | Module 15 owns audit; record actor/service, target IDs, scope, action, source event, timestamp and relevant reason. |
| COURSE-D28 | Status transitions | Transitions: create→DRAFT; DRAFT→PUBLISHED; PUBLISHED→DRAFT; DRAFT/PUBLISHED→ARCHIVED; ARCHIVED→DRAFT. HOD/SUPER_ADMIN only. |
| COURSE-D29 | Enrollment trigger matrix | Reconcile on published assignment, Course publish, Student join/activation, and Section activation in DEPT_WIDE/GLOBAL; outbox/retry/idempotency. |
| COURSE-D30 | Enrollment privacy | Enrollment read: SUPER_ADMIN global; HOD Department; Manager Section; Student own; Finance none. Creation system-owned. |
| COURSE-D31 | Scope cross-check | Scope meanings are those in D07; GLOBAL is College-wide only and SUPER_ADMIN controlled. |
| COURSE-D32 | Scope change impact | Scope/org/Section changes require HOD/SUPER_ADMIN, impact preview and audit; preserve enrollments and make out-of-scope history-only. |
| COURSE-D33 | Archive/retention | Archive soft retirement; restore to DRAFT; no hard delete/cascade. |
| COURSE-D34 | Problem association | Problem attach requires same-College Problem and Module 5 policy; unordered/unlink-only; Manager own SECTION_ONLY scope. |
| COURSE-D35 | Student RBAC conflict | Student RBAC View/Enroll conflict resolves to View-only based on Page/API matrices; self-enrollment disabled. |
| COURSE-D36 | Enrollment data privacy | Enrollment detail scope follows D30; minimum fields only; Finance receives no Student academic data. |
| COURSE-D37 | Progress representation | No percentage or automatic progress inference; status/timestamps only. |
| COURSE-D38 | Completion | HOD manually verifies completion; SUPER_ADMIN corrects with reason/audit; set COMPLETED + server completedAt; no automatic activity formula. |
| COURSE-D39 | GroupCourseProblem | Module 10 owns GroupCourseProblem writes/Group permission; Module 4 reads after authorization; Group-only links do not enroll. |
| COURSE-D40 | Concurrency/failure | Use transaction/outbox, unique Student-Course key, worker revalidation, optimistic concurrency and no false partial success. |

## Source reconciliation and legacy boundary
- Business Flow original/v2 supports HOD Course ownership, Manager assigned-Section scope and Student Course/Problem activity; it does not define automatic enrollment.
- Page–DB Mapping original/v2 maps Course/CourseSection/CourseEnrollment to HOD/Manager Course areas and Student My Courses; it does not define all lifecycle behavior.
- RBAC has Student View/Enroll vs View-only and Manager Create/Update* ambiguity; the least-privilege adopted matrix resolves both.
- Target schema contains Course, CourseSection, CourseEnrollment, CourseProblem, GroupCourseProblem, ContentScope and PublishStatus. Proposed lifecycle fields are a schema delta.
- Legacy Module 04 documentation describes CourseRoom/ordered Rooms, Syllabus/overlays, Learning Paths/PathProgress and no CourseEnrollment. These are historical only and are not imported into target Module 4.

## Engineering gates
1. Migrate CourseEnrollment lifecycle fields and unique/index constraints.
2. Update canonical RBAC/page/API matrices.
3. Implement Section audience materialization and Module 2 Section/roster events.
4. Implement Option A worker with outbox/retry/idempotency and Module 3 entitlement.
5. Implement lifecycle/history-only behavior and Module 5–10 permissions.
6. Integrate Module 13 notifications, Module 15 audit, Module 17 transitions and Module 18 failure handling.
7. Execute 180 QA cases, migration validation and security review before release.

**Approval boundary:** product decisions are adopted; schema migration, RBAC, event delivery, notification, audit, runtime behavior and QA execution are not represented as already implemented or tested.