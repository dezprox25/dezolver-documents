# Module 2 — College / Department / Section Management
## QA Test Cases

**Document ID:** DEZ-ORG-QA-002  
**Version:** 0.1 — Detailed QA draft  
**Status:** Draft; expected behavior for unresolved decisions is decision-gated  
**Specification:** [Module 2 README](./README.md)

> Test cases are derived from source-supported permission/scope and hierarchy requirements. Where behavior depends on an open decision, the case validates that the product decision must be made before implementation; it does not approve a specific option.

| Test ID | Scenario | Preconditions / test data | Steps | Expected result | Trace |
|---|---|---|---|---|---|
| ORG-QA-001 | Create onboarding record as authorized actor | SUPER_ADMIN session; valid candidate data | Open onboarding; submit | Record saved only if required field policy is approved; returned record/reference | ORG-F001, D01/D02 |
| ORG-QA-002 | Unauthorized onboarding submission | Student/Manager session | Attempt onboarding create endpoint/page | Denied; no record created | ORG-F001, RBAC |
| ORG-QA-003 | Missing onboarding required field | Required-field contract approved; omit one required field | Submit | Validation error; no record created | ORG-F001, D02 |
| ORG-QA-004 | Invalid contact email | Email validation approved; malformed email | Submit | Reject according to approved email validation; no record created | ORG-F001, D02 |
| ORG-QA-005 | Onboarding duplicate candidate | Duplicate policy approved; submit duplicate name/domain/contact | Submit | Outcome matches approved duplicate policy; no silent merge | ORG-F001, D02/D09 |
| ORG-QA-006 | Onboarding save failure | Simulated storage failure | Submit | No false success; retry/idempotency follows approved behavior | ORG-F001 |
| ORG-QA-007 | Authorized reviewer views onboarding list | Reviewer permission approved; multiple records | Open onboarding list | Only authorized records returned; UI list details per approved fields | ORG-F002, D03/D13 |
| ORG-QA-008 | Unauthorized user views onboarding detail | Student/Manager session; known onboarding ID | Request detail | Denied/not-found per approved security response; no contact data leaked | ORG-F002, D36 |
| ORG-QA-009 | Onboarding ID tampering | Reviewer for one scope; modify record ID to another scope | Request details | Scope check denies access | ORG-F002, D13/D36 |
| ORG-QA-010 | Review onboarding record | Review process approved; eligible onboarding record | Open record; perform approved review action | Reviewer/time recorded according to approved status transition | ORG-F003, D03/D04 |
| ORG-QA-011 | Unauthorized review | Actor lacks reviewer permission | Attempt review | Denied; review fields unchanged | ORG-F003 |
| ORG-QA-012 | Review record already reviewed | Review lifecycle/transition approved | Repeat review action | Outcome follows approved transition/re-review rule; no unintended duplicate state | ORG-F003, D03 |
| ORG-QA-013 | Review data not equal payment state | Review and payment semantics clarified | Set paymentStatus/review outcome | Approval state and payment status are not conflated unless explicitly approved | ORG-F003, D05 |
| ORG-QA-014 | Create College as SUPER_ADMIN | SUPER_ADMIN; valid College input | Open College Management; create | College created under approved field/default rules | ORG-F004, D08 |
| ORG-QA-015 | Create College as Finance Team | Finance Team assigned to College | Attempt create | Denied; Finance Team College permission is View only | ORG-F004, RBAC |
| ORG-QA-016 | Create College as HOD | HOD session | Attempt create | Denied; College CRUD belongs to SUPER_ADMIN | ORG-F004, RBAC |
| ORG-QA-017 | College create invalid data | Approved field validation; submit invalid name/domain | Submit | Validation error; no College created | ORG-F004, D08/D09 |
| ORG-QA-018 | Onboarding conversion retry | Conversion linkage and idempotency approved | Convert same onboarding twice | Duplicate College not created; behavior follows approved link/idempotency | ORG-F004, D06/D07 |
| ORG-QA-019 | SUPER_ADMIN lists Colleges | SUPER_ADMIN | Open College Management | Global College records returned | ORG-F005 |
| ORG-QA-020 | Finance Team lists College | Finance Team assigned College A; College B exists | List/request College records | Only College A and authorized subcontext returned | ORG-F005, RBAC |
| ORG-QA-021 | Finance Team direct-ID cross-College access | Finance Team A; College B ID known | Request College B by ID | Denied/not-found per approved response; no B data returned | ORG-F005, D36 |
| ORG-QA-022 | HOD College view permission | HOD assigned Department A; matrix/page discrepancy unresolved | Request College-level view | Access conforms to approved HOD view decision; do not assume full College access | ORG-F005, D12 |
| ORG-QA-023 | SUPER_ADMIN updates College approved field | Existing College; editable field policy approved | Edit permitted field; save | Updated value persists; timestamps follow schema policy | ORG-F006, D14 |
| ORG-QA-024 | Finance Team attempts College update | Finance Team session | Change College name/domain | Denied; College unchanged | ORG-F006, RBAC |
| ORG-QA-025 | HOD attempts College update | HOD session | Change College field | Denied; College unchanged | ORG-F006, RBAC |
| ORG-QA-026 | Attempt to update protected seat counter | College seat policy owned by Module 3 | Send update changing usedSeats/totalSeats | Update denied or handled only through approved Module 3 workflow | ORG-F006, D10/D41 |
| ORG-QA-027 | Change College status using approved transition | SUPER_ADMIN; transition matrix approved | Select allowed status; confirm | Status changes and downstream side effects match approved transition | ORG-F007, D16–D18 |
| ORG-QA-028 | Invalid College status transition | Transition matrix approved; request forbidden state | Submit transition | Reject; status remains unchanged | ORG-F007, Module 17 |
| ORG-QA-029 | College status does not trigger unapproved cascade | Dependent Users/Departments/Subscription exist | Change College status | No unapproved automatic child/subscription cascade | ORG-F007/F020, D17–D18/D39 |
| ORG-QA-030 | Non-SUPER_ADMIN changes College status | Finance Team/HOD/Manager | Attempt status change | Denied unless approved permission changes | ORG-F007, RBAC |
| ORG-QA-031 | Create Department under valid College | SUPER_ADMIN; active/existing College | Select College; create Department | Department has valid collegeId and appears under correct parent | ORG-F008, ORG-R01 |
| ORG-QA-032 | Department creation with nonexistent College | SUPER_ADMIN; invalid collegeId | Submit create | Reject; no Department created | ORG-F008, ORG-V01 |
| ORG-QA-033 | Department create as Finance Team | Finance Team | Attempt create | Denied; Finance Team is View only | ORG-F008, RBAC |
| ORG-QA-034 | Department create as HOD | HOD session | Attempt create | Denied under current RBAC matrix (HOD View/Update) | ORG-F008, RBAC |
| ORG-QA-035 | Department duplicate name/code | Constraint policy approved; duplicate candidate | Submit create | Result matches approved uniqueness policy | ORG-F008, D19 |
| ORG-QA-036 | SUPER_ADMIN views Department hierarchy | College has departments and sections | Open College Details | Authorized Department/Section hierarchy displayed per page design | ORG-F009 |
| ORG-QA-037 | Finance Team views Department in assigned College | Finance assigned College A | List Department records | Only College A departments returned | ORG-F009, RBAC |
| ORG-QA-038 | HOD views assigned Department | HOD assigned College A/Dept A | Open Department Management | Assigned Department and permitted Sections returned | ORG-F009 |
| ORG-QA-039 | HOD direct access to another Department | HOD Dept A; Dept B in same College | Request Dept B | Denied; Department scope cannot be bypassed | ORG-F009, RBAC |
| ORG-QA-040 | SUPER_ADMIN updates Department | Existing Department; approved editable field | Change name/code; save | Approved field updated | ORG-F010 |
| ORG-QA-041 | HOD updates permitted Department field | HOD assigned Department; field allowlist approved | Update permitted field | Field updated within assigned Department | ORG-F010, D20 |
| ORG-QA-042 | HOD updates field outside allowlist | HOD; protected field such as collegeId/hodId/seats | Submit update | Denied; protected value unchanged | ORG-F010, D20 |
| ORG-QA-043 | Finance Team updates Department | Finance Team | Attempt edit | Denied; View only | ORG-F010, RBAC |
| ORG-QA-044 | Department parent College tampering | Existing Dept; send collegeId of another College | Update | Reject parent move unless approved migration workflow; no cross-tenant move | ORG-F010, D22 |
| ORG-QA-045 | Delete Department with no dependencies | SUPER_ADMIN; delete policy approved; no dependent records | Delete/archive action | Result follows approved delete semantics | ORG-F011, D23 |
| ORG-QA-046 | Delete Department with Sections/users/resources | SUPER_ADMIN; dependent records exist | Attempt delete | Restrict/archive/cascade exactly as approved; no implicit cascade | ORG-F011, D23 |
| ORG-QA-047 | Unauthorized Department delete | HOD/Finance Team | Attempt delete | Denied; record and dependents unchanged | ORG-F011, RBAC |
| ORG-QA-048 | Create Section under valid Department | Section CRUD permission approved; valid Department | Create Section | Section has correct departmentId and inherited College scope | ORG-F012, ORG-R02/R03 |
| ORG-QA-049 | Create Section under nonexistent Department | SUPER_ADMIN; invalid departmentId | Submit | Reject; no Section created | ORG-F012, ORG-V02 |
| ORG-QA-050 | HOD Section create permission | HOD assigned Department; policy pending | Attempt create | Behavior follows approved Section permission; no assumption before decision | ORG-F012, D24 |
| ORG-QA-051 | Section create with mismatched College/Department | Request includes parent values that conflict | Submit | Reject; Department parent chain authoritative | ORG-F012, ORG-R03 |
| ORG-QA-052 | Duplicate Section name/code | Uniqueness policy approved | Create duplicate | Result matches approved policy | ORG-F012, D25 |
| ORG-QA-053 | SUPER_ADMIN views Sections | College with multiple departments/sections | Open College Details | Global permitted Sections shown | ORG-F013 |
| ORG-QA-054 | HOD views assigned Department Sections | HOD assigned Dept A; sections A1/A2, Dept B section B1 | Open department view | A1/A2 shown; B1 not shown | ORG-F013, RBAC |
| ORG-QA-055 | Manager sees assigned Section context | Manager assigned Section A | Open permitted operational view | Only assigned Section context is available | ORG-F013, RBAC |
| ORG-QA-056 | Manager enumerates other Sections | Manager assigned Section A; Section B exists | Change section ID/query | Denied; no Section B information leaked | ORG-F013, RBAC |
| ORG-QA-057 | Update Section name/code | Section permission and editable fields approved | Edit approved field | Change persists in correct parent scope | ORG-F014, D24/D25 |
| ORG-QA-058 | HOD Section update permission | HOD assigned Department; permission unresolved | Attempt update | Outcome follows approved permission matrix | ORG-F014, D24 |
| ORG-QA-059 | Section parent Department tampering | Existing Section; update departmentId to another Dept | Submit | Reject unless approved migration flow | ORG-F014, D27 |
| ORG-QA-060 | Finance Team/Manager update Section config | Finance Team/Manager | Attempt update | Denied unless explicit role permission approved | ORG-F014, RBAC |
| ORG-QA-061 | Delete Section with no dependencies | Section delete policy approved; no dependent records | Delete/archive | Result matches approved lifecycle policy | ORG-F015, D28 |
| ORG-QA-062 | Delete Section with dependent Users/academic data | Dependencies exist | Attempt delete | Restrict/archive/reassign/cascade only as approved; no implicit cascade | ORG-F015, D28 |
| ORG-QA-063 | Unauthorized Section delete | HOD/Manager/Student without explicit permission | Attempt delete | Denied; Section unchanged | ORG-F015, D24/D28 |
| ORG-QA-064 | Assign eligible HOD | HOD assignment policy approved; eligible User exists | Select HOD; save | Department.hodId references eligible user; assignment shown | ORG-F016, D29–D31 |
| ORG-QA-065 | Assign User with non-HOD role | User exists but role != HOD | Attempt assign | Reject under approved eligibility policy | ORG-F016, D31 |
| ORG-QA-066 | Assign HOD from another College/Department | HOD candidate scope mismatch | Attempt assign | Reject or follow approved multi-department rule; no unauthorized assignment | ORG-F016, D30/D31 |
| ORG-QA-067 | Unassign/reassign HOD | Department currently assigned HOD | Change assignment | Outcome follows approved replacement/removal rules; no dangling reference | ORG-F016, D30/D31 |
| ORG-QA-068 | User assigned to valid College/Department/Section chain | Module 1 user create/update; valid references | Save user assignment | Module 1 persists assignment only after hierarchy validation | ORG-F017 |
| ORG-QA-069 | User Department belongs to different College | User request College A, Department B under College B | Submit | Reject; no mismatched user scope stored | ORG-F017, ORG-V03 |
| ORG-QA-070 | User Section belongs to different Department | User Dept A, Section under Dept B | Submit | Reject; no mismatched user scope stored | ORG-F017, ORG-V04 |
| ORG-QA-071 | Manager has section outside assigned department | Manager user placement inconsistent | Create/update assignment | Reject under approved Manager scope policy | ORG-F017, D34 |
| ORG-QA-072 | Student Section nullability | Student assignment without Section | Submit | Outcome follows approved Student placement policy | ORG-F017, D33 |
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
| ORG-QA-083 | Concurrent parent deletion/update | Parent record changes during child creation/update | Execute concurrent operations | Transaction/conflict outcome follows approved integrity contract; no orphan record | ORG-F019, D38 |
| ORG-QA-084 | College status impact on organization | College status transition approved; dependents exist | Change status | Child/User/Subscription effects match approved policy; no unapproved cascade | ORG-F020, D16–D18/D39 |
| ORG-QA-085 | Status transition invalid | Approved transition matrix; invalid state requested | Submit | Reject; state unchanged | ORG-F020, Module 17 |
| ORG-QA-086 | Status change actor unauthorized | Finance Team/HOD/Manager | Attempt status change | Denied | ORG-F020, RBAC |
| ORG-QA-087 | Status change audit trace | Audit event approved in Module 15 | Change status | Actor/target/action/time recorded according to approved audit spec | ORG-F020, D40 |
| ORG-QA-088 | Onboarding privacy/retention access | Contact data exists; permissions/retention approved | Request as unauthorized role | Deny/minimize data; retention follows approved policy | ORG-F001/F002, D40 |
| ORG-QA-089 | Parent-child data leak through search | HOD/Manager/Finance search query includes foreign names/IDs | Search | Search results respect same scope as direct list/detail | ORG-F018, RBAC |
| ORG-QA-090 | Seat counters edited outside billing workflow | College/Department seat values exist; Module 3 policy | Attempt direct edit in organization UI/API | Denied or delegated to Module 3-approved operation; no inconsistent counter update | ORG-F006/F008/F010, D41 |

## QA execution note

Before execution, product/security/architecture decisions referenced by each test must be resolved. Tests with unresolved decisions should be tagged blocked or updated with the approved outcome; they must not be used to infer the missing product behavior.
