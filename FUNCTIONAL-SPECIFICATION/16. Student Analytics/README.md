# DEZOLVER — Module 16: Student Analytics

**Document ID:** STUDENT-ANALYTICS-FS-001  
**Version:** 0.1  
**Status:** Detailed draft — legacy findings captured; analytics target requirements and policy decisions remain open  
**Governing standard:** [Functional Specification Standard v1.0](../FUNCTIONAL-SPECIFICATION-STANDARD.md)  
**Source reconstruction:** [Earlier Module 16 forensic specification](../../functional-spec-from-old%20code/module-16-analytics/SPECIFICATION.md)

> **Evidence caveat:** This draft uses the previous Module 16 source-forensic reconstruction as legacy evidence. It is not a fresh source-code checkout/reverification. Existing behavior, especially defects and reported code locations, must be validated against the current application branch before implementation planning. Legacy behavior is not target behavior.

## 1. Purpose and mission

Student Analytics provides visibility into learner progress, learning-path completion, academic performance, cohort progression, dashboard indicators, and competitive standing.

The current legacy forensic report describes an incomplete/asymmetric implementation:
- Personal learning-path progress is persisted in the `paths` schema.
- Room progress is reported as raw SQL access to `paths.room_progress`, not mapped as a Prisma model.
- A progress aggregation consumer recomputes path progress.
- Dashboard KPIs and several analytics pages are reported to use mocked or client-side calculations rather than real backend analytics APIs.
- Global leaderboard and individual standing endpoints are reported defective.
- A skip-to-certificate vulnerability is reported because skipped rooms are counted as completed.

**Core distinction:** this module documents observed legacy capabilities/limitations and candidate target behavior. It does not approve new analytics endpoints, dashboard metrics, or grading rules.

## 2. Scope

### In scope
- Personal learning-path progress list and detail.
- Next-step resolution.
- Room start/skip/complete progress mutation.
- Asynchronous path progress aggregation and completion.
- Student dashboard KPI/activity analytics (legacy mock behavior and target decisions).
- Assessment/event/platform analytics surfaces.
- Faculty/cohort progress analytics.
- Global leaderboard and individual standing.
- Platform rating/ELO calculation references.
- Authorization, tenant isolation, data integrity, fraud prevention, privacy, and performance.
- Analytics export, retention, metrics, and learning analytics as open capability decisions.

### Out of scope
- Course/path/room content authoring (Modules 4/5).
- Code submission execution and grading mechanics (Modules 6/7); analytics consumes outcomes only.
- Contest participation/scoring lifecycle (Module 11); analytics consumes published/approved ranking inputs.
- Certificate issuance mechanics (Module 12); Module 16 must not independently grant certificates.
- Notification delivery (Module 13).
- Audit event persistence (Module 15), except analytics access/export audit requirements.
- Canonical lifecycle rules (Module 17) and general failure/retry behavior (Module 18), which remain cross-references.

## 3. Sources and evidence

| Source | Evidence used | Limitations |
|---|---|---|
| [Prior Module 16 forensic reconstruction](../../functional-spec-from-old%20code/module-16-analytics/SPECIFICATION.md) | Reported controllers, endpoints, path progress, ELO, UI mock behavior, defects, and traceability | Secondary forensic artifact; revalidate code paths before implementation sign-off |
| [Business Flow v2](../../docs/source-text/Business%20Flow%20v2.md) | Student learning path/course/problem/code submission → backend → Judge0 → evaluation → result → score | High-level workflow only |
| [Page–DB Mapping v1](../../docs/source-text/Page%20-%20DB%20Mapping.md) | Role pages and entity associations | Analytics pages/metrics are not fully defined |
| [Page–DB Mapping v2](../../docs/source-text/Page%20-%20DB%20Mapping%20v2.md) | Current mapped page/entity references | Does not define analytics calculations |
| [RBAC](../../docs/source-text/RBAC.md) | Roles, tenant/department/section/group scope, ownership, server-side checks | Analytics-specific permissions are not fully specified |
| [Schema](../../docs/source-text/scheema.md) | Conceptual entities including Course, CourseEnrollment, CodeSubmission, Assessment, Contest, Certificate, User | Does not define a dedicated Analytics schema |
| [Functional Specification Standard](../FUNCTIONAL-SPECIFICATION-STANDARD.md) | Required function structure, evidence labels, acceptance criteria, QA and decision gates | Authoring standard |

