# Module 5 — Problem Management QA Test Cases

**Version:** 0.1 — QA companion to Functional Specification v0.2  
**Status:** Draft; product/security decisions remain open  
**Spec:** [Problem Management](./README.md)

## Execution conventions
- Test identities: SUPER_ADMIN; Finance-A (College A); HOD-A (College A/Department A); Manager-A/Section A1; Student-A; HOD-B/Department B; Student-B.
- Test records: Problem-A (College A/Department A), Problem-B (College B/Department B), TestCase-A1, Course/Lab/Assessment/Contest association records, and CodeSubmission records.
- **Blocked** means expected behavior cannot be finalized until the referenced PROB-D decision is approved. Security checks grounded in RBAC/derived safeguards remain testable.
- TestCase input/expected output marked hidden must not be exposed to Student payloads.

| Test ID | Scenario | Preconditions / data | Steps | Expected result | Trace/status |
|---|---|---|---|---|---|
| PROB-QA-001 | HOD lists own Department Problems | HOD-A; Problem-A/B | Open Problems page | Problem-A appears; Problem-B excluded | F001/RBAC |
| PROB-QA-002 | HOD requests foreign Problem by ID | HOD-A; Problem-B | Direct API/URL | Deny/not-found; no data leak | F001 |
| PROB-QA-003 | Manager lists assigned Section Problems | Manager-A; Section-scoped content | Open Problems | Only permitted assigned activity/Section content | F001; scope policy TBD |
| PROB-QA-004 | Manager accesses unrelated Section Problem | Manager-A; foreign Section | Direct ID request | Deny; no content disclosed | F001/RBAC |
| PROB-QA-005 | Student accesses permitted Course Problem | Student-A enrolled/eligible | Open Course Problem | Problem statement/safe fields returned | F001/F017 |
| PROB-QA-006 | Student accesses unassigned private Problem | Student-A; Problem outside allowed context | Direct URL/API | Deny/not-found | F001 |
| PROB-QA-007 | SUPER_ADMIN views cross-College Problem | SUPER_ADMIN; Problem-A/B | Search/detail | Global role access follows RBAC | F001 |
| PROB-QA-008 | Finance Team views Problem | Finance-A | Open Problem page/API | Must follow resolved permission matrix; currently blocked | F001; D01 |
| PROB-QA-009 | Client sends foreign College filter | HOD-A | Query College B | Server-side scope remains College A | F001/F021 |
| PROB-QA-010 | Student requests hidden TestCase with Problem detail | Hidden TestCase exists | Request Problem payload | Hidden input/expectedOutput absent | F001/F015 |
| PROB-QA-011 | HOD creates valid Problem in own Department | HOD-A; valid fields | Submit create | Problem saved; creator is HOD-A | F002 |
| PROB-QA-012 | HOD creates Problem in foreign Department | HOD-A; Department B | Submit create | Denied; no record created | F002 |
| PROB-QA-013 | SUPER_ADMIN creates tenant-scoped Problem | SUPER_ADMIN | Submit valid College/Department | Allowed if required fields valid | F002 |
| PROB-QA-014 | Manager creates Problem within assigned scope | Manager-A | Create scoped Problem | Allowed only if approved Manager create rule permits | F002; D01; Blocked |
| PROB-QA-015 | Manager creates Problem outside assigned scope | Manager-A; foreign Section/Department | Submit | Denied | F002/RBAC |
| PROB-QA-016 | Forged createdById | HOD-A; payload names another user | Create Problem | Server derives creator from authenticated actor or rejects field | F002 |
| PROB-QA-017 | Title exceeds 255 characters | Create form/API | Submit >255 chars | Reject; no Problem row | F002/schema |
| PROB-QA-018 | Invalid difficulty enum | difficulty=unsupported | Submit create | Reject | F002 |
| PROB-QA-019 | Invalid ContentScope | scope=unsupported | Submit create | Reject | F002/F005 |
| PROB-QA-020 | Missing required field | Requiredness decision approved | Submit incomplete form | Validation matches approved field requirements | F002; D06 |
| PROB-QA-021 | Update own Problem metadata | HOD-A; Problem-A | Edit title/tag/save | Allowed fields update; updatedAt changes | F003 |
| PROB-QA-022 | Update foreign Problem | HOD-A; Problem-B | PATCH foreign ID | Denied; no mutation | F003 |
| PROB-QA-023 | Manager updates assigned Problem | Manager-A | Edit scoped Problem | Allowed only under approved Manager Update policy | F003; D01; Blocked |
| PROB-QA-024 | Forge creator/ID in update | HOD-A | Update protected fields | Protected identity fields rejected/ignored | F003 |
| PROB-QA-025 | Change College/Department parent | Existing Problem links | Reparent Problem | Follow approved transfer policy; no silent scope change | F003/F004; D23; Blocked |
| PROB-QA-026 | Concurrent metadata edits | Two authorized editors | Save overlapping updates | Approved conflict policy; no silent corruption | F003; Module 18; Blocked |
| PROB-QA-027 | Edit Problem after submissions exist | Existing CodeSubmission | Update statement/limits | Historical interpretation follows approved version policy; no implicit rejudge | F003/F023; D17/D18 |
| PROB-QA-028 | Edit archived/retired Problem | Lifecycle state decided | Attempt edit | Follow approved state policy | F003/F006; D02; Blocked |
| PROB-QA-029 | Set valid College/Department pair | College A/Department A | Save | Association passes parent validation | F004 |
| PROB-QA-030 | Department belongs to different College | College A + Department B | Create/update | Reject | F004 |
| PROB-QA-031 | Manager sets foreign Department | Manager-A | Set Department B | Deny | F004/RBAC |
| PROB-QA-032 | academicBatch format | Batch format approved | Enter valid/invalid batch | Validate per approved format | F004; D09; Blocked |
| PROB-QA-033 | Problem with no College/Department | Global/null behavior unresolved | Create/edit | Blocked until null/global policy approved | F004; D04/D23 |
| PROB-QA-034 | Change batch after activity assignment | Existing assignments/submissions | Edit batch | Existing eligibility follows approved impact policy | F004; D09/D17; Blocked |
| PROB-QA-035 | Set DEPT_WIDE scope | Scope semantics approved | Save and access as HOD/Student | Visibility matches approved matrix | F005; D05; Blocked |
| PROB-QA-036 | Set SECTION_ONLY scope without Section relation | Problem schema lacks Section field | Save | Follow approved association/context design; cannot assume | F005; D05; Blocked |
| PROB-QA-037 | Set GLOBAL scope across College | College-scoped Problem | Change to GLOBAL | No cross-College access absent explicit approved rule | F005; D05/D23 |
| PROB-QA-038 | isPublic=true with tenant scope | College A Problem | Access from College B | Outcome follows approved visibility policy; no silent bypass | F005; D03/D04; Blocked |
| PROB-QA-039 | isPublic=false with GLOBAL | Global scope combination | View as other tenant | Follow approved combined semantics | F005; D03–D05; Blocked |
| PROB-QA-040 | Public Problem anonymous access | No authenticated user | Request public URL | Anonymous access not assumed; deny unless explicitly approved | F005; D03 |
| PROB-QA-041 | Student sees private Problem in Practice Problems | Student not assigned | Search/list | Excluded unless approved eligibility says otherwise | F016/F021 |
| PROB-QA-042 | Change scope with existing Course/Lab assignments | Linked activities exist | Update scope | Access changes only per approved impact policy | F005/F023; D17 |
| PROB-QA-043 | Scope change audit | Audit policy defined | Change public/scope | Module 15 records approved event/actor/target | F005/F024; D25 |
| PROB-QA-044 | Publish Problem | Lifecycle state machine approved | Request publish | Transition matches Module 17-approved rule | F006; D02; Blocked |
| PROB-QA-045 | Publish incomplete Problem | Publish checklist approved | Publish missing content/TestCases | Reject according to checklist; no state change | F006; D02; Blocked |
| PROB-QA-046 | Manager publishes Problem | Manager role | Publish action | Deny unless explicit grant | F006; D01 |
| PROB-QA-047 | Unpublish Problem with active Course/Lab | Linked activities | Unpublish | Follow approved access/retention policy | F006; D02/D17; Blocked |
| PROB-QA-048 | Generic PublishStatus enum used for Problem | Problem schema has no status field | API requests status | Reject/ignore until schema decision; no inferred field | F006; D02 |
| PROB-QA-049 | Concurrent publish/edit | Multiple editors | Publish while metadata changes | Canonical transition/concurrency policy | F006; Module 17/18; Blocked |
| PROB-QA-050 | Delete Problem with no references | Problem unused | Delete | Follow approved deletion policy | F007; D19; Blocked |
| PROB-QA-051 | Delete Problem with CourseProblem | Course link exists | Delete | No unapproved cascade | F007 |
| PROB-QA-052 | Delete Problem with submissions | CodeSubmission history exists | Delete | Preserve history per approved retention policy | F007/F023; D19 |
| PROB-QA-053 | Manager attempts delete | Manager role | Delete Problem | Deny unless separately granted | F007/RBAC |
| PROB-QA-054 | Delete Problem used in active Assessment/Contest | Active activity references | Delete | Block or approved handling; no implicit detach | F007; D16/D19 |
| PROB-QA-055 | Restore archived Problem | Archive model approved | Restore | Follow approved lifecycle/restore policy | F007; D02/D19; Blocked |
| PROB-QA-056 | Delete repeated / stale Problem ID | Record deleted or stale | Repeat delete | No unintended mutation; API behavior per policy | F007; D19 |
| PROB-QA-057 | Edit statement as authorized HOD | Problem-A | Save updated statement | Statement update persists safely | F008 |
| PROB-QA-058 | Statement includes unsupported HTML/script | Content sanitization policy | Submit/render | Safe output per approved sanitization policy | F008; D06; Blocked |
| PROB-QA-059 | Statement format Markdown/rich text | Format not approved | Save/render | Behavior remains blocked until editor format chosen | F008; D06 |
| PROB-QA-060 | Student views updated statement | Student eligible | Open Problem | Current content returned; history/version behavior remains decision-gated | F008/F023 |
| PROB-QA-061 | Statement edit during active contest | Contest active | Edit statement | Apply approved snapshot/change policy | F008/F023; D17 |
| PROB-QA-062 | Statement edit unauthorized | Student | Attempt edit | Denied | F008 |
| PROB-QA-063 | Set valid difficulty | EASY/MEDIUM/HARD | Save | Enum accepted | F009 |
| PROB-QA-064 | Set invalid difficulty | Invalid string | Save | Reject | F009 |
| PROB-QA-065 | Add tag under approved catalogue | Catalogue defined | Add valid tag | Save/normalize according to policy | F009; D08 |
| PROB-QA-066 | Duplicate tag/case variation | Tag rules approved | Add duplicate | Normalize/reject per approved rule | F009; D08; Blocked |
| PROB-QA-067 | Student filters by difficulty/tag | Practice page | Apply filter | Only eligible Problem results returned | F009/F021 |
| PROB-QA-068 | Tag editing after publication | Published state policy approved | Edit tags | Allowed/denied per approved lifecycle policy | F009; D08/D02; Blocked |
| PROB-QA-069 | Set supported language ID | Canonical catalogue approved | Save language | Accepted | F010; D10 |
| PROB-QA-070 | Unsupported language ID | Unknown language | Save | Reject | F010; D10 |
| PROB-QA-071 | Empty allowedLanguages | Empty-list semantics unresolved | Save/submit | Behavior follows approved rule; not treated as unrestricted by assumption | F010; D10; Blocked |
| PROB-QA-072 | Time limit outside approved range | Resource policy approved | Enter invalid time | Reject | F010; D11 |
| PROB-QA-073 | Memory limit outside approved range | Resource policy approved | Enter invalid memory | Reject | F010; D11 |
| PROB-QA-074 | Change limits while Judge0 request queued | Submission in progress | Update limit | Module 6/18 consistency policy; no implicit old submission rewrite | F010/F023; Blocked |
| PROB-QA-075 | Activity attempts override Problem languages | Activity configuration | Set conflicting languages | Apply approved override rule; no assumption | F010; D10 |
| PROB-QA-076 | Judge0 language mapping missing | Language catalogue | Submit/execute | Module 6 error handling; Problem config cannot silently bypass | F010; Module 6 |
| PROB-QA-077 | Create TestCase with valid parent | Problem-A editable | Add input/output | TestCase saved under Problem-A | F011 |
| PROB-QA-078 | Create TestCase for nonexistent Problem | Invalid problemId | Submit | Reject; no TestCase row | F011 |
| PROB-QA-079 | Manager creates TestCase in assigned scope | Manager-A | Add TestCase | Allowed only if permission policy grants it | F011; D01; Blocked |
| PROB-QA-080 | Manager creates TestCase for foreign Problem | Manager-A; Problem-B | Submit | Deny | F011/RBAC |
| PROB-QA-081 | Empty input TestCase | Empty-input semantics TBD | Create | Follow approved empty input policy | F011; D12; Blocked |
| PROB-QA-082 | Empty expected output | Empty-output semantics TBD | Create | Follow approved output policy | F011; D12; Blocked |
| PROB-QA-083 | Input/output exceeds size limit | Limits approved | Submit large payload | Reject before persistence | F011; D12 |
| PROB-QA-084 | TestCase creation audit | Audit policy approved | Create case | Module 15 event recorded per catalog | F011/F024; D25 |
| PROB-QA-085 | Update TestCase input/output | Existing case | Edit and save | Allowed fields update | F012 |
| PROB-QA-086 | Update TestCase under foreign Problem | Manager/HOD out of scope | Edit by ID | Deny | F012 |
| PROB-QA-087 | Change TestCase after historical submissions | CodeSubmission exists | Edit expected output | Historical verdicts not silently rewritten/rejudged | F012/F023; D18 |
| PROB-QA-088 | Edit TestCase during active contest | Contest active | Update case | Approved snapshot/version policy applied | F012/F023; D17 |
| PROB-QA-089 | Concurrent TestCase edits | Two editors | Save competing changes | Approved concurrency behavior; no silent corruption | F012; Module 18 |
| PROB-QA-090 | Move TestCase to another Problem | Existing TestCase | Change problemId | Deny ordinary edit unless transfer is approved | F012; D19 |
| PROB-QA-091 | Delete unused TestCase | No references/history | Delete | Follow approved delete policy | F013; D19; Blocked |
| PROB-QA-092 | Delete TestCase used in history | Historical submissions | Delete | No unapproved destructive removal | F013/F023 |
| PROB-QA-093 | Manager deletes TestCase | Manager role | Delete | Deny absent explicit delete permission | F013/RBAC |
| PROB-QA-094 | Delete last TestCase | Problem has one case | Delete | Follow approved minimum-case rule | F013; D12; Blocked |
| PROB-QA-095 | Delete during evaluation | Judge execution references case | Delete | Module 7/18 consistency policy; no unsafe partial mutation | F013; Blocked |
| PROB-QA-096 | Reorder TestCases within same Problem | Multiple cases | Reorder list | Stored order is deterministic | F014 |
| PROB-QA-097 | Reorder across two Problems | IDs from different Problems | Submit reorder | Reject; no partial mutation | F014 |
| PROB-QA-098 | Duplicate/negative orderIndex | Ordering rules approved | Save indices | Reject/normalize per approved policy | F014; D13; Blocked |
| PROB-QA-099 | Reorder while submission runs | Evaluation active | Reorder cases | Historical failed index/result policy respected | F014/F023; D18 |
| PROB-QA-100 | Reorder persistence after refresh | Saved order | Reload list | Same deterministic order | F014 |
| PROB-QA-101 | Student requests visible TestCase | isHidden=false | Fetch sample/test data | Fields returned according to approved visible-case policy | F015; D15; Blocked |
| PROB-QA-102 | Student requests hidden TestCase | isHidden=true | Fetch by ID/API | Hidden input/expectedOutput not exposed | F015 |
| PROB-QA-103 | Hidden TestCase appears in frontend payload | Student page | Inspect response | No hidden payload in client response | F015 |
| PROB-QA-104 | Hidden TestCase appears in error/log | Failed evaluation | Inspect client error/log | No hidden input/expected output leak | F015 |
| PROB-QA-105 | Staff views hidden TestCase | Authorized HOD | Open editor | Access follows staff role/scope | F015 |
| PROB-QA-106 | Change hidden to visible | Existing submissions | Update flag | Approved visibility/history policy applies | F015; D14/D17 |
| PROB-QA-107 | Visible TestCase evaluation semantics | Module 7 policy unresolved | Evaluate submission | Behavior follows Module 7 decision | F015; D15; Blocked |
| PROB-QA-108 | Hidden TestCase count disclosed | Student page | Inspect count/metadata | Only approved aggregate data disclosed | F015; D14; Blocked |
| PROB-QA-109 | Student Practice Problems page | Student-A | Open page | Only eligible Problems appear | F016; D21; Blocked |
| PROB-QA-110 | Practice list includes assigned Department Problem | Eligibility policy approved | Open list | Include/exclude according to approved policy | F016; D21 |
| PROB-QA-111 | Practice list includes public Problem | Public policy approved | Open list | Follow public/scope matrix | F016; D03/D04 |
| PROB-QA-112 | Practice list includes foreign College private Problem | Student-A | Search/list | No foreign private content | F016/RBAC |
| PROB-QA-113 | Practice list empty | No eligible Problems | Open page | Empty state; no false records | F016 |
| PROB-QA-114 | Student changes Section/batch | User scope updated | Reload Practice Problems | Visibility follows approved new scope | F016; D09/D17 |
| PROB-QA-115 | Student searches hidden Problem title | Problem not visible | Search | No hidden content/snippet leakage | F016/F021 |
| PROB-QA-116 | Student direct URL bypasses list | Problem absent from list | Request detail | Same access rules; deny if ineligible | F016 |
| PROB-QA-117 | CourseProblem valid association | Student eligible in Module 4 | Open Course Problem | Problem content returned subject to Problem policy | F017 |
| PROB-QA-118 | Course view bypasses Problem scope | CourseProblem exists but Problem restricted | Open Problem | Problem restrictions still enforced | F017 |
| PROB-QA-119 | Course enrollment removed | Student no longer eligible | Direct Problem route | Module 4 eligibility policy applied | F017 |
| PROB-QA-120 | CourseProblem and GroupCourseProblem confusion | Group-only relation exists | Open general Course | Group-only Problem not treated as course-wide | F017 |
| PROB-QA-121 | Course access grants submission permission | Student only has Course view | Submit code | Module 6 permission independently checked | F017 |
| PROB-QA-122 | Problem archived while Course linked | Archive policy unresolved | Open Course Problem | Follow approved lifecycle/retention rule | F017; D02/D19; Blocked |
| PROB-QA-123 | Lab assignment grants permitted Problem access | Student eligible in Module 8 | Open Lab Problem | Both Lab and Problem access pass | F018 |
| PROB-QA-124 | Student not Lab participant | Nonmember/ineligible | Direct Problem ID | Deny via Lab eligibility | F018 |
| PROB-QA-125 | Lab assignment makes Problem public | Private Problem assigned to Lab | Open from Practice | Not public unless visibility rule says so | F018 |
| PROB-QA-126 | Lab assignment references unavailable Problem | Stale relation | Open Lab | Safe error/deny; no crash or leak | F018 |
| PROB-QA-127 | Lab access bypasses hidden cases | Student Lab attempt | Fetch hidden data | Hidden values withheld | F018/F015 |
| PROB-QA-128 | Assessment participant opens assigned Problem | Eligible attempt | Open assessment Problem | Problem returned after Assessment + Problem authorization | F019 |
| PROB-QA-129 | Assessment outside time window | Module 9 says not eligible | Open Problem | Assessment attempt access denied | F019 |
| PROB-QA-130 | AssessmentProblem missing | Invalid relation | Request Problem | Deny/not-found | F019 |
| PROB-QA-131 | Assessment access bypasses Problem scope | Problem restricted | Open from assessment | Problem access policy still enforced | F019 |
| PROB-QA-132 | Problem changes during Assessment | Active attempt | Editor updates Problem | Approved snapshot/version policy applied | F019/F023; Blocked |
| PROB-QA-133 | Contest participant opens Problem | Eligible contest participant | Open contest Problem | Contest + Problem access pass | F020 |
| PROB-QA-134 | Nonparticipant guesses Problem ID | Contest private | Direct URL | Deny; no bypass of Contest access | F020 |
| PROB-QA-135 | Contest ended | Contest state ended | Open Problem | Module 11 contest access policy applies | F020 |
| PROB-QA-136 | ContestProblem references foreign Problem | Cross-tenant relation | Open contest | Reject unless approved cross-tenant association | F020; D23 |
| PROB-QA-137 | Problem edit during live contest | Contest active | Update statement/TestCase | Approved snapshot policy; no silent change | F020/F023; D17 |
| PROB-QA-138 | Search by title | Authorized Problem list | Search title | Matching authorized results | F021; D20 |
| PROB-QA-139 | Filter by difficulty/tag | Filter catalogue approved | Apply filters | Results satisfy filter and scope | F021 |
| PROB-QA-140 | Sort/paginate | List API | Request page/sort | Stable approved order/page behavior | F021; D20 |
| PROB-QA-141 | Search foreign Problem | HOD-A; Problem-B | Search unique title | No foreign result/snippet | F021/RBAC |
| PROB-QA-142 | Search hidden Problem content | Student unauthorized | Query statement phrase | No restricted content disclosure | F021 |
| PROB-QA-143 | Invalid filter/sort | Unsupported value | Submit query | Reject/normalize per API contract | F021; TBD |
| PROB-QA-144 | Client passes foreign scope filter | Manager-A | Filter College B | Server scope remains authoritative | F021/RBAC |
| PROB-QA-145 | Pagination page size abuse | Query contract defined | Request excessive page size | Enforce approved limit | F021; D20 |
| PROB-QA-146 | Student views own Problem submissions | Student-A own CodeSubmission | Open history | Own records returned per Module 6 | F022 |
| PROB-QA-147 | Student views other Student submission | Foreign studentId | Request submission | Deny | F022/RBAC |
| PROB-QA-148 | HOD views submissions in Department | HOD-A; submissions in A/B | Open history | Department-scoped results only | F022 |
| PROB-QA-149 | Manager views submission from another Section | Manager-A | Request foreign submission | Deny | F022/RBAC |
| PROB-QA-150 | Problem view alone grants code access | User lacks submission permission | Request source code | Module 6 permission checked independently | F022 |
| PROB-QA-151 | Finance Team views Problem submissions | Finance role | Request history | Deny/unavailable unless explicit grant | F022; D01/D22 |
| PROB-QA-152 | Problem edit preserves historical verdict | Existing submission | Update Problem | Existing submission/verdict not silently rewritten | F023 |
| PROB-QA-153 | TestCase expectedOutput changes | Existing judged submission | Edit TestCase | No automatic rejudge absent approved policy | F023 |
| PROB-QA-154 | TestCase order changes failed index meaning | Historical failedTestCaseIndex | Reorder cases | Old result remains traceable under approved version policy | F023 |
| PROB-QA-155 | Problem deleted with historical submissions | History exists | Delete/archive | Historical records preserved per approved retention | F023 |
| PROB-QA-156 | Edit while Judge0 job running | Pending submission | Update limits/cases | Module 6/18 race policy applied | F023; Blocked |
| PROB-QA-157 | Concurrent Problem/TestCase changes | Multiple editors | Save concurrently | Approved consistency policy; no partial corruption | F023; Module 18 |
| PROB-QA-158 | Rejudge after content edit | Rejudge capability not confirmed | Trigger rejudge | No rejudge unless explicitly approved and Module 7 defines it | F023; D18 |
| PROB-QA-159 | Problem create/update audit | Audit event catalog approved | Mutate Problem | Module 15 records actor/action/target/scope | F024; D25 |
| PROB-QA-160 | Hidden TestCase audit payload | TestCase has private values | Create/update | Routine audit excludes hidden input/output unless approved | F024 |
| PROB-QA-161 | Problem publish notification | Notification policy unresolved | Publish | No notification until Module 13 trigger/recipient approved | F024; D24 |
| PROB-QA-162 | Notification delivery fails | Notification enabled by decision | Trigger event | Module 13/18 retry/failure policy applies | F024; Blocked |
| PROB-QA-163 | Audit service fails | Audit required | Mutate Problem | Module 15/18 failure policy; no silent untracked sensitive action | F024; Blocked |
| PROB-QA-164 | Backend checks role permission | User lacks Problem action | API mutation | Deny before DB write | F025/RBAC |
| PROB-QA-165 | College isolation | College A actor requests College B Problem | API/list/detail | Deny/no leakage | F025/RBAC |
| PROB-QA-166 | Department isolation | HOD-A requests Department B | API/list/detail | Deny/no leakage | F025/RBAC |
| PROB-QA-167 | Section/group isolation | Manager-A requests unrelated group/Section | API/list/detail | Deny/no leakage | F025/RBAC |
| PROB-QA-168 | Student ownership isolation | Student-A requests Student-B private resource | API | Deny | F025/RBAC |
| PROB-QA-169 | Frontend hides action but API direct call | UI hides edit | Call update endpoint directly | Backend denies if permission missing | F025/RBAC |
| PROB-QA-170 | Scope changes during request | concurrent organization/Problem scope update | Request Problem | Module 18 consistency policy; no stale scope bypass | F025; Blocked |

## QA review notes
- PROB-QA-001–170 map to PROB-F001–F025 and the source decision log.
- **Blocked** tests require decision approval; they are not evidence that a feature exists.
- This QA document does not resolve PROB-D01–D25 or approve implementation behavior.
