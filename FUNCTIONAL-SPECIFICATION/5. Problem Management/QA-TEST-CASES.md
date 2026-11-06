# Module 5 — Problem Management QA Test Cases

**Version:** 1.0 — Adopted target baseline QA companion
**Status:** 200 target cases; implementation execution not yet performed
**Spec:** [Problem Management](./README.md)

## Execution conventions
- Test identities: SUPER_ADMIN; Finance-A (College A); HOD-A (College A/Department A); Manager-A/Section A1; Student-A; HOD-B/Department B; Student-B.
- Test records: Problem-A (College A/Department A), Problem-B (College B/Department B), TestCase-A1, Course/Lab/Assessment/Contest association records, and CodeSubmission records.
Product behavior is governed by README Sections 25–27 and ADOPTED-DECISIONS.md. These are target expectations, not evidence that implementation has passed.
- Hidden TestCase input/expected output, ID, exact count and ordering must not be exposed to Student payloads.

| Test ID | Scenario | Preconditions / data | Steps | Expected result | Trace/status |
|---|---|---|---|---|---|
| PROB-QA-001 | HOD lists own Department Problems | HOD-A; Problem-A/B | Open Problems page | Problem-A appears; Problem-B excluded | F001/RBAC |
| PROB-QA-002 | HOD requests foreign Problem by ID | HOD-A; Problem-B | Direct API/URL | Deny/not-found; no data leak | F001 |
| PROB-QA-003 | Manager lists assigned Section Problems | Manager-A; Section-scoped content | Open Problems | Only permitted assigned activity/Section content | F001; scope policy TBD |
| PROB-QA-004 | Manager accesses unrelated Section Problem | Manager-A; foreign Section | Direct ID request | Deny; no content disclosed | F001/RBAC |
| PROB-QA-005 | Student accesses permitted Course Problem | Student-A enrolled/eligible | Open Course Problem | Problem statement/safe fields returned | F001/F017 |
| PROB-QA-006 | Student accesses unassigned private Problem | Student-A; Problem outside allowed context | Direct URL/API | Deny/not-found | F001 |
| PROB-QA-007 | SUPER_ADMIN views cross-College Problem | SUPER_ADMIN; Problem-A/B | Search/detail | Global role access follows RBAC | F001 |
| PROB-QA-008 | Finance Team Problem access | Finance-A in College A | Open Problem page or call Problem API | Denied; Finance has no Problem page/API access in v1 | PROB-F001; PROB-D01 |
| PROB-QA-009 | Client sends foreign College filter | HOD-A | Query College B | Server-side scope remains College A | F001/F021 |
| PROB-QA-010 | Student requests hidden TestCase with Problem detail | Hidden TestCase exists | Request Problem payload | Hidden input/expectedOutput absent | F001/F015 |
| PROB-QA-011 | HOD creates valid Problem in own Department | HOD-A; valid fields | Submit create | Problem saved; creator is HOD-A | F002 |
| PROB-QA-012 | HOD creates Problem in foreign Department | HOD-A; Department B | Submit create | Denied; no record created | F002 |
| PROB-QA-013 | SUPER_ADMIN creates tenant-scoped Problem | SUPER_ADMIN | Submit valid College/Department | Allowed if required fields valid | F002 |
| PROB-QA-014 | Manager creates Problem in assigned Section | Manager-A assigned Section A1; all required fields valid | Create SECTION_ONLY Problem and link Section A1 | Allowed; creator/College/Department server-derived; no publish permission | PROB-F002; PROB-D01/PROB-D05 |
| PROB-QA-015 | Manager creates Problem outside assigned scope | Manager-A; foreign Section/Department | Submit | Denied | F002/RBAC |
| PROB-QA-016 | Forged createdById | HOD-A; payload names another user | Create Problem | Server derives creator from authenticated actor or rejects field | F002 |
| PROB-QA-017 | Title exceeds 255 characters | Create form/API | Submit >255 chars | Reject; no Problem row | F002/schema |
| PROB-QA-018 | Invalid difficulty enum | difficulty=unsupported | Submit create | Reject | F002 |
| PROB-QA-019 | Invalid ContentScope | scope=unsupported | Submit create | Reject | F002/F005 |
| PROB-QA-020 | Required Problem field validation | Create payload missing title, statement, difficulty or owning org | Submit incomplete Problem | Rejected; title, statement, difficulty, College and owning Department required | PROB-F002; PROB-D06/PROB-D09 |
| PROB-QA-021 | Update own Problem metadata | HOD-A; Problem-A | Edit title/tag/save | Allowed fields update; updatedAt changes | F003 |
| PROB-QA-022 | Update foreign Problem | HOD-A; Problem-B | PATCH foreign ID | Denied; no mutation | F003 |
| PROB-QA-023 | Manager updates assigned Problem | Manager-A owns SECTION_ONLY Problem A1 | Update permitted metadata in DRAFT | Allowed only within assigned Section; College/Department/scope/status protected | PROB-F003; PROB-D01/PROB-D05 |
| PROB-QA-024 | Forge creator/ID in update | HOD-A | Update protected fields | Protected identity fields rejected/ignored | F003 |
| PROB-QA-025 | Reparent Problem | Existing Problem has Course/Lab/submission references | Change College or owning Department | Rejected; published Problem cannot be reparented; create a new Problem and reassign explicitly | PROB-F003/F004; PROB-D04/PROB-D17/PROB-D23 |
| PROB-QA-026 | Concurrent metadata edits | Two authorized editors load same revision | Submit conflicting updates | One update succeeds; stale revision receives conflict; no lost update or partial write | PROB-F003; PROB-D17/PROB-D25 |
| PROB-QA-027 | Edit Problem after submissions exist | Published version has CodeSubmissions | Edit statement/limits and publish a new revision | New ProblemVersion created; existing submissions remain pinned to old version; no automatic rejudge | PROB-F003/F023; PROB-D17/PROB-D18 |
| PROB-QA-028 | Edit archived Problem | Problem status ARCHIVED | Attempt metadata edit | Denied until authorized restore to DRAFT; archive remains intact | PROB-F003/F006/F007; PROB-D02/PROB-D19 |
| PROB-QA-029 | Set valid College/Department pair | College A/Department A | Save | Association passes parent validation | F004 |
| PROB-QA-030 | Department belongs to different College | College A + Department B | Create/update | Reject | F004 |
| PROB-QA-031 | Manager sets foreign Department | Manager-A | Set Department B | Deny | F004/RBAC |
| PROB-QA-032 | academicBatch format | academicBatch is optional text | Save trimmed batch identifier and an overlength value | Accept valid trimmed identifier within field size; reject invalid length; null means no batch restriction | PROB-F004; PROB-D09 |
| PROB-QA-033 | Problem without College/Department | Create request omits College or owning Department | Submit create | Rejected; tenant-owned Problem requires College and owning Department | PROB-F004; PROB-D04/PROB-D23 |
| PROB-QA-034 | Change batch after activity assignment | Existing assignments/submissions | Edit batch | Existing eligibility follows approved impact policy | F004; PROB-D09/PROB-D17 |
| PROB-QA-035 | Set DEPT_WIDE scope | HOD-A in Department A with active Sections | Set DEPT_WIDE and publish | Visible to eligible active Students in owning Department only; other Departments and Colleges excluded | PROB-F005; PROB-D04/PROB-D05/PROB-D23 |
| PROB-QA-036 | Set SECTION_ONLY scope | ProblemSection relation implemented | Set SECTION_ONLY and omit ProblemSection links, then add valid Section link | Publish denied without valid link; after link, only active Students in linked Section eligible | PROB-F005; PROB-D05 |
| PROB-QA-037 | Set GLOBAL scope | College A Problem; HOD-A and SUPER_ADMIN available | HOD attempts GLOBAL; SUPER_ADMIN sets GLOBAL | HOD denied; SUPER_ADMIN may expose to active Sections across College A only; College B excluded | PROB-F005; PROB-D05/PROB-D23 |
| PROB-QA-038 | isPublic=true with tenant scope | Problem P-A published; isPublic true; College A | Student from College B requests Practice list/detail | No College B access; isPublic never bypasses College or ContentScope | PROB-F005/F016; PROB-D03/PROB-D04/PROB-D23 |
| PROB-QA-039 | isPublic=false with GLOBAL | Published Problem in College A; isPublic false; GLOBAL | Student A opens Practice; eligible activity participant opens linked activity | Absent from Practice discovery; eligible same-College activity participant may access through valid link | PROB-F005/F016/F017; PROB-D03/PROB-D04/PROB-D16 |
| PROB-QA-040 | Anonymous Problem access | No authenticated user | Request Problem catalog/detail URL | Denied; no anonymous Internet publication in v1 | PROB-F005/F016; PROB-D03 |
| PROB-QA-041 | Student sees private Problem in Practice Problems | Student not assigned | Search/list | Excluded unless approved eligibility says otherwise | F016/F021 |
| PROB-QA-042 | Change scope with existing Course/Lab assignments | Linked activities exist | Update scope | Access changes only per approved impact policy | F005/F023; PROB-D17 |
| PROB-QA-043 | Scope change audit | Audit policy defined | Change public/scope | Module 15 records approved event/actor/target | F005/F024; PROB-D25 |
| PROB-QA-044 | Publish valid Problem | DRAFT Problem has required fields, language, resource limits and at least one TestCase | Request publish | Allowed; creates immutable ProblemVersion and sets PUBLISHED | PROB-F006; PROB-D02/PROB-D06/PROB-D10/PROB-D11/PROB-D12/PROB-D17 |
| PROB-QA-045 | Publish incomplete Problem | DRAFT Problem missing required field or TestCase | Request publish | Rejected with validation details; remains DRAFT and creates no version | PROB-F006; PROB-D02/PROB-D06/PROB-D10/PROB-D11/PROB-D12 |
| PROB-QA-046 | Manager publishes Problem | Manager-A manages SECTION_ONLY Problem | Call publish endpoint | Denied; only HOD/SUPER_ADMIN may publish | PROB-F006; PROB-D01/PROB-D02 |
| PROB-QA-047 | Unpublish Problem with active Course/Lab | Linked activities | Unpublish | Follow approved access/retention policy | F006; PROB-D02/PROB-D17 |
| PROB-QA-048 | Problem lifecycle field | Target schema migration is applied | Create Problem without explicit status | Problem.status defaults DRAFT; shared PublishStatus values used; status cannot be inferred from isPublic | PROB-F006; PROB-D02 |
| PROB-QA-049 | Concurrent publish/edit | Multiple editors | Publish while metadata changes | Canonical transition/concurrency policy | F006; Module 17/18 |
| PROB-QA-050 | Archive Problem with no references | PUBLISHED unused Problem | HOD archives | Status ARCHIVED; content retained; no hard delete | PROB-F007; PROB-D02/PROB-D19 |
| PROB-QA-051 | Delete Problem with CourseProblem | Course link exists | Delete | No unapproved cascade | F007 |
| PROB-QA-052 | Delete Problem with submissions | CodeSubmission history exists | Delete | Preserve history per approved retention policy | F007/F023; PROB-D19 |
| PROB-QA-053 | Manager attempts delete | Manager role | Delete Problem | Deny unless separately granted | F007/RBAC |
| PROB-QA-054 | Delete Problem used in active activity | Problem referenced by Course/Lab/Assessment/Contest | Attempt hard delete | No hard-delete operation exists; archive only; associations/history are not cascaded | PROB-F007; PROB-D19 |
| PROB-QA-055 | Restore archived Problem | Problem status ARCHIVED; HOD in scope | Restore Problem | Transitions to DRAFT; not automatically public or PUBLISHED; republish creates a new version | PROB-F007; PROB-D02/PROB-D17/PROB-D19 |
| PROB-QA-056 | Delete repeated / stale Problem ID | Record deleted or stale | Repeat delete | No unintended mutation; API behavior per policy | F007; PROB-D19 |
| PROB-QA-057 | Edit statement as authorized HOD | Problem-A | Save updated statement | Statement update persists safely | F008 |
| PROB-QA-058 | Statement includes unsafe HTML/script | Problem statement submitted in Markdown | Submit script/event-handler/unsafe HTML and render | Unsafe markup is stripped/rejected; safe Markdown remains; no script execution | PROB-F008; PROB-D06 |
| PROB-QA-059 | Statement format | Problem statement text | Save Markdown headings, lists, code fences and math-safe text | Markdown stored as text and rendered through allowlisted safe renderer; no raw executable HTML | PROB-F008; PROB-D06 |
| PROB-QA-060 | Student views updated statement | Student eligible | Open Problem | Current content returned; history/version behavior remains decision-gated | F008/F023 |
| PROB-QA-061 | Statement edit during active contest | Contest active | Edit statement | Apply approved snapshot/change policy | F008/F023; PROB-D17 |
| PROB-QA-062 | Statement edit unauthorized | Student | Attempt edit | Denied | F008 |
| PROB-QA-063 | Set valid difficulty | EASY/MEDIUM/HARD | Save | Enum accepted | F009 |
| PROB-QA-064 | Set invalid difficulty | Invalid string | Save | Reject | F009 |
| PROB-QA-065 | Add free-form tag | Problem draft | Add valid tag with whitespace and casing | Tag trimmed/lowercase normalized and stored; no global tag catalogue required | PROB-F009; PROB-D08 |
| PROB-QA-066 | Duplicate tag/case variation | Problem contains tag graphs | Add Graphs and duplicate whitespace variant | Stored once per Problem after case-insensitive normalization | PROB-F009; PROB-D08 |
| PROB-QA-067 | Student filters by difficulty/tag | Practice page | Apply filter | Only eligible Problem results returned | F009/F021 |
| PROB-QA-068 | Tag editing after publication | Published state policy approved | Edit tags | Allowed/denied per approved lifecycle policy | F009; PROB-D08/PROB-D02 |
| PROB-QA-069 | Set supported language ID | Canonical catalogue approved | Save language | Accepted | F010; PROB-D10 |
| PROB-QA-070 | Unsupported language ID | Unknown language | Save | Reject | F010; PROB-D10 |
| PROB-QA-071 | Empty allowedLanguages | DRAFT Problem has empty language list | Attempt publish or submit | Publish rejected; empty list never means all languages | PROB-F010; PROB-D10 |
| PROB-QA-072 | Time limit outside approved range | Resource policy approved | Enter invalid time | Reject | F010; PROB-D11 |
| PROB-QA-073 | Memory limit outside approved range | Resource policy approved | Enter invalid memory | Reject | F010; PROB-D11 |
| PROB-QA-074 | Change limits while Judge0 request queued | Submission accepted and pinned to version 1 | Publish version 2 with different limits while version 1 job is queued | Queued submission runs with version 1 limits; no retroactive mutation | PROB-F010/F023; PROB-D11/PROB-D17/PROB-D18 |
| PROB-QA-075 | Activity attempts broaden allowed languages | Problem allows Python only; activity offers Python and Java | Student selects Java | Rejected; activity may narrow but cannot broaden Problem whitelist | PROB-F010; PROB-D10 |
| PROB-QA-076 | Judge0 language mapping missing | Language catalogue | Submit/execute | Module 6 error handling; Problem config cannot silently bypass | F010; Module 6 |
| PROB-QA-077 | Create TestCase with valid parent | Problem-A editable | Add input/output | TestCase saved under Problem-A | F011 |
| PROB-QA-078 | Create TestCase for nonexistent Problem | Invalid problemId | Submit | Reject; no TestCase row | F011 |
| PROB-QA-079 | Manager creates TestCase in assigned Problem | Manager-A owns SECTION_ONLY Problem in assigned Section | Add TestCase to DRAFT revision | Allowed within Section scope; cannot edit a published version | PROB-F011; PROB-D01/PROB-D12/PROB-D17 |
| PROB-QA-080 | Manager creates TestCase for foreign Problem | Manager-A; Problem-B | Submit | Deny | F011/RBAC |
| PROB-QA-081 | Empty input TestCase | Editable DRAFT revision | Create TestCase with input empty string | Accepted; empty input is a valid test condition | PROB-F011; PROB-D12 |
| PROB-QA-082 | Empty expected output | Editable DRAFT revision | Create TestCase with expectedOutput empty string | Accepted; empty output is a valid expected result | PROB-F011; PROB-D12 |
| PROB-QA-083 | Input/output exceeds size limit | Limits approved | Submit large payload | Reject before persistence | F011; PROB-D12 |
| PROB-QA-084 | TestCase creation audit | Audit policy approved | Create case | Module 15 event recorded per catalog | F011/F024; PROB-D25 |
| PROB-QA-085 | Update TestCase input/output | Existing case | Edit and save | Allowed fields update | F012 |
| PROB-QA-086 | Update TestCase under foreign Problem | Manager/HOD out of scope | Edit by ID | Deny | F012 |
| PROB-QA-087 | Change TestCase after historical submissions | ProblemVersion 1 has submissions | Modify TestCase in draft revision and publish Version 2 | Version 1 TestCase input/output/order remain immutable; old verdict unchanged | PROB-F012/F023; PROB-D17/PROB-D18 |
| PROB-QA-088 | Edit TestCase during active contest | Contest active | Update case | Approved snapshot/version policy applied | F012/F023; PROB-D17 |
| PROB-QA-089 | Concurrent TestCase edits | Two editors | Save competing changes | Approved concurrency behavior; no silent corruption | F012; Module 18 |
| PROB-QA-090 | Move TestCase to another Problem | Existing TestCase | Change problemId | Deny ordinary edit unless transfer is approved | F012; PROB-D19 |
| PROB-QA-091 | Remove unused TestCase from draft | DRAFT revision with multiple TestCases | Remove one TestCase | Allowed in draft; published snapshots and past submissions remain immutable | PROB-F013; PROB-D12/PROB-D17/PROB-D19 |
| PROB-QA-092 | Delete TestCase used in history | Historical submissions | Delete | No unapproved destructive removal | F013/F023 |
| PROB-QA-093 | Manager deletes TestCase | Manager role | Delete | Deny absent explicit delete permission | F013/RBAC |
| PROB-QA-094 | Remove last TestCase | DRAFT Problem has one TestCase | Remove it then attempt publish | Removal allowed in draft; publication rejected until at least one valid TestCase exists | PROB-F013/F006; PROB-D12 |
| PROB-QA-095 | Delete during evaluation | Judge execution references case | Delete | Module 7/18 consistency policy; no unsafe partial mutation | F013 |
| PROB-QA-096 | Reorder TestCases within same Problem | Multiple cases | Reorder list | Stored order is deterministic | F014 |
| PROB-QA-097 | Reorder across two Problems | IDs from different Problems | Submit reorder | Reject; no partial mutation | F014 |
| PROB-QA-098 | Duplicate/negative orderIndex | DRAFT Problem has multiple TestCases | Submit duplicate, negative or gapped order indices | Reject or atomically normalize to contiguous zero-based order; no partial reorder | PROB-F014; PROB-D13/PROB-D25 |
| PROB-QA-099 | Reorder while submission runs | Evaluation active | Reorder cases | Historical failed index/result policy respected | F014/F023; PROB-D18 |
| PROB-QA-100 | Reorder persistence after refresh | Saved order | Reload list | Same deterministic order | F014 |
| PROB-QA-101 | Student requests visible TestCase | isHidden=false TestCase in published Problem | Fetch safe Problem detail | Visible sample input/expectedOutput returned; no hidden data or metadata included | PROB-F015; PROB-D14/PROB-D15 |
| PROB-QA-102 | Student requests hidden TestCase | isHidden=true | Fetch by ID/API | Hidden input/expectedOutput not exposed | F015 |
| PROB-QA-103 | Hidden TestCase appears in frontend payload | Student page | Inspect response | No hidden payload in client response | F015 |
| PROB-QA-104 | Hidden TestCase appears in error/log | Failed evaluation | Inspect client error/log | No hidden input/expected output leak | F015 |
| PROB-QA-105 | Staff views hidden TestCase | Authorized HOD | Open editor | Access follows staff role/scope | F015 |
| PROB-QA-106 | Change hidden to visible | Existing submissions | Update flag | Approved visibility/history policy applies | F015; PROB-D14/PROB-D17 |
| PROB-QA-107 | Visible TestCase evaluation semantics | Published Problem has visible and hidden cases | Submit eligible solution | Both visible and hidden cases are included in formal evaluation; scoring/verdict aggregation belongs to Module 7 | PROB-F015; PROB-D15 |
| PROB-QA-108 | Hidden TestCase count disclosed | Problem contains hidden cases | Inspect Student payload and UI | Exact hidden count, IDs, input/output and ordering are omitted | PROB-F015; PROB-D14 |
| PROB-QA-109 | Student Practice Problems page | Student-A with current active membership | Open Practice Problems | Only PUBLISHED isPublic=true Problems within College, scope, Section and batch eligibility appear | PROB-F016; PROB-D03/PROB-D04/PROB-D05/PROB-D21 |
| PROB-QA-110 | Practice excludes non-public assigned Problem | Student eligible in Course/Lab; Problem isPublic=false | Open Practice and open explicit activity link | Absent from Practice; accessible only through valid activity route | PROB-F016/F017/F018; PROB-D03/PROB-D16/PROB-D21 |
| PROB-QA-111 | Practice includes public Problem within scope | Student-A matches College/Department/Section/batch | Open Practice Problems | PUBLISHED isPublic=true Problem appears within scope only | PROB-F016; PROB-D03/PROB-D04/PROB-D21 |
| PROB-QA-112 | Practice list includes foreign College private Problem | Student-A | Search/list | No foreign private content | F016/RBAC |
| PROB-QA-113 | Practice list empty | No eligible Problems | Open page | Empty state; no false records | F016 |
| PROB-QA-114 | Student changes Section/batch | Student membership changes to another Section or batch | Reload Practice Problems and direct URL | Recompute current scope; Problems outside new eligibility disappear and direct access is denied | PROB-F016; PROB-D04/PROB-D09/PROB-D21 |
| PROB-QA-115 | Student searches hidden Problem title | Problem not visible | Search | No hidden content/snippet leakage | F016/F021 |
| PROB-QA-116 | Student direct URL bypasses list | Problem absent from list | Request detail | Same access rules; deny if ineligible | F016 |
| PROB-QA-117 | CourseProblem valid association | Student eligible in Module 4 | Open Course Problem | Problem content returned subject to Problem policy | F017 |
| PROB-QA-118 | Course view bypasses Problem scope | CourseProblem exists but Problem restricted | Open Problem | Problem restrictions still enforced | F017 |
| PROB-QA-119 | Course enrollment removed | Student no longer eligible | Direct Problem route | Module 4 eligibility policy applied | F017 |
| PROB-QA-120 | CourseProblem and GroupCourseProblem confusion | Group-only relation exists | Open general Course | Group-only Problem not treated as course-wide | F017 |
| PROB-QA-121 | Course access grants submission permission | Student only has Course view | Submit code | Module 6 permission independently checked | F017 |
| PROB-QA-122 | Problem archived while Course linked | Archived Problem remains referenced by Course | Student opens Course Problem route | No new activity access/submission; historical result remains read-only under Module 4/6 policy | PROB-F017/F023; PROB-D16/PROB-D19 |
| PROB-QA-123 | Lab assignment grants permitted Problem access | Student eligible in Module 8 | Open Lab Problem | Both Lab and Problem access pass | F018 |
| PROB-QA-124 | Student not Lab participant | Nonmember/ineligible | Direct Problem ID | Deny via Lab eligibility | F018 |
| PROB-QA-125 | Lab assignment makes Problem public | Private Problem assigned to Lab | Open from Practice | Not public unless visibility rule says so | F018 |
| PROB-QA-126 | Lab assignment references unavailable Problem | Stale relation | Open Lab | Safe error/deny; no crash or leak | F018 |
| PROB-QA-127 | Lab access bypasses hidden cases | Student Lab attempt | Fetch hidden data | Hidden values withheld | F018/F015 |
| PROB-QA-128 | Assessment participant opens assigned Problem | Eligible attempt | Open assessment Problem | Problem returned after Assessment + Problem authorization | F019 |
| PROB-QA-129 | Assessment outside time window | Module 9 says not eligible | Open Problem | Assessment attempt access denied | F019 |
| PROB-QA-130 | AssessmentProblem missing | Invalid relation | Request Problem | Deny/not-found | F019 |
| PROB-QA-131 | Assessment access bypasses Problem scope | Problem restricted | Open from assessment | Problem access policy still enforced | F019 |
| PROB-QA-132 | Problem changes during Assessment | Assessment attempt pinned to ProblemVersion 1 | Publish ProblemVersion 2 mid-attempt | Active attempt continues against Version 1; new attempts may use Version 2 after Module 9 rebind | PROB-F019/F023; PROB-D17/PROB-D18 |
| PROB-QA-133 | Contest participant opens Problem | Eligible contest participant | Open contest Problem | Contest + Problem access pass | F020 |
| PROB-QA-134 | Nonparticipant guesses Problem ID | Contest private | Direct URL | Deny; no bypass of Contest access | F020 |
| PROB-QA-135 | Contest ended | Contest state ended | Open Problem | Module 11 contest access policy applies | F020 |
| PROB-QA-136 | ContestProblem references foreign Problem | Contest College A; Problem College B | Create or open foreign association | Rejected; no cross-College sharing in v1 | PROB-F020; PROB-D23 |
| PROB-QA-137 | Problem edit during live contest | Contest uses immutable ProblemVersion 1 | Publish revised ProblemVersion 2 | Live contest remains pinned to Version 1; no silent content change | PROB-F020/F023; PROB-D17/PROB-D18 |
| PROB-QA-138 | Search by title | Authorized Problem list | Search title | Matching authorized results | F021; PROB-D20 |
| PROB-QA-139 | Filter by difficulty/tag | Filter catalogue approved | Apply filters | Results satisfy filter and scope | F021 |
| PROB-QA-140 | Sort/paginate | List API | Request page/sort | Stable approved order/page behavior | F021; PROB-D20 |
| PROB-QA-141 | Search foreign Problem | HOD-A; Problem-B | Search unique title | No foreign result/snippet | F021/RBAC |
| PROB-QA-142 | Search hidden Problem content | Student unauthorized | Query statement phrase | No restricted content disclosure | F021 |
| PROB-QA-143 | Invalid filter/sort | Problem list API | Submit unknown filters, unsupported sort or invalid pagination | Reject invalid query values; preserve server-side scope | PROB-F021; PROB-D20 |
| PROB-QA-144 | Client passes foreign scope filter | Manager-A | Filter College B | Server scope remains authoritative | F021/RBAC |
| PROB-QA-145 | Pagination page size abuse | Query contract defined | Request excessive page size | Enforce approved limit | F021; PROB-D20 |
| PROB-QA-146 | Student views own Problem submissions | Student-A own CodeSubmission | Open history | Own records returned per Module 6 | F022 |
| PROB-QA-147 | Student views other Student submission | Foreign studentId | Request submission | Deny | F022/RBAC |
| PROB-QA-148 | HOD views submissions in Department | HOD-A; submissions in A/B | Open history | Department-scoped results only | F022 |
| PROB-QA-149 | Manager views submission from another Section | Manager-A | Request foreign submission | Deny | F022/RBAC |
| PROB-QA-150 | Problem view alone grants code access | User lacks submission permission | Request source code | Module 6 permission checked independently | F022 |
| PROB-QA-151 | Finance Team views Problem submissions | Finance role | Request Problem-linked submission history | Denied; Finance has no Problem/submission access | PROB-F022; PROB-D01/PROB-D22 |
| PROB-QA-152 | Problem edit preserves historical verdict | Existing submission | Update Problem | Existing submission/verdict not silently rewritten | F023 |
| PROB-QA-153 | TestCase expectedOutput changes | Existing judged submission | Edit TestCase | No automatic rejudge absent approved policy | F023 |
| PROB-QA-154 | TestCase order changes failed index meaning | Historical failedTestCaseIndex | Reorder cases | Old result remains traceable under approved version policy | F023 |
| PROB-QA-155 | Problem deleted with historical submissions | History exists | Delete/archive | Historical records preserved per approved retention | F023 |
| PROB-QA-156 | Edit while Judge0 job running | Submission pinned to ProblemVersion 1 | Publish version 2 while Judge0 job is queued/running | Worker continues using immutable Version 1 snapshot and limits | PROB-F023; PROB-D17/PROB-D18 |
| PROB-QA-157 | Concurrent Problem/TestCase changes | Multiple editors | Save concurrently | Approved consistency policy; no partial corruption | F023; Module 18 |
| PROB-QA-158 | Rejudge after content edit | Old submission uses Version 1; Version 2 published | No explicit rejudge requested | No automatic rejudge; explicit rejudge is separately owned/audited by Module 7 | PROB-F023; PROB-D18 |
| PROB-QA-159 | Problem create/update audit | Audit event catalog approved | Mutate Problem | Module 15 records actor/action/target/scope | F024; PROB-D25 |
| PROB-QA-160 | Hidden TestCase audit payload | TestCase has private values | Create/update | Routine audit excludes hidden input/output unless approved | F024 |
| PROB-QA-161 | Problem publish notification | Module 13 event integration active | Publish a new version | Emit ProblemPublished/ProblemVersionPublished event; Module 13 resolves recipients/channel | PROB-F024; PROB-D24 |
| PROB-QA-162 | Notification delivery fails | Module 13 accepts event | Simulate provider failure after publish | Problem publish remains committed; Module 13 retries delivery per its contract | PROB-F024; PROB-D24 |
| PROB-QA-163 | Audit service fails | Module 15 durable audit/outbox configured | Attempt sensitive scope/lifecycle/TestCase mutation during audit outage | Mutation does not succeed without durable audit record or approved outbox acceptance | PROB-F024; PROB-D25 |
| PROB-QA-164 | Backend checks role permission | User lacks Problem action | API mutation | Deny before DB write | F025/RBAC |
| PROB-QA-165 | College isolation | College A actor requests College B Problem | API/list/detail | Deny/no leakage | F025/RBAC |
| PROB-QA-166 | Department isolation | HOD-A requests Department B | API/list/detail | Deny/no leakage | F025/RBAC |
| PROB-QA-167 | Section/group isolation | Manager-A requests unrelated group/Section | API/list/detail | Deny/no leakage | F025/RBAC |
| PROB-QA-168 | Student ownership isolation | Student-A requests Student-B private resource | API | Deny | F025/RBAC |
| PROB-QA-169 | Frontend hides action but API direct call | UI hides edit | Call update endpoint directly | Backend denies if permission missing | F025/RBAC |
| PROB-QA-170 | Scope changes during request | Concurrent Section/Problem scope update | Request Problem using stale cached audience | Revalidate current status/scope/membership before access; stale authorization cannot bypass new scope | PROB-F025; PROB-D04/PROB-D05/PROB-D23 |

