# Module 8 — Labs & Lab Scoring QA Test Cases

**Version:** 0.2 — QA companion aligned to Functional Specification Standard v1.0  
**Status:** Detailed draft; target behavior/decisions remain open  
**Specification:** [Module 8 — Labs & Lab Scoring](./README.md)

> These test cases are grounded in the six source documents and Module 8 functional draft. Where source behavior is missing, expected behavior is decision-blocked and explicitly marked. These QA tests do not approve a proposed product rule.

## Test conventions
- Staff-HOD: HOD within assigned College + Department.
- Staff-Manager: Manager within assigned College + Department + Section/permitted Group.
- Student-A: Student owns the submission.
- Student-B: different Student; foreign ownership case.
- Group-A: Group within authorized staff scope; Group-B: out-of-scope Group.
- Lab-A: LabAssignment in scope; Lab-B: Lab outside actor scope.
- Course-A/Problem-A: valid scoped entities; Course-B/Problem-B: outside scope.
- **Decision-blocked** means a source gap must be resolved later before target QA can be finalized.

| Test ID | Scenario | Preconditions / test data | Steps | Expected result | Trace |
|---|---|---|---|---|---|
| LAB-QA-001 | HOD lists Labs in Department | HOD assigned Department-A; Labs in A/B | Open Courses & Labs | Only Department-A permitted Labs returned | LAB-F001/R01 |
| LAB-QA-002 | Manager lists Labs in assigned Section | Manager assigned Section-A | Open page | Results limited to assigned Section/permitted Groups | LAB-F001 |
| LAB-QA-003 | Super Admin list | Super Admin | Open staff Lab list | Global access per RBAC | LAB-F001 |
| LAB-QA-004 | Finance Team views Labs | Finance Team assigned College-A | Open list/detail | View allowed only within approved field/scope policy; field-level decision blocked | LAB-F001/D36 |
| LAB-QA-005 | Student opens staff Courses & Labs | Student | Directly request staff page/API | Denied; Student uses My Labs/Assignments | LAB-F001 |
| LAB-QA-006 | Foreign scope filter | Manager requests Department/Section B | Apply filter/direct API | Scope filter cannot broaden access or disclose existence | LAB-F001/V001 |
| LAB-QA-007 | Create Lab with valid Course and Problem | HOD; Course-A/Problem-A | Fill source fields and save | LabAssignment created with valid references | LAB-F002 |
| LAB-QA-008 | Create Lab missing Course | Staff create permission | Submit without courseId | Save blocked | LAB-F002/V002 |
| LAB-QA-009 | Create Lab missing Problem | Staff create permission | Submit without problemId | Save blocked | LAB-F002/V003 |
| LAB-QA-010 | Course outside staff scope | Manager; Course-B | Attempt create | Denied; no Lab created | LAB-F002 |
| LAB-QA-011 | Problem outside staff scope | HOD; Problem-B | Attempt create | Denied; no Lab created | LAB-F002 |
| LAB-QA-012 | Title exceeds 255 chars | Create form | Submit oversized title | Reject field | LAB-F002/V004 |
| LAB-QA-013 | Invalid ContentScope | Scope value not in enum | Submit | Reject; allowed enum only | LAB-F002/V005 |
| LAB-QA-014 | maxScore invalid | zero/negative/large boundary | Submit | Expected result decision-blocked until range approved | LAB-F002/D04 |
| LAB-QA-015 | dueDate malformed | invalid timestamp | Submit | Reject date/time input | LAB-F002/V007 |
| LAB-QA-016 | Staff edits valid Lab | HOD scoped Lab | Update title/description | Fields update; updatedAt refreshed | LAB-F003 |
| LAB-QA-017 | Student edits Lab | Student | Direct update API | Deny | LAB-F003/V010 |
| LAB-QA-018 | Manager edits out-of-scope Lab | Manager; Lab-B | Direct update | Deny | LAB-F003 |
| LAB-QA-019 | Change Problem after submissions exist | Lab has submissions | Edit problemId | Behavior decision-blocked; historic scores must not silently change | LAB-F003/D12 |
| LAB-QA-020 | Change maxScore after grading | Graded Lab | Edit maxScore | Recalculation/history policy decision-blocked | LAB-F003/D08/D12 |
| LAB-QA-021 | Change dueDate after submission | Lab has submission | Edit deadline | Historical late classification must follow approved policy | LAB-F003/D09 |
| LAB-QA-022 | Concurrent staff edits | Two editors | Save stale edit | Conflict/reload behavior; no silent overwrite | LAB-F003/V008 |
| LAB-QA-023 | Change Lab scope | Existing Group/Student assignments | Update scope | Eligibility follows approved scope policy; no unauthorized access | LAB-F003/D32 |
| LAB-QA-024 | Publish valid Lab | Lifecycle field/flow approved | Publish | Status transition and visibility follow approved state model | LAB-F004/D10 |
| LAB-QA-025 | Publish incomplete Lab | Missing required fields | Publish | Required fields block publication if approved; required set TBD | LAB-F004/V012 |
| LAB-QA-026 | Student archives Lab | Student | Direct status API | Deny | LAB-F004/V013 |
| LAB-QA-027 | Invalid lifecycle enum | status=Closed or unsupported | Save transition | Reject unsupported enum; Closed not in source enum | LAB-F004/R10 |
| LAB-QA-028 | Archive Lab with submissions | Historical records exist | Archive | Preserve LabSubmission/CodeSubmission history | LAB-F004/R11 |
| LAB-QA-029 | Reopen archived Lab | Archived Lab | Request reopen | Transition policy decision-blocked | LAB-F004/D11 |
| LAB-QA-030 | Lab delete permission for Manager | Manager | Attempt delete | Decision-blocked due RBAC Full CRUD vs V/C/U* discrepancy; do not assume delete | LAB-F004/D06 |
| LAB-QA-031 | Assign Lab to Group-A | Staff scoped to Group-A | Create GroupLab relation | Relation created with correct IDs/timestamp | LAB-F005 |
| LAB-QA-032 | Assign Lab to Group-B | Staff not authorized for Group-B | Assign | Deny | LAB-F005/V014 |
| LAB-QA-033 | Group outside Lab/College scope | Group from foreign College | Assign | Deny unless explicit approved scope exception | LAB-F005/V015 |
| LAB-QA-034 | Duplicate GroupLab relation | Existing relation | Add same link again | Idempotency/constraint behavior decision-blocked; no unintended duplicates | LAB-F005/V016 |
| LAB-QA-035 | Group inactive | Group.isActive=false | Assign/access | Behavior decision-blocked; inactive Group must be evaluated | LAB-F005/D14 |
| LAB-QA-036 | Remove GroupLab after submissions | Existing Student submissions | Remove assignment | Preserve history; future access policy decision-blocked | LAB-F005/D15 |
| LAB-QA-037 | Group membership alone grants Lab | Student is GroupMember but not enrolled | Check access | Decision-blocked; Group membership and CourseEnrollment are distinct | LAB-F005/F017 |
| LAB-QA-038 | Lab assigned to multiple Groups | Multiple GroupLab rows | View eligible list | Multiple assignment behavior decision-blocked | LAB-F005/D05 |
| LAB-QA-039 | Student sees eligible Lab | Student enrolled/assigned | Open My Labs | Eligible Lab appears | LAB-F006 |
| LAB-QA-040 | Student not enrolled/assigned | No approved eligibility relation | Open My Labs | Deny/exclude per eligibility policy | LAB-F006/D14 |
| LAB-QA-041 | Student tries studentId spoofing | Student-A requests Student-B list | Direct API | Deny; identity derived from session | LAB-F006/V018 |
| LAB-QA-042 | Student joins Group after Lab publish | GroupLab exists | Refresh My Labs | Inheritance policy decision-blocked | LAB-F006/D15 |
| LAB-QA-043 | Student leaves Group | Prior eligibility | Refresh/access Lab | Current vs historical access decision-blocked | LAB-F006/D15 |
| LAB-QA-044 | Student changes Section | CourseSection changed | Access Lab | Scope/eligibility rechecked; rule TBD | LAB-F006/D14 |
| LAB-QA-045 | Student revoked CourseEnrollment | Enrollment removed/inactive | Access Lab | Current access and historical score policy decision-blocked | LAB-F006/D14 |
| LAB-QA-046 | No Labs available | Student has no eligible Lab | Open My Labs | Empty state; no unauthorized records | LAB-F006 |
| LAB-QA-047 | Student opens valid Lab detail | Eligible Student | Open Lab | Correct Lab, Course, Problem shown | LAB-F007 |
| LAB-QA-048 | Foreign Student opens Lab | Student not eligible | Direct Lab ID | Deny/not-found-safe | LAB-F007 |
| LAB-QA-049 | Problem relation mismatch | Client tries substitute Problem ID | Open/submit | Server uses LabAssignment.problemId | LAB-F007/R17 |
| LAB-QA-050 | Hidden testcase data requested | Student detail request | Inspect response | Hidden inputs/expected outputs not returned | LAB-F007/V020 |
| LAB-QA-051 | Problem archived after Lab publish | Historical Lab | Open Lab | Historical visibility/version behavior decision-blocked | LAB-F007/D13 |
| LAB-QA-052 | Lab archived | Student opens detail | Request detail | Access follows approved archive/history policy | LAB-F007/D11 |
| LAB-QA-053 | Student submits valid code | Eligible Lab/Problem/language | Submit | CodeSubmission and LabSubmission linkage created; evaluation begins | LAB-F008 |
| LAB-QA-054 | Student submits inaccessible Lab | Foreign Lab | Submit | Deny before CodeSubmission/Judge0 creation | LAB-F008/R20 |
| LAB-QA-055 | Student substitutes arbitrary Problem | Lab links Problem-A; request Problem-B | Submit | Reject/ignore client Problem ID; use linked Problem | LAB-F008/R19 |
| LAB-QA-056 | Student ID spoof | Client sends Student-B ID | Submit as Student-A | Server uses Student-A auth; deny spoof attempt | LAB-F008 |
| LAB-QA-057 | Unsupported language | Language unsupported by Module 6 | Submit | Module 6 validation; no invalid execution | LAB-F008/V023 |
| LAB-QA-058 | Submit after dueDate | Deadline passed | Submit | Decision-blocked until late/hard cutoff policy approved | LAB-F008/D09 |
| LAB-QA-059 | Attempt limit reached | Attempt policy configured | Submit again | Block per approved limit; policy TBD | LAB-F008/D18 |
| LAB-QA-060 | Duplicate submit click | Same request retried | Submit twice | Idempotency/duplicate attempt behavior decision-blocked | LAB-F008/V024 |
| LAB-QA-061 | Judge0 outage after accepted submit | CodeSubmission accepted | Execute | Operational failure/recovery, not Student verdict | LAB-F008/F010 |
| LAB-QA-062 | Deadline race | Submit near dueDate | Submit concurrently | Server timestamp and inclusive rule decision-blocked | LAB-F008/D29/D31 |
| LAB-QA-063 | Evaluation pending | CodeSubmission created | View result | Pending/incomplete state; no final score | LAB-F008/F010 |
| LAB-QA-064 | Lab submission to wrong Group context | Student Group not assigned | Submit | Deny per eligibility | LAB-F008/F017 |
| LAB-QA-065 | Create LabSubmission with valid linkage | CodeSubmission accepted in Lab context | Persist | LabAssignment/Student/CodeSubmission IDs consistent | LAB-F009 |
| LAB-QA-066 | LabSubmission links foreign Student | Student-A submission but Student-B id | Persist | Reject integrity violation | LAB-F009/R23 |
| LAB-QA-067 | LabSubmission links foreign Lab | CodeSubmission for Lab-A, LabSubmission for Lab-B | Persist | Reject mismatch | LAB-F009/R24 |
| LAB-QA-068 | LabSubmission links wrong CodeSubmission | IDs mismatch | Persist | Reject relation mismatch | LAB-F009 |
| LAB-QA-069 | LabSubmission create retries | Same submission event repeated | Process event | Idempotent/reconcile; no unintended duplicate | LAB-F009/V027 |
| LAB-QA-070 | LabSubmission write fails after CodeSubmission | DB failure | Submit flow | Recovery/reconciliation; no orphan silently lost | LAB-F009/E16 |
| LAB-QA-071 | Evaluation enqueue fails after LabSubmission | Queue failure | Process submission | Recover evaluation without creating another attempt | LAB-F009/E17 |
| LAB-QA-072 | Evaluate Lab submission | Valid linked CodeSubmission | Run Module 6/7 | Correct Problem/testcase evaluation | LAB-F010 |
| LAB-QA-073 | Evaluation belongs to another CodeSubmission | Wrong ID/result | Correlate | Reject result | LAB-F010/V030 |
| LAB-QA-074 | Judge0 infrastructure error | Judge0 unavailable | Evaluate | Distinct operational status, not default Wrong Answer | LAB-F010/R27 |
| LAB-QA-075 | Partial testcase results | Evaluation incomplete | Consume result | No final Lab score unless policy permits | LAB-F010/V029 |
| LAB-QA-076 | Hidden testcase result leak | Evaluation payload includes hidden input/output | Student result | Module 7 masks hidden details | LAB-F010/F012 |
| LAB-QA-077 | Duplicate evaluation completion event | Same result event twice | Process | Idempotent score update | LAB-F010/F020 |
| LAB-QA-078 | Rejudge result arrives late | Old generation after newer | Consume | Stale result cannot overwrite latest | LAB-F010/F019 |
| LAB-QA-079 | Problem/testcase changed before worker | Queued submission | Evaluate | Version/snapshot decision-blocked | LAB-F010/D13 |
| LAB-QA-080 | Convert 100% evaluation score | maxScore configured | Calculate | Formula decision-blocked; if approved, follows formula exactly | LAB-F011/D07 |
| LAB-QA-081 | Convert partial evaluation | Partial percent | Calculate | Score conversion/partial policy decision-blocked | LAB-F011/D07 |
| LAB-QA-082 | Evaluation verdict-only, no numeric percent | CE/RE/TLE etc. | Calculate | Verdict-to-score mapping decision-blocked | LAB-F011/D07 |
| LAB-QA-083 | maxScore zero | Lab maxScore=0 | Calculate | Reject/explicit rule; no divide-by-zero | LAB-F011/V032 |
| LAB-QA-084 | Score exceeds maxScore | invalid evaluation/manual data | Persist | Reject/flag under approved range | LAB-F011/R28 |
| LAB-QA-085 | Rounding boundary | computed decimal near rounding boundary | Calculate | Precision/rounding decision-blocked | LAB-F011/D08 |
| LAB-QA-086 | Score precision exceeds decimal(5,2) | calculated value >2 decimals | Persist/display | Storage/display rounding policy TBD | LAB-F011 |
| LAB-QA-087 | Rejudge changes evaluation score | Existing score | Rejudge | Recalculation/history policy decision-blocked | LAB-F011/D21 |
| LAB-QA-088 | maxScore changes after score assigned | Existing LabSubmission | Edit Lab | Recalculation policy decision-blocked | LAB-F011/D27 |
| LAB-QA-089 | Late penalty not configured | Late submission | Calculate | No penalty assumed/applied | LAB-F011/R44 |
| LAB-QA-090 | Infrastructure failure result | Judge0 failure | Calculate | No fabricated Student grade | LAB-F011/R27 |
| LAB-QA-091 | Student reads own LabSubmission | Student-A owns | GET result | Safe own result returned | LAB-F012 |
| LAB-QA-092 | Student reads Student-B LabSubmission | Foreign ID | GET result | Deny/not-found-safe | LAB-F012/V035 |
| LAB-QA-093 | Student requests raw Judge0 result | Student | GET raw endpoint | Deny or safe DTO only | LAB-F012/R34 |
| LAB-QA-094 | Result pending | Evaluation running | Read | Pending distinct from final | LAB-F012/V036 |
| LAB-QA-095 | Rejudge pending | Current run pending | Read | State/history policy applied; no false final | LAB-F012 |
| LAB-QA-096 | Student enrollment revoked | Historical LabSubmission | Read | Historical access policy decision-blocked | LAB-F012/D14 |
| LAB-QA-097 | Lab archived but Student owns prior score | Prior submission | Read | Historical access policy decision-blocked | LAB-F012/D11 |
| LAB-QA-098 | HOD reviews Department Lab results | Scoped Students | Open Student Performance | Only College/Department records | LAB-F013 |
| LAB-QA-099 | HOD reviews foreign Department | Foreign record | Direct ID/filter | Deny | LAB-F013 |
| LAB-QA-100 | Manager reviews assigned Section | Section-A | Open performance page | Permitted Section records only | LAB-F013 |
| LAB-QA-101 | Manager reviews foreign Section | Section-B | Direct ID/filter | Deny | LAB-F013 |
| LAB-QA-102 | Finance Team views Student scores | Finance Team | Open detail | Decision-blocked; RBAC broad View needs field-level policy | LAB-F013/D36 |
| LAB-QA-103 | Staff has View but requests raw JudgeRun | Staff role | Raw endpoint | Deny absent separate permission | LAB-F013/R37 |
| LAB-QA-104 | Staff exports Lab scores | No export permission | Export action | Deny; export not implied by View | LAB-F013/V038 |
| LAB-QA-105 | Staff filters by foreign Student | Out-of-scope ID | Query | Deny/scope-filter | LAB-F013/V037 |
| LAB-QA-106 | Staff writes feedback | Scoped submission | Save feedback | Persist feedback | LAB-F014 |
| LAB-QA-107 | Staff feedback to foreign submission | Out-of-scope ID | Save | Deny | LAB-F014/V039 |
| LAB-QA-108 | Student reads own feedback | Student-A | View result | Feedback returned | LAB-F014 |
| LAB-QA-109 | Student reads another Student feedback | Student-B | Direct ID | Deny | LAB-F014 |
| LAB-QA-110 | Feedback exceeds length limit | Long text | Save | Reject once limit specified; limit TBD | LAB-F014/V040 |
| LAB-QA-111 | Feedback cleared/blank | Existing feedback | Save blank | Clear/no-op behavior decision-blocked | LAB-F014/V041 |
| LAB-QA-112 | Manual override feature unapproved | No policy | Attempt override | Feature unavailable/denied | LAB-F015 |
| LAB-QA-113 | Override by unauthorized role | No grading permission | Override score | Deny | LAB-F015/V043 |
| LAB-QA-114 | Override with reason | Feature approved | Adjust score | Reason and original score retained per policy | LAB-F015 |
| LAB-QA-115 | Override without reason | Reason required | Save | Reject | LAB-F015/V042 |
| LAB-QA-116 | Override exceeds maxScore | Approved override | Save | Reject unless bonus policy allows | LAB-F015/R41 |
| LAB-QA-117 | Concurrent overrides | Two graders | Save competing changes | Conflict/review; no silent overwrite | LAB-F015/V044 |
| LAB-QA-118 | Rejudge after manual override | Existing override | Rejudge | Precedence/history policy decision-blocked | LAB-F015/D34 |
| LAB-QA-119 | Submit before dueDate | Deadline configured | Submit | On-time status according to approved policy | LAB-F016 |
| LAB-QA-120 | Submit exactly at dueDate | Deadline boundary | Submit | Inclusive/exclusive behavior decision-blocked | LAB-F016/D29 |
| LAB-QA-121 | Submit after dueDate | Deadline passed | Submit | Late acceptance/denial policy decision-blocked | LAB-F016/D09 |
| LAB-QA-122 | Judge0 completes after dueDate | Submitted before deadline | Evaluate | Late classification based on submission time under derived rule | LAB-F016/R43 |
| LAB-QA-123 | Staff extends dueDate | Individual extension requested | Update | Extension support decision-blocked | LAB-F016/D30 |
| LAB-QA-124 | Timezone/DST boundary | Different user locale | Submit | Timezone authority decision-blocked | LAB-F016/D29 |
| LAB-QA-125 | DueDate edited after graded submission | Existing score | Update deadline | Historic lateness not silently recalculated | LAB-F016/V047 |
| LAB-QA-126 | Client sends forged submittedAt | Student request | Submit | Server timestamp is authoritative per decision | LAB-F016/D31 |
| LAB-QA-127 | Student enrolled and in assigned Group | Eligible relationships | Request Lab | Access allowed | LAB-F017 |
| LAB-QA-128 | Student GroupMember but not enrolled | Group membership only | Request Lab | Decision-blocked; relation precedence unresolved | LAB-F017/D14 |
| LAB-QA-129 | Student enrolled but not GroupMember | Group assignment required | Request Lab | Decision-blocked | LAB-F017/D14/D16 |
| LAB-QA-130 | Student not in Section scope | SECTION_ONLY Lab | Request | Deny under approved scope definition | LAB-F017/D32 |
| LAB-QA-131 | DEPT_WIDE Lab | Department Student | Request | Access under approved semantics; enum name alone insufficient | LAB-F017/D32 |
| LAB-QA-132 | GLOBAL Lab in scoped Course | Cross-Department Student | Request | Scope conflict decision-blocked | LAB-F017/D32 |
| LAB-QA-133 | Student removed from Group | Former member | Request Lab | Current access policy decision-blocked | LAB-F017/D15 |
| LAB-QA-134 | Group inactive | isActive=false | Student request | Eligibility policy decision-blocked | LAB-F017/D14 |
| LAB-QA-135 | CourseEnrollment completedAt set | Completed enrollment | Request Lab | Access policy decision-blocked | LAB-F017/D14 |
| LAB-QA-136 | Enrollment revoked | Student no longer enrolled | Request | Deny current access unless approved historical access | LAB-F017 |
| LAB-QA-137 | Cross-College relation | foreign Course/Group | Request | Deny unless explicit global exception approved | LAB-F017 |
| LAB-QA-138 | Student has not submitted | Eligible Lab | View progress | Display not-submitted only if status semantics approved | LAB-F018/D33 |
| LAB-QA-139 | Student submitted, evaluation pending | LabSubmission exists | View progress | Submitted/pending; not completed | LAB-F018/R50 |
| LAB-QA-140 | Evaluation complete, score below threshold | Threshold undefined | View progress | Completion policy decision-blocked | LAB-F018/D17 |
| LAB-QA-141 | CourseEnrollment.completedAt set | Course completed | View Lab progress | Must not imply Lab completion | LAB-F018/R48 |
| LAB-QA-142 | Multiple attempts | Several LabSubmission rows | View progress | Best/latest/attempt summary policy decision-blocked | LAB-F018/D18 |
| LAB-QA-143 | Lab archived | Prior progress | View Dashboard | Historical progress policy decision-blocked | LAB-F018 |
| LAB-QA-144 | Score manually adjusted | Override enabled | View progress | Completion/progress update follows approved policy | LAB-F018/D22/D33 |
| LAB-QA-145 | Authorized rejudge | Existing submission | Request rejudge | Module 6/7 permissions and state apply | LAB-F019 |
| LAB-QA-146 | Student requests rejudge | Student | Rerun endpoint | Deny absent approved Student rerun permission | LAB-F019 |
| LAB-QA-147 | Rejudge active | Evaluation already running | Request another rerun | Serialize/reject per Module 6/7 | LAB-F019/D20 |
| LAB-QA-148 | Rejudge returns newer result | Existing Lab score | Process result | Update score only per approved recalculation/history policy | LAB-F019/D21 |
| LAB-QA-149 | Stale rejudge result | Old generation | Process late result | Cannot overwrite current generation | LAB-F019/R52 |
| LAB-QA-150 | Rejudge fails operationally | Judge0 outage | Rerun | Preserve recoverable status; no false score | LAB-F019 |
| LAB-QA-151 | Rejudge creates attempt | Existing Student attempt | Inspect attempt count | Must not count as new attempt unless explicitly approved | LAB-F019/R51 |
| LAB-QA-152 | Lab create audit event | Audit policy enabled | Create Lab | Module 15 event with approved actor/target context | LAB-F020 |
| LAB-QA-153 | Lab publish notification | Notification policy enabled | Publish | Module 13 recipient/event policy applied | LAB-F020 |
| LAB-QA-154 | Score notification | Final score event | Complete evaluation | Recipient/notification policy applied | LAB-F020 |
| LAB-QA-155 | Audit/notification failure | Operation committed | Event integration | Failure/retry behavior follows Modules 13/15/18 | LAB-F020/D35 |
| LAB-QA-156 | Duplicate score event | Same event delivered twice | Consumer processes | No duplicate score/progress side effects | LAB-F020/R56 |
| LAB-QA-157 | Hidden testcase data in notification | Event includes raw payload | Publish result event | Hidden input/output omitted/masked | LAB-F020/R56 |
| LAB-QA-158 | Source code/token in audit payload | Raw execution context | Emit audit | Sensitive code/token not included in normal event | LAB-F020/R56 |

## QA review notes
- LAB-QA-001–158 cover LAB-F001–F020.
- Cases marked decision-blocked are deliberately retained for later review after all modules are authored.
- Source-confirmed enum/field/permission tests should be verified against the exact source version; legacy audit findings are not target approval.