### Evidence labels
- **Source-confirmed:** explicit in supplied source documentation.
- **Legacy-verified (prior forensic record):** reported by previous Module 16 forensic analysis; not freshly rechecked here.
- **Derived:** logically inferred from sources.
- **Recommendation:** suggested target behavior, not approved.
- **TBD / Decision Required:** product, academic, security, or architecture decision is unresolved.

## 4. Terminology

| Term | Definition |
|---|---|
| Learning path | Ordered learning steps/rooms associated with a learner. |
| Path progress | Persisted aggregate progress for a user/path pair. |
| Room progress | Per-user state for a room; legacy report describes `in_progress`, `skipped`, `completed`. |
| Progress aggregator | Asynchronous consumer recomputing aggregate path progress from room states. |
| Analytics metric | Aggregated or derived information about learning, performance, participation, or platform activity. |
| Cohort | A group of students scoped by college/department/section/group according to approved academic relationships. |
| Platform rating | Competitive rating reported as ELO-like value in the legacy reconstruction. |
| Leaderboard | Ranked list of users/persons based on a defined rating or score. |
| Skip | Learner action that bypasses a room; whether it satisfies academic completion is a critical unresolved policy. |

## 5. Roles and scope

| Role | Source RBAC scope | Analytics access status |
|---|---|---|
| SUPER_ADMIN | Global | Dashboard/platform analytics access is implied by role; exact permission set TBD |
| FINANCE_TEAM | Assigned College | Analytics access to student academic data not explicitly established; default deny until approved |
| HOD | College + Department | Cohort/department analytics are a candidate capability; permission details TBD |
| MANAGER | College + Department + Section/Group | Section/cohort progress is a candidate capability; permission details TBD |
| STUDENT | Self + permitted academic data | Personal path progress and personal standing are reported legacy capabilities |
| Faculty/instructor identity | Not independently defined in role list | Mapping to HOD/MANAGER or separate role requires decision |

**Security rule:** backend must enforce role, tenant, scope, and ownership. UI page visibility must not be the sole control. Analytics responses must not reveal student records outside the viewer’s authorized scope.

## 6. Functional inventory

| ID | Function | Evidence / status |
|---|---|---|
| ANA-F001 | View personal learning paths and progress | Legacy-verified (prior forensic record) |
| ANA-F002 | View path progress detail | Legacy-verified |
| ANA-F003 | Resolve next learning step | Legacy-verified |
| ANA-F004 | Start, skip, or complete a room | Legacy-verified; progress and certificate semantics need correction/approval |
| ANA-F005 | Aggregate room progress into path progress | Legacy-verified; skip policy defect reported |
| ANA-F006 | Display student dashboard KPIs and activity | Legacy report says mocked; real metrics/backend TBD |
| ANA-F007 | View assessment analytics | Legacy report says frontend-derived/client-side; target aggregates TBD |
| ANA-F008 | View event/contest analytics | Legacy report says frontend-derived; target semantics TBD |
| ANA-F009 | View platform analytics/growth | Legacy report says frontend-derived from raw lists; target API TBD |
| ANA-F010 | View faculty/cohort progress analytics | Legacy report says backend capability absent/skeleton |
| ANA-F011 | View global leaderboard | Legacy endpoint reported defective |
| ANA-F012 | View individual competitive standing | Legacy endpoint reported defective |
| ANA-F013 | Calculate/update platform rating | Legacy ELO service/consumer reported implemented; algorithm policy TBD |
| ANA-F014 | Analytics export, privacy, and retention | Not source-defined; decision required |

## 7. Legacy architecture summary

The prior forensic report describes the following split:
- **Personal progress:** `MePathsController` reads `PathProgress`; room state uses raw SQL against `paths.room_progress`.
- **Aggregation:** `RoomProgressUpdated` outbox event triggers `ProgressAggregatorConsumer`; it calculates completed steps and upserts path progress.
- **Competitive rating:** `PlatformRatingService` processes `EventCompleted` and updates person rating and Redis leaderboard data.
- **Dashboard:** mock values in browser memory with artificial delay.
- **Analytics pages:** client-side reductions over fetched entity lists.
- **Faculty analytics:** no backend cohort analytics service reported.
- **Leaderboard:** repository query reportedly targets a non-existent Prisma model, returning empty data; individual standing reportedly returns 404.

