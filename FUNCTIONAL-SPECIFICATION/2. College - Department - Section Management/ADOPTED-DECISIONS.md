# Module 2 — Adopted Product & Architecture Decisions

**Document ID:** DEZ-ORG-DEC-002  
**Decision date:** 2026-11-06  
**Applies to:** College, Department & Section Management  
**Authority:** Product owner authorized the remaining minimal decisions to be resolved using the recommended answers. This document records that adopted baseline.  
**Status:** Adopted target baseline; implementation and cross-module/schema alignment pending.

## Decision precedence

This document supersedes conflicting TBD, “decision required”, or unapproved-recommendation wording in the Module 2 functional specification and QA companion for ORG-D01–ORG-D42. It does not claim that migrations, APIs, UI, or dependent modules have already been implemented. Legacy Module 02 material remains historical evidence only.

## ORG-D01–ORG-D42 — adopted baseline

| ID | Adopted decision |
|---|---|
| D01 | SUPER_ADMIN only submits College onboarding. SUPER_ADMIN may submit on behalf of a College contact. |
| D02 | Required onboarding fields: collegeName, contactPerson, contactEmail. Optional: phone, address, studentCount, departmentDetails. Validate email and non-negative studentCount. Normalize email/domain/name and flag likely duplicates for SUPER_ADMIN review; never auto-merge or reject on name match alone. |
| D03 | Onboarding states: SUBMITTED → UNDER_REVIEW → NEEDS_INFORMATION / APPROVED / REJECTED. SUPER_ADMIN reviews. Approval does not create the College automatically. |
| D04 | reviewedById and reviewedAt store the latest reviewer and review time. Preserve every review action in Module 15 audit history. |
| D05 | Deprecate/remove CollegeOnboarding.paymentStatus from onboarding workflow. Module 3 owns payments, invoices and subscription state. |
| D06 | Add nullable unique CollegeOnboarding.collegeId FK to College.id; set atomically during conversion. |
| D07 | After APPROVED, SUPER_ADMIN separately selects Create College. Repeated conversion returns the linked College and never creates a duplicate. |
| D08 | College name is required. Domain is optional until verified. New College defaults to ACTIVE after explicit creation. Seats remain Module 3-owned. |
| D09 | College names may repeat globally. A populated domain must be normalized, verified and globally unique. |
| D10 | Seat allocation and usage come from Module 3. Module 2 displays seat counters read-only. |
| D11 | College creation is onboarding-led. SUPER_ADMIN may create onboarding internally/on behalf of a College; no direct-create bypass outside onboarding. |
| D12 | HOD sees limited parent College identity context only; no College Management/API administration or financial fields. |
| D13 | FINANCE_TEAM sees its assigned College and basic Department details. Department summary may show aggregate Section counts; no Section-level academic detail or Section administration. |
| D14 | SUPER_ADMIN may edit College name and verified domain. Status changes use Module 17. Seat/system fields are not directly editable in Module 2. |
| D15 | College.domain means the primary official institutional email domain, optional and verified before enforcement. It is not a website URL or Dezolver login subdomain. Multiple domains require a future CollegeDomain model. |
| D16 | College uses EntityStatus: ACTIVE, SUSPENDED, DEACTIVATED, ARCHIVED. SUPER_ADMIN changes status through Module 17. New College defaults ACTIVE; archive/restore is reversible through approved transitions. |
| D17 | Non-ACTIVE College blocks new logins and protected requests for memberships in that College and revokes those College-scoped sessions. Do not deactivate global identity or unrelated memberships. |
| D18 | College status and Subscription status are independent. College status gates organization availability; Subscription status controls paid entitlements/seats. Billing must not silently rewrite College status. |
| D19 | Department requires collegeId, name and code. Name and code are normalized/case-insensitive unique within a College. hodId may be null. Seat fields are Module 3-managed. |
| D20 | HOD may update assigned Department name and code only. HOD cannot change collegeId, hodId, seats or lifecycle status. SUPER_ADMIN may update non-system fields except seats. |
| D21 | Department Management shows name, code, assigned HOD, Section count, Manager/Student counts and read-only seat summary where Module 3 provides it. Support search by name/code, HOD filter, stable sort and pagination. |
| D22 | Department.collegeId is immutable in normal edits. Only SUPER_ADMIN may perform controlled migration after validating dependent memberships/academic references, in a transaction and with audit. |
| D23 | Department supports reversible archive/restore; never hard-delete. Preserve Sections and history. Archived parent makes descendant access ineffective; no cascade delete. |
| D24 | SUPER_ADMIN and the HOD assigned to the parent Department may create/update/archive/restore Sections within scope. FINANCE_TEAM is read-only at Department summary level; MANAGER/STUDENT cannot administer Section structure. |
| D25 | Section requires departmentId, name and code. Name and code are normalized/case-insensitive unique within a Department. departmentId is immutable in normal edits. |
| D26 | FINANCE_TEAM sees Department-level Section counts only. MANAGER sees only explicitly assigned Section(s) and permitted Group context, read-only for organization structure. |
| D27 | Section.departmentId is immutable in normal edits. Only SUPER_ADMIN may perform controlled cross-Department migration after validating memberships/courses/groups, in a transaction and with audit. |
| D28 | Section supports archive/restore; never hard-delete. Archive blocks new assignments/enrollments and new Section-scoped work; preserve historical links. Reassignment belongs to Modules 1/4 as applicable. |
| D29 | SUPER_ADMIN and FINANCE_TEAM may assign HOD within the Finance Team's assigned College. Finance Team action is limited to HOD provisioning/assignment; it cannot change Department parent or seat fields. |
| D30 | One HOD reference per Department (Department.hodId). The same HOD may head multiple Departments through separate memberships. |
| D31 | HOD candidate must be ACTIVE and have HOD role in a membership for the same College and target Department. Unassign/reassign updates hodId; do not deactivate global User. Revoke affected membership sessions per Module 1. |
| D32 | Use Module 1 global User plus multiple organizational memberships. Role and scope belong to each membership; activate one workspace context at a time. Direct single organization fields on User are not the authorization source of truth. |
| D33 | Student Section membership is optional at initial identity provisioning. It is required before Section-scoped courses, groups or activities; Module 4 controls enrollment eligibility. |
| D34 | Manager may have multiple Section memberships, including across Departments/Colleges only where each membership is explicitly provisioned. Each active context remains limited to that membership. |
| D35 | Module 4 owns academicBatch meaning, values and eligibility. Module 2 validates organization membership and does not derive academicBatch from Section. |
| D36 | Return 404 for a resource outside actor data scope to prevent enumeration; return 403 if the resource is in scope but the action is not permitted. Use generic error payloads. |
| D37 | On Department/Section/role membership change, revoke affected membership sessions and require authentication/workspace selection again. Unrelated memberships remain active. |
| D38 | Use database transactions and FK constraints for parent-child writes; use optimistic concurrency/version checks for mutable organization records. Conflicts return a retryable conflict and never leave orphan/mismatched links. |
| D39 | Department/Section have their own lifecycle status. Effective access requires College, Department and Section ancestors to be ACTIVE. Parent status does not rewrite child status; restoring a parent re-enables only active descendants. |
| D40 | Audit all College/Department/Section create/update/archive/restore/status/parent-migration/HOD-assignment events. Notify affected HODs/Managers on assignment/scope changes and relevant College administrators on College status changes. Modules 13/15 define delivery, events and retention. |
| D41 | Seat counts and allocations are read-only in Module 2. Purchase/allocation/adjustment/quota changes go through Module 3 only. |
| D42 | College, Department and Section support reversible archive/restore. SUPER_ADMIN controls College/Department; SUPER_ADMIN and assigned HOD control Section within scope. No permanent deletion. |

