# Module 2 — College / Department / Section Management
## QA Test Cases

**Document ID:** DEZ-ORG-QA-002  
**Version:** 0.2 — Adopted decision baseline  
**Status:** QA draft aligned to adopted ORG-D01–ORG-D42; schema and cross-module dependencies remain gated  
**Specification:** [Module 2 README](./README.md)

> Test cases reflect the adopted Module 2 decision baseline. Where a case depends on schema migrations or an unfinished cross-module contract, it remains blocked until that prerequisite is delivered. See [ADOPTED-DECISIONS.md](./ADOPTED-DECISIONS.md).

| Test ID | Scenario | Preconditions / test data | Steps | Expected result | Trace |
|---|---|---|---|---|---|
|ORG-QA-001 | Create onboarding record as authorized actor | SUPER_ADMIN session; collegeName, contactPerson and valid contactEmail supplied | Open onboarding; submit | Create a SUBMITTED onboarding record and return its reference. Only SUPER_ADMIN can submit. | ORG-F001, D01/D02|
| ORG-QA-002 | Unauthorized onboarding submission | Student/Manager session | Attempt onboarding create endpoint/page | Denied; no record created | ORG-F001, RBAC |
|ORG-QA-003 | Missing onboarding required field | SUPER_ADMIN session; omit collegeName or contactPerson or contactEmail | Submit | Reject with field validation errors; do not create an onboarding record. | ORG-F001, D02|
|ORG-QA-004 | Invalid contact email | SUPER_ADMIN session; malformed contactEmail | Submit | Reject malformed email; do not create an onboarding record. | ORG-F001, D02|
|ORG-QA-005 | Onboarding duplicate candidate | Existing candidates with matching normalized email/domain/name | Submit a likely duplicate | Flag the candidate for SUPER_ADMIN review. Do not silently merge or reject solely because the College name matches. | ORG-F001, D02/D09|
| ORG-QA-006 | Onboarding save failure | Simulated storage failure | Submit | No false success; retry/idempotency follows approved behavior | ORG-F001 |
|ORG-QA-007 | Authorized reviewer views onboarding list | SUPER_ADMIN session; multiple onboarding records | Open onboarding list | Return onboarding records only to SUPER_ADMIN; non-SUPER_ADMIN roles cannot view onboarding contact data. | ORG-F002, D03/D13|
| ORG-QA-008 | Unauthorized user views onboarding detail | Student/Manager session; known onboarding ID | Request detail | Denied/not-found per approved security response; no contact data leaked | ORG-F002, D36 |
| ORG-QA-009 | Onboarding ID tampering | Reviewer for one scope; modify record ID to another scope | Request details | Scope check denies access | ORG-F002, D13/D36 |
|ORG-QA-010 | Review onboarding record | SUPER_ADMIN session; onboarding in a reviewable state | Move through an allowed review transition | Persist the new review status and latest reviewedById/reviewedAt; preserve each review action in Module 15 audit history. | ORG-F003, D03/D04|
| ORG-QA-011 | Unauthorized review | Actor lacks reviewer permission | Attempt review | Denied; review fields unchanged | ORG-F003 |
| ORG-QA-012 | Review record already reviewed | Review lifecycle/transition approved | Repeat review action | Outcome follows approved transition/re-review rule; no unintended duplicate state | ORG-F003, D03 |
|ORG-QA-013 | Review data not equal payment state | Onboarding record; Module 3 payment/subscription contract available | Review onboarding outcome independently of billing | Onboarding review status is not a payment state. paymentStatus is not used by Module 2; Module 3 owns payment and subscription state. | ORG-F003, D05|
|ORG-QA-014 | Create College as SUPER_ADMIN | SUPER_ADMIN session; APPROVED onboarding without collegeId link | Select Create College | Create an ACTIVE College explicitly and persist the CollegeOnboarding.collegeId link atomically. | ORG-F004, D08|
| ORG-QA-015 | Create College as Finance Team | Finance Team assigned to College | Attempt create | Denied; Finance Team College permission is View only | ORG-F004, RBAC |
| ORG-QA-016 | Create College as HOD | HOD session | Attempt create | Denied; College CRUD belongs to SUPER_ADMIN | ORG-F004, RBAC |
|ORG-QA-017 | College create invalid data | SUPER_ADMIN session; missing College name or unverified/invalid domain | Submit College creation | Require a College name. Domain may be omitted; a supplied domain must be normalized and verified before enforcement. | ORG-F004, D08/D09|
|ORG-QA-018 | Onboarding conversion retry | Approved onboarding already linked to a College | Repeat Create College conversion | Return the already linked College. Do not create a duplicate. | ORG-F004, D06/D07|
| ORG-QA-019 | SUPER_ADMIN lists Colleges | SUPER_ADMIN | Open College Management | Global College records returned | ORG-F005 |
| ORG-QA-020 | Finance Team lists College | Finance Team assigned College A; College B exists | List/request College records | Only College A and authorized subcontext returned | ORG-F005, RBAC |
| ORG-QA-021 | Finance Team direct-ID cross-College access | Finance Team A; College B ID known | Request College B by ID | Denied/not-found per approved response; no B data returned | ORG-F005, D36 |
|ORG-QA-022 | HOD College view permission | HOD assigned to a Department; College identity context exists | Request parent College context | Allow only limited basic College identity context; deny College Management/admin API access and financial fields. | ORG-F005, D12|
| ORG-QA-023 | SUPER_ADMIN updates College approved field | Existing College; editable field policy approved | Edit permitted field; save | Updated value persists; timestamps follow schema policy | ORG-F006, D14 |
| ORG-QA-024 | Finance Team attempts College update | Finance Team session | Change College name/domain | Denied; College unchanged | ORG-F006, RBAC |
| ORG-QA-025 | HOD attempts College update | HOD session | Change College field | Denied; College unchanged | ORG-F006, RBAC |
|ORG-QA-026 | Attempt to update protected seat counter | College seat counters exist; caller has Module 2 access | Attempt direct update of totalSeats/usedSeats | Deny changes in Module 2. Seat counters are read-only here and are changed only through Module 3. | ORG-F006, D10/D41|
|ORG-QA-027 | Change College status using approved transition | College has memberships and active sessions; status transition authorized by Module 17 | Set College to a non-ACTIVE status | Block new login/protected requests for memberships in that College and revoke only those College-scoped sessions. | ORG-F007, D16–D18|
| ORG-QA-028 | Invalid College status transition | Transition matrix approved; request forbidden state | Submit transition | Reject; status remains unchanged | ORG-F007, Module 17 |
|ORG-QA-029 | College status does not trigger unapproved cascade | College has Departments, Sections and a Subscription | Change College status | Do not rewrite child or Subscription statuses. Effective descendant access is blocked while an ancestor is non-ACTIVE. | ORG-F007/F020, D17–D18/D39|
| ORG-QA-030 | Non-SUPER_ADMIN changes College status | Finance Team/HOD/Manager | Attempt status change | Denied unless approved permission changes | ORG-F007, RBAC |
| ORG-QA-031 | Create Department under valid College | SUPER_ADMIN; active/existing College | Select College; create Department | Department has valid collegeId and appears under correct parent | ORG-F008, ORG-R01 |
| ORG-QA-032 | Department creation with nonexistent College | SUPER_ADMIN; invalid collegeId | Submit create | Reject; no Department created | ORG-F008, ORG-V01 |
| ORG-QA-033 | Department create as Finance Team | Finance Team | Attempt create | Denied; Finance Team is View only | ORG-F008, RBAC |
| ORG-QA-034 | Department create as HOD | HOD session | Attempt create | Denied under current RBAC matrix (HOD View/Update) | ORG-F008, RBAC |
|ORG-QA-035 | Department duplicate name/code | Department exists in College A; candidate has normalized name/code collision | Create Department | Reject duplicate normalized name or code within the same College; allow the same values in a different College. | ORG-F008, D19|
| ORG-QA-036 | SUPER_ADMIN views Department hierarchy | College has departments and sections | Open College Details | Authorized Department/Section hierarchy displayed per page design | ORG-F009 |
| ORG-QA-037 | Finance Team views Department in assigned College | Finance assigned College A | List Department records | Only College A departments returned | ORG-F009, RBAC |
| ORG-QA-038 | HOD views assigned Department | HOD assigned College A/Dept A | Open Department Management | Assigned Department and permitted Sections returned | ORG-F009 |
| ORG-QA-039 | HOD direct access to another Department | HOD Dept A; Dept B in same College | Request Dept B | Denied; Department scope cannot be bypassed | ORG-F009, RBAC |
| ORG-QA-040 | SUPER_ADMIN updates Department | Existing Department; approved editable field | Change name/code; save | Approved field updated | ORG-F010 |
|ORG-QA-041 | HOD updates permitted Department field | HOD membership assigned to Department A | Update Department name/code and attempt protected-field changes | Allow name/code updates only. Reject changes to collegeId, hodId, seats and lifecycle status. | ORG-F010, D20|
| ORG-QA-042 | HOD updates field outside allowlist | HOD; protected field such as collegeId/hodId/seats | Submit update | Denied; protected value unchanged | ORG-F010, D20 |
| ORG-QA-043 | Finance Team updates Department | Finance Team | Attempt edit | Denied; View only | ORG-F010, RBAC |
| ORG-QA-044 | Department parent College tampering | Existing Dept; send collegeId of another College | Update | Reject parent move unless approved migration workflow; no cross-tenant move | ORG-F010, D22 |
|ORG-QA-045 | Delete Department with no dependencies | SUPER_ADMIN; Department has no active work requiring a migration | Archive Department | Archive reversibly; preserve child Sections and history. Do not physically delete the Department. | ORG-F011, D23|
|ORG-QA-046 | Delete Department with Sections/users/resources | SUPER_ADMIN; Department has Sections/users/resources | Archive Department | Archive without cascade deletion. Preserve dependents/history; descendants become inaccessible while the parent is archived. | ORG-F011, D23|
| ORG-QA-047 | Unauthorized Department delete | HOD/Finance Team | Attempt delete | Denied; record and dependents unchanged | ORG-F011, RBAC |
|ORG-QA-048 | Create Section under valid Department | SUPER_ADMIN or HOD assigned to the parent Department; valid Department | Create Section | Create Section with the correct departmentId. Enforce parent scope and normalized name/code uniqueness. | ORG-F012, ORG-R02/R03|
| ORG-QA-049 | Create Section under nonexistent Department | SUPER_ADMIN; invalid departmentId | Submit | Reject; no Section created | ORG-F012, ORG-V02 |
|ORG-QA-050 | HOD Section create permission | HOD assigned to the target Department | Create Section within assigned Department | Allow the assigned HOD to create a Section in scope; deny cross-Department or cross-College creation. | ORG-F012, D24|
| ORG-QA-051 | Section create with mismatched College/Department | Request includes parent values that conflict | Submit | Reject; Department parent chain authoritative | ORG-F012, ORG-R03 |
|ORG-QA-052 | Duplicate Section name/code | Section exists in Department A; candidate has normalized name/code collision | Create Section | Reject duplicate normalized name or code within the same Department; allow the same values in another Department. | ORG-F012, D25|
| ORG-QA-053 | SUPER_ADMIN views Sections | College with multiple departments/sections | Open College Details | Global permitted Sections shown | ORG-F013 |
| ORG-QA-054 | HOD views assigned Department Sections | HOD assigned Dept A; sections A1/A2, Dept B section B1 | Open department view | A1/A2 shown; B1 not shown | ORG-F013, RBAC |
| ORG-QA-055 | Manager sees assigned Section context | Manager assigned Section A | Open permitted operational view | Only assigned Section context is available | ORG-F013, RBAC |
| ORG-QA-056 | Manager enumerates other Sections | Manager assigned Section A; Section B exists | Change section ID/query | Denied; no Section B information leaked | ORG-F013, RBAC |
| ORG-QA-057 | Update Section name/code | Section permission and editable fields approved | Edit approved field | Change persists in correct parent scope | ORG-F014, D24/D25 |
|ORG-QA-058 | HOD Section update permission | HOD assigned to the Section's parent Department | Update Section name/code and attempt parent change | Allow name/code updates within scope; reject departmentId changes in normal edits. | ORG-F014, D24|
| ORG-QA-059 | Section parent Department tampering | Existing Section; update departmentId to another Dept | Submit | Reject unless approved migration flow | ORG-F014, D27 |
| ORG-QA-060 | Finance Team/Manager update Section config | Finance Team/Manager | Attempt update | Denied unless explicit role permission approved | ORG-F014, RBAC |
|ORG-QA-061 | Delete Section with no dependencies | SUPER_ADMIN or assigned HOD; Section exists | Archive Section | Archive reversibly; preserve historical records and relationships. Do not physically delete the Section. | ORG-F015, D28|
|ORG-QA-062 | Delete Section with dependent Users/academic data | Section has dependent Users/academic records | Archive Section | Preserve dependents/history and block new assignments, enrollments and new Section-scoped work; do not cascade-delete. | ORG-F015, D28|
| ORG-QA-063 | Unauthorized Section delete | HOD/Manager/Student without explicit permission | Attempt delete | Denied; Section unchanged | ORG-F015, D24/D28 |
|ORG-QA-064 | Assign eligible HOD | SUPER_ADMIN or FINANCE_TEAM scoped to the target College; eligible HOD candidate | Assign HOD to Department | Allow assignment only within the actor's College scope. Department.hodId references an eligible HOD membership in the target College/Department. | ORG-F016, D29–D31|
|ORG-QA-065 | Assign User with non-HOD role | Candidate User is active but has no HOD role in the required membership | Attempt HOD assignment | Reject assignment; candidate must be ACTIVE and hold HOD role in the target College and Department membership. | ORG-F016, D31|
|ORG-QA-066 | Assign HOD from another College/Department | HOD candidate belongs to a different College or lacks membership in the target Department | Attempt HOD assignment | Reject the cross-scope assignment. A HOD may lead multiple Departments only through separately valid memberships. | ORG-F016, D30/D31|
|ORG-QA-067 | Unassign/reassign HOD | Department currently has an assigned HOD; affected membership session active | Unassign or replace HOD | Update Department.hodId and membership scope consistently; revoke only the affected membership session(s). Never deactivate the global User. | ORG-F016, D30/D31|
| ORG-QA-068 | User assigned to valid College/Department/Section chain | Module 1 user create/update; valid references | Save user assignment | Module 1 persists assignment only after hierarchy validation | ORG-F017 |
| ORG-QA-069 | User Department belongs to different College | User request College A, Department B under College B | Submit | Reject; no mismatched user scope stored | ORG-F017, ORG-V03 |
| ORG-QA-070 | User Section belongs to different Department | User Dept A, Section under Dept B | Submit | Reject; no mismatched user scope stored | ORG-F017, ORG-V04 |
|ORG-QA-071 | Manager has section outside assigned department | Manager has an explicit membership for Section A only | Attempt assignment/access to Section B | Reject unless a separate explicit membership for Section B has been provisioned; each workspace context remains restricted to its membership. | ORG-F017, D34|
|ORG-QA-072 | Student Section nullability | Student identity created without a Section membership | Provision Student, then request Section-scoped course/group activity | Allow initial identity provisioning without Section; deny Section-scoped activity until a valid Section membership/enrollment exists. | ORG-F017, D33|
| ORG-QA-073 | SUPER_ADMIN scoped hierarchy lookup | SUPER_ADMIN request all Colleges/Departments/Sections | Query | Global results permitted | ORG-F018 |
| ORG-QA-074 | Finance Team hierarchy lookup | Finance assigned College A; request College B | Query | Results restricted to College A | ORG-F018, RBAC |
| ORG-QA-075 | HOD hierarchy lookup | HOD assigned Dept A; query sibling Dept B | Query | Dept B excluded/denied | ORG-F018, RBAC |
| ORG-QA-076 | Manager hierarchy lookup | Manager Section A; query Sections A and B | Query | Only assigned/permitted Section scope returned | ORG-F018, RBAC |
| ORG-QA-077 | Student attempts organization admin query | Student session | Request College/Department admin endpoints | Denied | ORG-F018, RBAC |
| ORG-QA-078 | Scope tampering on list/search | Finance/HOD/Manager; modify scope query parameter | Query | Backend ignores/validates client scope and returns only allowed data | ORG-F018, RBAC |
| ORG-QA-079 | Department parent integrity | Existing College A/B; create Dept with invalid parent | Submit | Invalid parent rejected | ORG-F019, ORG-R01 |
| ORG-QA-080 | Section parent integrity | Existing Department A/B; create Section with invalid/mismatched parent | Submit | Invalid parent rejected | ORG-F019, ORG-R02 |
| ORG-QA-081 | HOD reference points to nonexistent User | HOD assignment request with invalid User ID | Submit | Reject; Department.hodId unchanged | ORG-F019 |
| ORG-QA-082 | Update request changes parent chain across tenant | Actor sends foreign college/department IDs | Submit | Reject before write; no foreign parent link | ORG-F019, RBAC |
|ORG-QA-083 | Concurrent parent deletion/update | Parent and child records; concurrent update/create requests | Execute competing parent/child operations | Use FK constraints, transaction and optimistic concurrency. Return a retryable conflict where needed; never leave orphan or mismatched parent links. | ORG-F019, D38|
|ORG-QA-084 | College status impact on organization | College has Department/Section descendants and active memberships | Change College status | Block effective access for the inactive College hierarchy and revoke its scoped sessions; leave child and Subscription statuses unchanged. | ORG-F020, D16–D18/D39|
| ORG-QA-085 | Status transition invalid | Approved transition matrix; invalid state requested | Submit | Reject; state unchanged | ORG-F020, Module 17 |
| ORG-QA-086 | Status change actor unauthorized | Finance Team/HOD/Manager | Attempt status change | Denied | ORG-F020, RBAC |
| ORG-QA-087 | Status change audit trace | Audit event approved in Module 15 | Change status | Actor/target/action/time recorded according to approved audit spec | ORG-F020, D40 |
|ORG-QA-088 | Onboarding privacy/retention access | Onboarding contact data exists; caller is not SUPER_ADMIN | Request onboarding list/detail/contact data | Deny access; onboarding contact data is visible only to SUPER_ADMIN. | ORG-F001/F002, D40|
| ORG-QA-089 | Parent-child data leak through search | HOD/Manager/Finance search query includes foreign names/IDs | Search | Search results respect same scope as direct list/detail | ORG-F018, RBAC |
|ORG-QA-090 | Seat counters edited outside billing workflow | College/Department seat values exist; Module 3 is authoritative | Attempt direct edit through Module 2 UI/API | Reject the edit or route only through the approved Module 3 operation; do not update counters in Module 2. | ORG-F006/F008/F010, D41|

## QA execution note

ORG-D01–ORG-D42 are adopted and are no longer product-decision blockers. Before execution, block only cases whose required schema migrations, canonical RBAC/Module 1 membership changes, or Modules 3/4/10/13/15/17/18 contracts are not yet implemented. Do not treat a blocked implementation dependency as an unresolved product decision.
