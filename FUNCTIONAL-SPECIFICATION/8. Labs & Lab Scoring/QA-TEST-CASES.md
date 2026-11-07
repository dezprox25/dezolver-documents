# Module 8 — Labs & Lab Scoring QA Test Cases

**Version:** 1.0 — Adopted target baseline QA companion  
**Status:** 188 target cases defined; implementation execution not yet performed  
**Specification:** [Module 8 — Labs & Lab Scoring](./README.md)

> These test cases define the adopted Module 8 target behavior. They are test specifications, not execution evidence. Source/audit history remains separate from the adopted decisions.

## Test conventions
- Staff-HOD: HOD within assigned College + Department.
- Staff-Manager: Manager within assigned College + Department + Section/permitted Group.
- Student-A: Student owns the submission.
- Student-B: different Student; foreign ownership case.
- Group-A: Group within authorized staff scope; Group-B: out-of-scope Group.
- Lab-A: LabAssignment in scope; Lab-B: Lab outside actor scope.
- Course-A/Problem-A: valid scoped entities; Course-B/Problem-B: outside scope.
- LAB-D01–LAB-D38 resolve the previously open product/architecture/security questions; QA expectations below are target expectations, not pass/fail execution results.

| Test ID | Scenario | Preconditions / test data | Steps | Expected result | Trace |
|---|---|---|---|---|---|
| LAB-QA-001 | HOD lists Labs in Department | HOD assigned Department-A; Labs in A/B | Open Courses & Labs | Only Department-A permitted Labs returned | LAB-F001/R01 |
| LAB-QA-002 | Manager lists Labs in assigned Section | Manager assigned Section-A | Open page | Results limited to assigned Section/permitted Groups | LAB-F001 |
| LAB-QA-003 | Super Admin list | Super Admin | Open staff Lab list | Global access per RBAC | LAB-F001 |
| LAB-QA-004 | Finance metadata-only view | Finance Team assigned College-A; scoped Labs exist | Open Lab list/detail | Return Lab title/Course/Problem metadata, status, maxScore and dueDate only; no Student submission/score/feedback | LAB-F001; LAB-D06/D36 |
| LAB-QA-005 | Student opens staff Courses & Labs | Student | Directly request staff page/API | Denied; Student uses My Labs/Assignments | LAB-F001 |
| LAB-QA-006 | Foreign scope filter | Manager requests Department/Section B | Apply filter/direct API | Scope filter cannot broaden access or disclose existence | LAB-F001/V001 |
| LAB-QA-007 | Create Lab with valid Course and Problem | HOD; Course-A/Problem-A | Fill source fields and save | LabAssignment created with valid references | LAB-F002 |
| LAB-QA-008 | Create Lab missing Course | Staff create permission | Submit without courseId | Save blocked | LAB-F002/V002 |
| LAB-QA-009 | Create Lab missing Problem | Staff create permission | Submit without problemId | Save blocked | LAB-F002/V003 |
| LAB-QA-010 | Course outside staff scope | Manager; Course-B | Attempt create | Denied; no Lab created | LAB-F002 |
| LAB-QA-011 | Problem outside staff scope | HOD; Problem-B | Attempt create | Denied; no Lab created | LAB-F002 |
| LAB-QA-012 | Title exceeds 255 chars | Create form | Submit oversized title | Reject field | LAB-F002/V004 |
| LAB-QA-013 | Invalid ContentScope | Scope value not in enum | Submit | Reject; allowed enum only | LAB-F002/V005 |
| LAB-QA-014 | maxScore invalid | zero, negative or outside integer type range | Create LabAssignment | Reject; maxScore must be a positive integer and score precision must be representable | LAB-F002; LAB-D04/D08 |
| LAB-QA-015 | dueDate malformed | invalid timestamp | Submit | Reject date/time input | LAB-F002/V007 |
| LAB-QA-016 | Staff edits valid Lab | HOD scoped Lab | Update title/description | Fields update; updatedAt refreshed | LAB-F003 |
| LAB-QA-017 | Student edits Lab | Student | Direct update API | Deny | LAB-F003/V010 |
| LAB-QA-018 | Manager edits out-of-scope Lab | Manager; Lab-B | Direct update | Deny | LAB-F003 |
| LAB-QA-019 | Change Problem after submissions | Published Lab has accepted attempts | Edit problemId | Require a new LabAssignmentRevision and published ProblemVersion; existing attempts retain original revision/version | LAB-F003; LAB-D12/D13 |
| LAB-QA-020 | Change maxScore after grading | Published revision has finalized score | Edit maxScore | Create new Lab revision; prior scores remain calculated from original maxScore snapshot | LAB-F003/F011; LAB-D08/D12/D27 |
| LAB-QA-021 | Change dueDate after submission | Published revision has prior attempts | Edit dueDate | Create a new revision; previous submittedAt/isLate values do not change | LAB-F003/F016; LAB-D12/D29 |
| LAB-QA-022 | Concurrent staff edits | Two editors | Save stale edit | Conflict/reload behavior; no silent overwrite | LAB-F003/V008 |
| LAB-QA-023 | Change Lab scope | Existing Group/Student assignments | Update scope | Eligibility follows approved scope policy; no unauthorized access | LAB-F003/D32 |
| LAB-QA-024 | Publish valid Lab | Lifecycle field/flow approved | Publish | Status transition and visibility follow approved state model | LAB-F004/D10 |
| LAB-QA-025 | Publish incomplete Lab | Missing title, Course, Problem, positive maxScore or valid audience/ProblemVersion | Publish | Block publication; all required source and target configuration must validate | LAB-F004; LAB-D02/D03/D04/D12/D13 |
| LAB-QA-026 | Student archives Lab | Student | Direct status API | Deny | LAB-F004/V013 |
| LAB-QA-027 | Invalid lifecycle enum | status=Closed or unsupported | Save transition | Reject unsupported enum; Closed not in source enum | LAB-F004/R10 |
| LAB-QA-028 | Archive Lab with submissions | Historical records exist | Archive | Preserve LabSubmission/CodeSubmission history | LAB-F004/R11 |
| LAB-QA-029 | Reopen archived Lab | Lab status ARCHIVED | Request reopen | Restore to DRAFT; a new publish operation/revision is required | LAB-F004; LAB-D10/D11 |
| LAB-QA-030 | Manager attempts Lab Delete | Manager assigned Section | Call DELETE/archive endpoint | Deny; Manager has scoped View/Create/Update only and no Delete/lifecycle action | LAB-F004; LAB-D06/D11 |
| LAB-QA-031 | Assign Lab to Group-A | Staff scoped to Group-A | Create GroupLab relation | Relation created with correct IDs/timestamp | LAB-F005 |
| LAB-QA-032 | Assign Lab to Group-B | Staff not authorized for Group-B | Assign | Deny | LAB-F005/V014 |
| LAB-QA-033 | Group outside Lab/College scope | Group from foreign College | Assign | Deny unless explicit approved scope exception | LAB-F005/V015 |
| LAB-QA-034 | Duplicate GroupLab relation | GroupLab link already exists | Add same Group link again | Idempotent no-op or conflict; never create duplicate association | LAB-F005; LAB-D05/D35 |
| LAB-QA-035 | Group inactive | Group.isActive=false | Assign/access Lab | Reject new GroupLab assignment and deny Student access through inactive Group | LAB-F005/F017; LAB-D05/D14 |
| LAB-QA-036 | Remove GroupLab after submissions | Existing LabSubmissions for Group members | Remove GroupLab link | Block new Group-based access/attempts; preserve each Student's own prior Lab history read-only | LAB-F005/F006; LAB-D15/D24 |
| LAB-QA-037 | Group membership without CourseEnrollment | Student is GroupMember but not enrolled in Course | Check eligibility | Deny; Group membership never replaces active CourseEnrollment | LAB-F005/F017; LAB-D14 |
| LAB-QA-038 | Lab assigned to multiple Groups | Multiple GroupLab rows | Check eligible Students | Students in any linked active Group qualify only when Course/Section/Lab/Problem checks also pass | LAB-F005/F017; LAB-D05/D14 |
| LAB-QA-039 | Student sees eligible Lab | Student enrolled/assigned | Open My Labs | Eligible Lab appears | LAB-F006 |
| LAB-QA-040 | Student not enrolled/assigned | No approved eligibility relation | Open My Labs | Deny/exclude per eligibility policy | LAB-F006/D14 |
| LAB-QA-041 | Student tries studentId spoofing | Student-A requests Student-B list | Direct API | Deny; identity derived from session | LAB-F006/V018 |
| LAB-QA-042 | Student joins Group after publish | Lab has GroupLab; Student becomes active GroupMember and remains enrolled | Refresh My Labs | Lab becomes visible and submittable after current membership/other eligibility checks pass | LAB-F006/F017; LAB-D14/D15 |
| LAB-QA-043 | Student leaves/removal from Group | Previously eligible Student has a prior submission | Refresh/access Lab | Deny new attempts; allow own historical LabSubmission/result as read-only | LAB-F006/F012; LAB-D15 |
| LAB-QA-044 | Student changes Section | CourseSection/User Section changes | Request list/detail/submit | Re-evaluate current active Section against Course/Lab/Problem scope; deny new access when no longer included | LAB-F006/F017; LAB-D14/D32 |
| LAB-QA-045 | CourseEnrollment withdrawn | CourseEnrollment.status=WITHDRAWN | Request Lab | Deny new Lab access/submission; preserve own prior submission history read-only | LAB-F006/F017; LAB-D14/D15 |
| LAB-QA-046 | No Labs available | Student has no eligible Lab | Open My Labs | Empty state; no unauthorized records | LAB-F006 |
| LAB-QA-047 | Student opens valid Lab detail | Eligible Student | Open Lab | Correct Lab, Course, Problem shown | LAB-F007 |
| LAB-QA-048 | Foreign Student opens Lab | Student not eligible | Direct Lab ID | Deny/not-found-safe | LAB-F007 |
| LAB-QA-049 | Problem relation mismatch | Client tries substitute Problem ID | Open/submit | Server uses LabAssignment.problemId | LAB-F007/R17 |
| LAB-QA-050 | Hidden testcase data requested | Student detail request | Inspect response | Hidden inputs/expected outputs not returned | LAB-F007/V020 |
| LAB-QA-051 | Problem archived after Lab publish | Lab revision pins published ProblemVersion | Open current published Lab | Existing Lab revision remains tied to pinned ProblemVersion and is available under Lab status/deadline; new revision cannot pin archived Problem | LAB-F007; LAB-D12/D13 |
| LAB-QA-052 | Lab archived | Student opens detail | Request detail | Access follows approved archive/history policy | LAB-F007/D11 |
| LAB-QA-053 | Student submits valid code | Eligible Lab/Problem/language | Submit | CodeSubmission and LabSubmission linkage created; evaluation begins | LAB-F008 |
| LAB-QA-054 | Student submits inaccessible Lab | Foreign Lab | Submit | Deny before CodeSubmission/Judge0 creation | LAB-F008/R20 |
| LAB-QA-055 | Student substitutes arbitrary Problem | Lab links Problem-A; request Problem-B | Submit | Reject/ignore client Problem ID; use linked Problem | LAB-F008/R19 |
| LAB-QA-056 | Student ID spoof | Client sends Student-B ID | Submit as Student-A | Server uses Student-A auth; deny spoof attempt | LAB-F008 |
| LAB-QA-057 | Unsupported language | Language unsupported by Module 6 | Submit | Module 6 validation; no invalid execution | LAB-F008/V023 |
| LAB-QA-058 | Submit after dueDate | Default BLOCK_AFTER_DUE; server receivedAt > dueDate | Submit | Reject before CodeSubmission/Judge0 creation | LAB-F008/F016; LAB-D09/D29/D31 |
| LAB-QA-059 | Attempt limit reached | Revision maxAttempts=2; two accepted attempts | Submit third attempt | Reject; only accepted submissions count; rejudge/idempotent replay do not increment attempt | LAB-F008; LAB-D18/D19/D20 |
| LAB-QA-060 | Duplicate submit click | Same Student, Lab and Idempotency-Key/payload | Retry POST | Return existing accepted CodeSubmission/LabSubmission; do not allocate another attempt | LAB-F008/F009; LAB-D18/D19 |
| LAB-QA-061 | Judge0 outage after accepted submit | CodeSubmission accepted | Execute | Operational failure/recovery, not Student verdict | LAB-F008/F010 |
| LAB-QA-062 | Deadline race | Trusted server receivedAt equals dueDate | Submit at exact boundary | Accept as on-time; comparison is inclusive and uses UTC server timestamp | LAB-F008/F016; LAB-D29/D31 |
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
| LAB-QA-079 | Problem/testcase changed while queued | Submission/LabSubmission pinned to Lab revision and ProblemVersion | Worker evaluates | Use original pinned ProblemVersion snapshot; later Problem versions do not change queued work | LAB-F010; LAB-D13/D27 |
| LAB-QA-080 | Convert 100% evaluation | EvaluationScorePercent=100; maxScore=20 | Finalize score | Persist 20.00 | LAB-F011; LAB-D07/D08 |
| LAB-QA-081 | Convert partial evaluation | EvaluationScorePercent=50; maxScore=20 | Finalize score | Persist 10.00 | LAB-F011; LAB-D07/D08 |
| LAB-QA-082 | Complete code-failure evaluation | Module 7 complete result with 0% or partial percent and a final code verdict | Finalize score | Convert the finalized percentage using the same formula; never infer score from verdict name alone | LAB-F011; LAB-D07 |
| LAB-QA-083 | maxScore zero | Lab maxScore=0 | Calculate | Reject/explicit rule; no divide-by-zero | LAB-F011/V032 |
| LAB-QA-084 | Score exceeds maxScore | invalid evaluation/manual data | Persist | Reject/flag under approved range | LAB-F011/R28 |
| LAB-QA-085 | Rounding boundary | Computed score has >2 decimal places | Finalize score | Round half-up once to 2 decimal places; preserve Module 7 integer percentage and maxScore snapshot | LAB-F011; LAB-D08 |
| LAB-QA-086 | Score precision | maxScore within supported range; result is two-decimal value | Persist/display | Persist DECIMAL(12,2); score remains between 0 and maxScore | LAB-F011; LAB-D04/D08 |
| LAB-QA-087 | Rejudge changes evaluation score | Attempt has prior completed score; SUPER_ADMIN rejudge completes | Consume finalized rejudge result | Recompute same attempt score from pinned revision; append LabScoreHistory; preserve prior history | LAB-F011/F019; LAB-D20/D21 |
| LAB-QA-088 | maxScore changes after prior score | Old revision has finalized submissions | Publish new revision with changed maxScore | Existing scores remain on old snapshot; new attempts use new revision/maxScore | LAB-F011; LAB-D12/D27 |
| LAB-QA-089 | Late penalty not configured | Late submission | Calculate | No penalty assumed/applied | LAB-F011/R44 |
| LAB-QA-090 | Infrastructure failure result | Judge0 failure | Calculate | No fabricated Student grade | LAB-F011/R27 |
| LAB-QA-091 | Student reads own LabSubmission | Student-A owns | GET result | Safe own result returned | LAB-F012 |
| LAB-QA-092 | Student reads Student-B LabSubmission | Foreign ID | GET result | Deny/not-found-safe | LAB-F012/V035 |
| LAB-QA-093 | Student requests raw Judge0 result | Student | GET raw endpoint | Deny or safe DTO only | LAB-F012/R34 |
| LAB-QA-094 | Result pending | Evaluation running | Read | Pending distinct from final | LAB-F012/V036 |
| LAB-QA-095 | Rejudge pending | Current run pending | Read | State/history policy applied; no false final | LAB-F012 |
| LAB-QA-096 | Course enrollment withdrawn after grading | Student owns historical LabSubmission | Read own history | Allow own historical result read-only; block new Lab attempts | LAB-F012; LAB-D14/D15 |
| LAB-QA-097 | Lab archived but Student owns prior score | Prior LabSubmission exists | Read My Labs/history | Show historical record/score read-only; no new attempts | LAB-F012; LAB-D11/D15 |
| LAB-QA-098 | HOD reviews Department Lab results | Scoped Students | Open Student Performance | Only College/Department records | LAB-F013 |
| LAB-QA-099 | HOD reviews foreign Department | Foreign record | Direct ID/filter | Deny | LAB-F013 |
| LAB-QA-100 | Manager reviews assigned Section | Section-A | Open performance page | Permitted Section records only | LAB-F013 |
| LAB-QA-101 | Manager reviews foreign Section | Section-B | Direct ID/filter | Deny | LAB-F013 |
| LAB-QA-102 | Finance Team views Student scores | Finance Team assigned College | Open Student Performance/detail API | Deny all Student-level submission, score, feedback, source and evaluation detail | LAB-F013; LAB-D06/D23/D36 |
| LAB-QA-103 | Staff has View but requests raw JudgeRun | Staff role | Raw endpoint | Deny absent separate permission | LAB-F013/R37 |
| LAB-QA-104 | Staff exports Lab scores | No export permission | Export action | Deny; export not implied by View | LAB-F013/V038 |
| LAB-QA-105 | Staff filters by foreign Student | Out-of-scope ID | Query | Deny/scope-filter | LAB-F013/V037 |
| LAB-QA-106 | Staff writes feedback | Scoped submission | Save feedback | Persist feedback | LAB-F014 |
| LAB-QA-107 | Staff feedback to foreign submission | Out-of-scope ID | Save | Deny | LAB-F014/V039 |
| LAB-QA-108 | Student reads own feedback | Student-A | View result | Feedback returned | LAB-F014 |
| LAB-QA-109 | Student reads another Student feedback | Student-B | Direct ID | Deny | LAB-F014 |
| LAB-QA-110 | Feedback exceeds length limit | Feedback exceeds 10,000 characters after trim | Save feedback | Reject; maximum is 10,000 characters | LAB-F014; LAB-D28 |
| LAB-QA-111 | Feedback cleared/blank | Existing feedback exists | Submit blank feedback | Clear feedback to null; only scoped HOD/Manager may update | LAB-F014; LAB-D28 |
| LAB-QA-112 | Manual override feature unapproved | No policy | Attempt override | Feature unavailable/denied | LAB-F015 |
| LAB-QA-113 | Override by unauthorized role | No grading permission | Override score | Deny | LAB-F015/V043 |
| LAB-QA-114 | Manual override attempt with reason | Manual score override disabled in v1 | Call score adjustment endpoint | Deny; no score override endpoint/control is available in v1 | LAB-F015; LAB-D22/D34 |
| LAB-QA-115 | Override without reason | Reason required | Save | Reject | LAB-F015/V042 |
| LAB-QA-116 | Override exceeds maxScore | Approved override | Save | Reject unless bonus policy allows | LAB-F015/R41 |
| LAB-QA-117 | Concurrent overrides | Two graders | Save competing changes | Conflict/review; no silent overwrite | LAB-F015/V044 |
| LAB-QA-118 | Rejudge after manual override | No manual score override allowed in v1 | Attempt override then rejudge | Override remains unavailable; only authorized Module 6/7 rejudge may recompute score | LAB-F015/F019; LAB-D20/D22/D34 |
| LAB-QA-119 | Submit before dueDate | dueDate set in UTC; receivedAt < dueDate | Submit | Accept as on-time if all other checks pass | LAB-F016; LAB-D09/D29/D31 |
| LAB-QA-120 | Submit exactly at dueDate | receivedAt equals dueDate | Submit | Accept as on-time; dueDate is inclusive | LAB-F016; LAB-D29/D31 |
| LAB-QA-121 | Submit after dueDate | Default BLOCK_AFTER_DUE; receivedAt > dueDate | Submit | Reject; if revision is ACCEPT_NO_PENALTY, accept as late and apply no penalty | LAB-F016; LAB-D09/D29 |
| LAB-QA-122 | Judge0 completes after dueDate | Submitted before deadline | Evaluate | Late classification based on submission time under derived rule | LAB-F016/R43 |
| LAB-QA-123 | Staff individual extension | Student-specific extension requested | Staff attempts extension | Reject/defer; per-Student extension is not supported in v1. A global due-date change requires a new revision | LAB-F016; LAB-D30 |
| LAB-QA-124 | Timezone/DST boundary | UTC dueDate and different Student locale | Display/submit | Store/compare UTC; localize only display; DST/client clock does not affect server decision | LAB-F016; LAB-D29/D31 |
| LAB-QA-125 | DueDate edited after graded submission | Existing score | Update deadline | Historic lateness not silently recalculated | LAB-F016/V047 |
| LAB-QA-126 | Client sends forged submittedAt | Student request | Submit | Server timestamp is authoritative per decision | LAB-F016/D31 |
| LAB-QA-127 | Student enrolled and in assigned Group | Eligible relationships | Request Lab | Access allowed | LAB-F017 |
| LAB-QA-128 | GroupMember but not enrolled | Student has GroupMember but no CourseEnrollment.status=ENROLLED | Request Lab | Deny; active CourseEnrollment is mandatory | LAB-F017; LAB-D14 |
| LAB-QA-129 | Enrolled but not GroupMember | Current Lab has GroupLab links; Student has no membership in any linked Group | Request Lab | Deny; Group membership is an additional filter when GroupLab links exist | LAB-F017; LAB-D05/D14 |
| LAB-QA-130 | Student not in Section scope | SECTION_ONLY Lab | Request | Deny under approved scope definition | LAB-F017/D32 |
| LAB-QA-131 | DEPT_WIDE Lab | Department Student | Request | Access under approved semantics; enum name alone insufficient | LAB-F017/D32 |
| LAB-QA-132 | GLOBAL Lab in scoped Course | Lab/Problem GLOBAL; Student in another College | Request Lab | Deny; GLOBAL is restricted to active Sections in owning College and all scopes intersect | LAB-F017; LAB-D14/D32 |
| LAB-QA-133 | Student removed from Group | Student has prior LabSubmission but current GroupMember absent | Request new Lab attempt and view history | Deny new attempt; retain owner-only historical record | LAB-F017; LAB-D15 |
| LAB-QA-134 | Group inactive | Group.isActive=false | Student requests linked Lab | Deny new access/submission through inactive Group | LAB-F017; LAB-D05/D14 |
| LAB-QA-135 | CourseEnrollment.completedAt set | CourseEnrollment.status=COMPLETED | Request Lab | Allow historical/read access only; no new attempt unless a new ENROLLED state is explicitly established by Module 4 | LAB-F017; LAB-D14 |
| LAB-QA-136 | Enrollment revoked | Student no longer enrolled | Request | Deny current access unless approved historical access | LAB-F017 |
| LAB-QA-137 | Cross-College relation | foreign Course/Group | Request | Deny unless explicit global exception approved | LAB-F017 |
| LAB-QA-138 | Student has not submitted | Eligible Lab | View progress | Display not-submitted only if status semantics approved | LAB-F018/D33 |
| LAB-QA-139 | Student submitted, evaluation pending | LabSubmission exists | View progress | Submitted/pending; not completed | LAB-F018/R50 |
| LAB-QA-140 | Evaluation complete, score below 50% | Complete Module 7 evaluation; no configured pass threshold | View progress | Lab is COMPLETED because evaluation is finalized; score does not gate completion | LAB-F018; LAB-D17/D33 |
| LAB-QA-141 | CourseEnrollment.completedAt set | Course completed | View Lab progress | Must not imply Lab completion | LAB-F018/R48 |
| LAB-QA-142 | Multiple attempts | Several completed LabSubmissions with different scores | View progress | Show latest attempt and best score; best is highest score, tie uses latest accepted attempt | LAB-F018; LAB-D18/D33 |
| LAB-QA-143 | Lab archived | Student has prior LabSubmission | View Dashboard/My Labs | Historical Lab and own result remain read-only | LAB-F018; LAB-D11/D15 |
| LAB-QA-144 | Manual score adjusted | Override disabled in v1 | View progress | No manual score change exists; progress derives from accepted evaluation only | LAB-F018; LAB-D17/D22/D33 |
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