## 8. Critical legacy defects / discrepancies

| ID | Finding from prior forensic record | Impact | Status |
|---|---|---|---|
| ANA-DEF-01 | Dedicated analytics service/controller/schema reportedly absent | Dashboard and faculty analytics lack authoritative server aggregation | Revalidate source |
| ANA-DEF-02 | Global leaderboard query reportedly references non-existent Prisma model | Leaderboard permanently empty | Revalidate source |
| ANA-DEF-03 | Individual standing query reportedly references non-existent model | Personal standing returns 404 | Revalidate source |
| ANA-DEF-04 | `paths.room_progress` reportedly lacks Prisma mapping and uses raw SQL | Type safety, migration and integrity risk | Revalidate source |
| ANA-DEF-05 | Skipped rooms reportedly count as completed in aggregator | Can allow skip-only path completion and certificate fraud | Critical; target policy must explicitly reject unintended bypass |
| ANA-DEF-06 | Raw SQL errors reportedly swallowed in progress mutation | Lost progress/outbox events may go unnoticed | Revalidate source |
| ANA-DEF-07 | Frontend dashboard metrics reportedly hardcoded/mocked | KPI values not authoritative | Revalidate source |
| ANA-DEF-08 | Analytics calculations reportedly run client-side over raw entity lists | Scale, privacy, performance, consistency risk | Revalidate source |
| ANA-DEF-09 | Zero-step path reportedly leaves aggregation without update | Stale/inconsistent completion | Revalidate source |

## 9. Data reference

### 9.1 PathProgress — legacy schema described by forensic report
| Field | Meaning |
|---|---|
| id | Progress record ID |
| pathId | Learning path ID |
| userId | Learner ID |
| tenantId | Tenant/college scope |
| stepsCompleted | Aggregate steps counted complete |
| stepsTotal | Total path steps |
| percentComplete | Completion percentage |
| lastActivityAt | Last progress activity |
| completedAt | Completion timestamp, nullable |

Prior report references `prisma/paths/schema.prisma` and `paths.path_progress`. Exact current source/migration status must be rechecked.

### 9.2 Room progress — legacy raw SQL description
| Field | Meaning |
|---|---|
| id | Record ID |
| room_id | Room |
| user_id | Learner |
| tenant_id | Tenant |
| status | Reported values: `in_progress`, `skipped`, `completed` |
| started_at | Start timestamp |
| skipped_at | Skip timestamp |
| completed_at | Completion timestamp |

This is reported as an unmapped table. It is not approved as the target data model.

### 9.3 Competitive rating
Prior forensic report describes `identity.persons.platformRating`, default rating 1200, floor 100, K-factor 32, pairwise ELO update, and Redis sorted sets. Exact target rating entity, algorithm versioning, rating scope, and recalculation policy remain TBD.

## 10. Progress rules and policy warnings

### 10.1 Legacy reported path completion formula
`percentComplete = completedSteps / totalSteps × 100` when totalSteps > 0; zero-step paths reportedly cause the aggregator to skip updating the record.

The prior reconstruction says:
- Room status `completed` counts as complete.
- Room status `skipped` also counts as complete.
- Non-room steps are counted complete unconditionally by the aggregator.

**Do not adopt these as target requirements without product/academic approval.** Completion must distinguish learning completion, administrative bypass, optional steps, and required steps.

### 10.2 Certificate eligibility
Module 16 does not issue certificates. Module 12 owns issuance; Module 17 owns state-machine coordination. Path completion signals must not bypass certificate eligibility requirements. A skip-only route must not qualify for a completion credential unless explicitly approved as a policy exception.

### 10.3 Rating formula (legacy report)
Prior reconstruction describes initial rating 1200, floor 100, K=32, and pairwise ELO calculation for ranked participants. These values are **legacy reported values**, not approved target configuration. Draw handling, event eligibility, ties, rating update idempotency, and recalculation/versioning require decision.

## 11. Cross-module consistency