## Required schema and data-contract changes

The supplied schema snapshot does not show these changes as implemented. They are required before the dependent behavior can be considered implementation-ready.

1. **CollegeOnboarding:** add status enum SUBMITTED, UNDER_REVIEW, NEEDS_INFORMATION, APPROVED, REJECTED; add nullable unique collegeId FK to College; retain latest review metadata; deprecate/remove paymentStatus from onboarding workflow.
2. **Department:** add status EntityStatus NOT NULL DEFAULT ACTIVE; add normalized composite uniqueness for (collegeId, name) and (collegeId, code).
3. **Section:** add status EntityStatus NOT NULL DEFAULT ACTIVE; add normalized composite uniqueness for (departmentId, name) and (departmentId, code).
4. **College.domain:** nullable verified primary institutional email domain; globally unique when populated.
5. **Membership:** use Module 1 membership records as the source of role/scope; reconcile/deprecate direct User.collegeId/departmentId/sectionId as authorization source.
6. **Integrity:** add FK/index coverage for CollegeOnboarding.collegeId, Department.collegeId, Section.departmentId, Department.hodId, parent-scoped uniqueness and membership lookups.
7. Do not configure physical-delete cascade for College, Department, Section or dependent identity/academic records.

## Canonical RBAC alignment required

- Resolve the source conflict where the platform matrix gives HOD College View while the API matrix gives HOD no College API access: adopted behavior is limited parent College identity context only, with no College Management/admin API access.
- Add an explicit Section Management permission row: SUPER_ADMIN and assigned HOD may manage Sections within scope; FINANCE_TEAM receives aggregate counts only; MANAGER sees assigned Section context but cannot administer structure; STUDENT has no organization-admin access.
- HOD Department update is limited to name/code. FINANCE_TEAM may perform HOD provisioning/assignment within its College but gains no general Department write permission.
- These alignments must be applied to the canonical RBAC source before implementation is treated as complete.

## Cross-module contracts

| Module | Required alignment |
|---|---|
| 1 — Authentication & User Lifecycle | Global User/membership model is authorization source; membership changes revoke affected sessions; FINANCE_TEAM provisioning and HOD assignment align. |
| 3 — Subscription & Seat Management | Owns seats, quotas, payment/subscription states and entitlement enforcement. |
| 4 — Course & Enrollment | Owns academicBatch and Section/course enrollment eligibility. |
| 10 — Groups | Defines permitted Manager Group scope and Section–Group association. |
| 13 — Notifications | Defines notification events, recipients, templates and delivery. |
| 15 — Audit Logging | Defines canonical audit events, review history, retention and visibility. |
| 17 — Functional State Machines | Defines College/Department/Section/onboarding transition graphs. |
| 18 — Edge Cases & Failure Handling | Defines shared conflict, retry, concurrency and error response contracts. |

## Implementation gate

The product decision baseline is adopted. The following are engineering/cross-module dependencies, not unanswered ORG-D01–D42 product questions: apply schema migrations; reconcile the canonical RBAC and Module 1 membership contracts; implement Module 3 seat/payment ownership; define Module 4 enrollment integration; align notification/audit/state-machine/failure contracts. QA cases that depend on those unfinished contracts or migrations remain blocked until those prerequisites are delivered.