| LAB-QA-159 | Published Lab pins ProblemVersion | Problem V1 is published; Lab revision R1 created | Publish Lab revision | R1 stores ProblemVersion V1; no floating latest-Problem reference | LAB-F004/F007; LAB-D12/D13 |
| LAB-QA-160 | New Problem version requires new Lab revision | Lab R1 pins Problem V1; Problem V2 published later | Attempt to mutate R1 | R1 remains V1; assigning V2 requires a new Lab revision and publish | LAB-F007; LAB-D12/D13 |
| LAB-QA-161 | Exactly one Problem per Lab | Create request attempts multiple Problem IDs | Save Lab | Reject extra Problem associations; v1 LabAssignment has one problemId | LAB-F002; LAB-D03 |
| LAB-QA-162 | Group assignment is optional | Eligible enrolled Course Student; no GroupLab links | Open My Labs | Student can access if Section/Course/Lab/Problem scopes pass | LAB-F005/F006; LAB-D05/D14 |
| LAB-QA-163 | Group membership is an additional filter | GroupLab links exist; Student enrolled but belongs to no linked Group | Open Lab/submit | Deny; Group membership does not replace enrollment or assigned Group targeting | LAB-F017; LAB-D05/D14 |
| LAB-QA-164 | Inactive Group does not grant access | GroupLab exists; Group.isActive=false | Student opens Lab | Deny new Lab access/attempts through inactive Group | LAB-F017; LAB-D05/D14 |
| LAB-QA-165 | Removed Group member keeps own history | Student has accepted attempt then is removed from Group | Open My Labs/history and submit | Own history remains read-only; new submissions denied | LAB-F006/F012; LAB-D15 |
| LAB-QA-166 | Late acceptance option | Revision policy ACCEPT_NO_PENALTY; request arrives after dueDate | Submit code | Accept as late, store isLate=true and apply no score penalty | LAB-F008/F016; LAB-D09/D29 |
| LAB-QA-167 | No dueDate means no deadline | Published Lab has dueDate=null and is not CLOSED | Submit | No date cutoff applies; all other availability/attempt checks still apply | LAB-F016; LAB-D09 |
| LAB-QA-168 | Unlimited attempts default | Revision maxAttempts=null | Submit multiple accepted attempts sequentially | Allow attempts within Module 6 rate guard and one-in-flight-per-Lab rule | LAB-F008; LAB-D18 |
| LAB-QA-169 | One in-flight attempt per Student/Lab | A Student has an active SubmissionRun for Lab A | Submit again with a new key | Reject/serialize until current run resolves; do not allocate concurrent attempt | LAB-F008; LAB-D18 |
| LAB-QA-170 | Highest finalized score selected | Attempt 1=8/10; attempt 2=10/10 | Read best score | Best score is 10/10; both attempts remain visible | LAB-F018; LAB-D18 |
| LAB-QA-171 | Tie selects latest attempt | Two finalized attempts have equal score | Read best/current selection | Latest accepted attempt wins tie; older attempt remains in history | LAB-F018; LAB-D18 |
| LAB-QA-172 | Rejudge does not increase attempt number | Attempt 2 is rejudged | Inspect LabSubmission history | Same LabSubmission/attemptNumber; new SubmissionRun only | LAB-F019; LAB-D20 |
| LAB-QA-173 | Incomplete evaluation has no Lab score | SubmissionRun has infrastructure FAILED/pending state | Process Lab result | LabSubmission.score remains null; progress is not COMPLETED | LAB-F010/F011/F018; LAB-D07/D17 |
| LAB-QA-174 | Zero-percent complete evaluation | Module 7 completed with scorePercent=0 | Finalize Lab score | Persist 0.00; mark evaluation completion, not pass/fail threshold | LAB-F011/F018; LAB-D07/D17 |
| LAB-QA-175 | Rejudge recomputes score with original maxScore | Submission pinned Lab revision R1 maxScore=20; current R2 maxScore=30 | Rejudge original submission | Recompute against R1 maxScore=20 and original ProblemVersion | LAB-F019; LAB-D20/D21/D27 |
| LAB-QA-176 | Failed rejudge preserves current score | Completed Lab score exists; rejudge fails operationally | Consume failure event | Keep last completed LabSubmission.score; no fabricated score change | LAB-F019; LAB-D21 |
| LAB-QA-177 | Manual score override blocked | HOD or Manager attempts score override | Call override endpoint | Deny; manual override is not part of v1 | LAB-F015; LAB-D22 |
| LAB-QA-178 | Score formula exactness | EvaluationScorePercent=75; maxScore=40 | Finalize score | Persist 30.00 and store formula/revision provenance | LAB-F011; LAB-D07/D08/D26 |
| LAB-QA-179 | Half-up score precision | Computed Lab score is 12.345 from approved input precision | Finalize score | Persist 12.35; round half-up only once at finalization | LAB-F011; LAB-D08 |
| LAB-QA-180 | Student cannot read foreign LabSubmission | Student B requests Student A's LabSubmission ID | GET result | Deny/not-found-safe; no foreign metadata or score leak | LAB-F012; LAB-D23 |
| LAB-QA-181 | Manager cannot archive/delete | Manager has Section-scoped Lab | Request archive/delete | Deny; HOD/SUPER_ADMIN own lifecycle actions; no hard delete | LAB-F003/F004; LAB-D06/D11 |
| LAB-QA-182 | Finance metadata response allowlist | Finance Team views College Lab | GET Lab list/detail | Only Lab/Course/Problem metadata, status, maxScore and dueDate; no Student fields | LAB-F001/F013; LAB-D06/D36 |
| LAB-QA-183 | Cross-College GLOBAL isolation | Lab scope GLOBAL in College A; Student is in College B | GET/submit | Deny; GLOBAL is College-local, not platform-public | LAB-F017; LAB-D14/D32 |
| LAB-QA-184 | Lab assignment revision snapshots | Lab R1 published; R2 changes dueDate/maxScore | Inspect submissions for R1/R2 | Each LabSubmission references correct revision snapshot | LAB-F003/F009; LAB-D12/D27 |
| LAB-QA-185 | Duplicate LabSubmission link event | Same CodeSubmission accepted event delivered twice | Create link | Unique codeSubmissionId/idempotent consumer produces one LabSubmission | LAB-F009/F020; LAB-D19/D35 |
| LAB-QA-186 | Outbox duplicate score delivery | Same SubmissionGraded event delivered twice | Consume event | One score update/history record; dedupe eventId/sourceRunId | LAB-F020; LAB-D21/D35 |
| LAB-QA-187 | Course/Problem scope intersection | Course audience narrower than Lab/Problem scope | Resolve Student eligibility | Use intersection; a broader Lab/Problem scope never expands Course audience | LAB-F017; LAB-D14/D32 |
| LAB-QA-188 | HOD lifecycle scope | HOD has Department A; Lab references Department B | Publish/archive Lab | Deny cross-Department action; HOD remains Department-bound | LAB-F004/F005; LAB-D06/D32 |

## QA review notes
- LAB-QA-001–188 trace to LAB-F001–LAB-F020 and LAB-D01–LAB-D38.
- All cases define adopted target expectations; none are marked passed until executed against the implementation with evidence.
- Source/legacy audit findings remain historical evidence and are not target implementation proof.
- Include migration, tenant isolation, idempotency, concurrency, rejudge, outbox and history-reconciliation tests in release validation.