| Module | Dependency |
|---|---|
| 1 Authentication & User Lifecycle | Self-identity, roles, account state, session authorization |
| 4 Course & Enrollment | Course enrollment, academic membership, course progress relationships |
| 5 Problem Management | Problem metadata and difficulty/tags |
| 6 Code Submission & Judge0 | Submission attempts and execution outcomes |
| 7 Test Case Evaluation | Accepted/rejected verdicts, score, evaluation lifecycle |
| 9 Assessments | Assessment attempts, score/grade, attempts, time window |
| 10 Groups | Group/cohort membership and scope |
| 11 Contests | Event participation, standings, rating inputs |
| 12 Certificates | Eligibility and issuance; analytics cannot directly issue credentials |
| 13 Notifications | Optional analytics threshold/event notifications, if approved |
| 15 Audit Logging | Analytics access/export audit |
| 17 Functional State Machines | Path, room, assessment, contest and certificate transitions |
| 18 Edge Cases & Failure Handling | Queue retries, duplicate events, consistency, partial failures |

## 12. Decision log

| ID | Decision | Considerations | Status |
|---|---|---|---|
| ANA-D01 | Define analytics scope and authoritative metrics | Personal, cohort, tenant, global; source of truth per metric | Open |
| ANA-D02 | Define role-to-analytics permission matrix | HOD/Manager/Finance/Super Admin and student visibility | Open |
| ANA-D03 | Define faculty/instructor role mapping | Existing roles vs separate faculty role | Open |
| ANA-D04 | Define path completion semantics | Required/optional steps, skipped, content-only steps | Open |
| ANA-D05 | Define certificate eligibility relation to path progress | Module 12/17 gate; skip/bypass restrictions | Open |
| ANA-D06 | Define zero-step/empty-path behavior | 0%, not applicable, invalid path, or completion | Open |
| ANA-D07 | Define progress aggregation consistency | Sync/async, eventual consistency, outbox/retry | Open |
| ANA-D08 | Define dashboard KPI catalogue | Metrics, formula, period, scope, freshness | Open |
| ANA-D09 | Define assessment analytics metrics | Score, attempts, pass rate, average, percentile, privacy | Open |
| ANA-D10 | Define event/contest analytics metrics | Registrations, participation, completion, ranks | Open |
| ANA-D11 | Define cohort analytics | Cohort definition, student inclusion, aggregation, drill-down | Open |
| ANA-D12 | Define leaderboard ranking | Scope, ties, eligibility, pagination, visibility | Open |
| ANA-D13 | Define rating algorithm policy | ELO parameters, rank ties, event scope, versioning | Open |
| ANA-D14 | Define rating update idempotency/recalculation | Duplicate EventCompleted, corrected contest results | Open |
| ANA-D15 | Define analytics aggregation architecture | Backend API, query/OLAP/cache, limits, refresh | Open |
| ANA-D16 | Define privacy/minimum cohort threshold | Student visibility, re-identification, role access | Open |
| ANA-D17 | Define export capabilities | CSV/XLSX, permission, limits, redaction, audit | Open |
| ANA-D18 | Define analytics retention and deletion behavior | Academic records, account deletion, tenant retention | Open |
| ANA-D19 | Define progress and analytics observability | Metrics, freshness, lag, data quality alerts | Open |
| ANA-D20 | Define metric definitions and versioning | Formula changes, historical comparability, time zone | Open |
| ANA-D21 | Define dashboard loading/failure/empty-state contract | Mock replacement and UI behavior | Open |
| ANA-D22 | Define cross-module event contract | Event schema, versioning, producers, consumers | Open |

## 13. Review status and readiness

- [x] Prior Module 16 forensic reconstruction reviewed.
- [x] Business Flow v2, Page–DB Mapping, RBAC, schema, and authoring standard reviewed.
- [x] Functional inventory and reported defects captured.
- [x] Function-level draft and QA companion prepared.
- [ ] Current source implementation independently revalidated.
- [ ] Metric definitions and permission matrix approved.
- [ ] Completion/certificate rules approved with Module 12/17.
- [ ] Rating policy approved with Module 11.
- [ ] Privacy, export, retention, and analytics architecture approved.
- [ ] Product, academic, security/privacy, and architecture sign-off completed.

**Implementation readiness:** Not ready. Open decisions ANA-D01–D22 materially affect target behavior.

## 14. Review workflow
1. Product/academic owners approve metrics and learning completion semantics.
2. Security/privacy approves student-level visibility, cohort thresholds, and export.
3. Architecture approves aggregation source, freshness, query strategy, and event consistency.
4. Module 11/12/17 owners resolve rating/certificate/state-machine dependencies.
5. QA updates decision-dependent expected outcomes.
6. Only approved decisions are converted into target requirements.