| PROB-QA-171 | ProblemSection same-College validation | SECTION_ONLY Problem owned by College A; Section in College B | Attempt to add foreign Section | Rejected; ProblemSection cannot cross College; no association created | PROB-F005/F025; PROB-D05/PROB-D23 |
| PROB-QA-172 | ProblemSection Department hierarchy | SECTION_ONLY Problem in Department A1; Section belongs to Department A2 | Attempt to link Section A2-1 | Rejected; Section must belong to owning Department for SECTION_ONLY | PROB-F005; PROB-D05 |
| PROB-QA-173 | Only SUPER_ADMIN can set GLOBAL | HOD-A and SUPER_ADMIN available; Problem College A | HOD sets GLOBAL; SUPER_ADMIN sets GLOBAL | HOD denied; SUPER_ADMIN permitted within owning College only | PROB-F005; PROB-D05/PROB-D23 |
| PROB-QA-174 | isPublic does not bypass Section scope | PUBLISHED isPublic Problem SECTION_ONLY linked to A1 | Student from A2/College B requests Practice route | Denied; only Student in linked Section A1 is eligible | PROB-F016; PROB-D03/PROB-D04/PROB-D05 |
| PROB-QA-175 | Immutable ProblemVersion on publish | DRAFT Problem with valid TestCases | Publish Version 1; change draft; publish Version 2 | Two immutable versions exist; Version 1 snapshot is unchanged | PROB-F006/F023; PROB-D17/PROB-D18 |
| PROB-QA-176 | New draft revision does not mutate current published version | Problem Version 1 published | Edit statement/tags/limits in new draft revision | Student-facing current published version remains Version 1 until Version 2 is published | PROB-F003/F008/F009/F010; PROB-D17 |
| PROB-QA-177 | Submission binds ProblemVersion | PUBLISHED Problem Version 1; submission request accepted | Create CodeSubmission through Module 6 | CodeSubmission references Version 1; future publish cannot change evaluation inputs | PROB-F023; PROB-D17/PROB-D18 |
| PROB-QA-178 | Archive preserves versions and relations | Problem has versions, activity links and submissions | Archive Problem | No cascade delete; all references and history remain queryable under permission | PROB-F007/F023; PROB-D19 |
| PROB-QA-179 | Restore returns to DRAFT | Problem ARCHIVED with historical versions | Restore as HOD | Status becomes DRAFT; new publish required; prior versions remain immutable | PROB-F007; PROB-D02/PROB-D17/PROB-D19 |
| PROB-QA-180 | Default resource values | New Problem draft omits timeLimitMs/memoryLimitMb | Create draft and inspect defaults | Suggested defaults are 2000 ms and 256 MB; deployment ceiling can be stricter | PROB-F010; PROB-D11 |
| PROB-QA-181 | Resource limit bounds | Configured target limits | Test 99ms/60001ms and 7MB/513MB | Reject values outside adopted 100–60000ms and 8–512MB bounds | PROB-F010; PROB-D11 |
| PROB-QA-182 | Activity resource override cannot broaden | Problem timeLimit 2000ms; activity proposes 5000ms | Launch activity attempt | Effective limit cannot exceed ProblemVersion or deployment ceiling | PROB-F010; PROB-D11/PROB-D17 |
| PROB-QA-183 | Bulk TestCase import atomicity | DRAFT Problem; import batch has valid and invalid rows | Submit bulk import | No partial commit; transaction rolls back or reports validation before commit | PROB-F011; PROB-D12/PROB-D25 |
| PROB-QA-184 | TestCase payload size guard | Configured input/output limits active | Submit oversized TestCase payload | Reject before persistence; limit comes from platform configuration | PROB-F011; PROB-D12 |
| PROB-QA-185 | Publish requires at least one TestCase | DRAFT Problem otherwise complete but with zero TestCases | Publish Problem | Rejected; remains DRAFT and no ProblemVersion is published | PROB-F006/F011; PROB-D12 |
| PROB-QA-186 | Published TestCase is immutable | ProblemVersion 1 published | Attempt direct update/delete of snapshot TestCase | Denied; edit only through new DRAFT revision | PROB-F012/F013; PROB-D17/PROB-D19 |
| PROB-QA-187 | Zero-based TestCase order | DRAFT Problem has three TestCases | Save orderIndex 0,1,2 and fetch list | Returned order is deterministic and identical to stored zero-based order | PROB-F014; PROB-D13 |
| PROB-QA-188 | Duplicate TestCase order | DRAFT Problem has two TestCases | Set both to orderIndex 1 | Reject or atomically normalize; unique contiguous zero-based order preserved | PROB-F014; PROB-D13/PROB-D25 |
| PROB-QA-189 | Hidden input/output absent from every Student surface | Problem has hidden and visible cases | Inspect details, practice, course, lab, assessment, contest, search, errors and logs | No hidden input/output, ID, exact count or order metadata reaches Student/client surfaces | PROB-F015; PROB-D14 |
| PROB-QA-190 | Visible and hidden cases both execute | Published Problem has visible and hidden TestCases | Submit eligible solution through Module 6/7 | Both types enter formal evaluation; only visible samples returned to Student; scoring belongs to Module 7 | PROB-F015; PROB-D14/PROB-D15 |
| PROB-QA-191 | Visible TestCase sample payload | Problem published with isHidden=false TestCase | Fetch Student-safe Problem details | Visible input and expectedOutput returned; TestCase remains part of formal evaluation | PROB-F015; PROB-D14/PROB-D15 |
| PROB-QA-192 | Practice Problem requires isPublic | Published in-scope Problem has isPublic=false | Student searches Practice Problems | Problem is not listed; flag does not change activity-link eligibility | PROB-F016; PROB-D03/PROB-D21 |
| PROB-QA-193 | isPublic true allows scoped Practice discovery | Published in-scope Problem has isPublic=true | Eligible authenticated Student searches Practice Problems | Problem listed only for Student satisfying College/Department/Section/batch scope | PROB-F016; PROB-D03/PROB-D04/PROB-D21 |
| PROB-QA-194 | Activity link does not bypass Problem scope | Course/Lab/Assessment/Contest references Problem outside Student scope | Student opens linked activity Problem | Denied; activity and Problem access are conjunctive; link cannot widen tenant/scope | PROB-F017/F018/F019/F020; PROB-D04/PROB-D16/PROB-D23 |
| PROB-QA-195 | Markdown statement sanitization | Problem statement contains safe Markdown and raw script | Render Student Problem view | Markdown displays; script/unsafe HTML stripped or rejected and never executes | PROB-F008; PROB-D06 |
| PROB-QA-196 | Audit payload excludes protected content | Problem/TestCase mutation includes hidden input/output | Inspect Module 15 audit event | Event includes actor/action/target/scope/version but excludes hidden values, source code and full statement | PROB-F024; PROB-D25 |
| PROB-QA-197 | HOD/Manager submission history scope | HOD-A and Manager-A; submissions in A1 and A2 | Query Problem-linked submission history | HOD sees assigned Department; Manager assigned Section; Student own; Finance denied | PROB-F022; PROB-D01/PROB-D22 |
| PROB-QA-198 | Canonical language identifier mapping | Problem allowedLanguages contains canonical key; Judge0 mapping changes | Submit in supported language | Module 6 resolves canonical key to current permitted Judge0 ID; client cannot inject runtime IDs | PROB-F010; PROB-D10 |
| PROB-QA-199 | Activity version pinning | Course/Assessment/Contest uses ProblemVersion 1 | Publish ProblemVersion 2 | Activity continues Version 1 until its owning module explicitly rebinds to Version 2 | PROB-F017/F019/F020/F023; PROB-D16/PROB-D17/PROB-D18 |
| PROB-QA-200 | Public Problem still requires active membership | isPublic=true; Problem PUBLISHED; Student inactive or College suspended | Request Practice Problem access | Denied; isPublic is discovery only and does not bypass active membership/College status | PROB-F016/F025; PROB-D03/PROB-D04/PROB-D21 |

## QA review notes

PROB-QA-001–200 map to PROB-F001–PROB-F025 and PROB-D01–PROB-D25. Product decisions are adopted; implementation execution remains pending. Do not mark a case passed until run against the implementation with evidence recorded. The suite includes tenant/role security, Practice visibility, immutable versions, hidden TestCase protection, ordering, Judge0 configuration boundaries and integration failure coverage.