# Module 7 — Test Case Evaluation QA Test Cases

**Version:** 0.1 — QA companion to Functional Specification v0.2  
**Status:** Draft; target decisions remain open  
**Specification:** [Module 7 — Test Case Evaluation](./README.md)

## Test conventions
- Submission-S = Student-owned submission; Submission-F = foreign Student submission.
- Problem-A and Problem-B are scoped to different Colleges/Departments.
- Sample = legacy isSample=true; hidden = legacy isSample=false. Conceptual isHidden mapping must be verified.
- **Decision-blocked** means expected target behavior cannot be finalized until referenced EVAL-D decision is approved.
- **Audit verification** checks reported legacy behavior only; it does not approve that behavior for target.

| Test ID | Scenario | Preconditions/data | Steps | Expected result | Trace/status |
|---|---|---|---|---|---|
| EVAL-QA-001 | Resolve valid Problem testcase set | Problem-A with samples + hidden cases | Start evaluation job | All approved official cases identified | TEST-F001 |
| EVAL-QA-002 | Problem ID missing | Submission references no Problem | Start evaluation | Configuration error; not Accepted | TEST-F001 |
| EVAL-QA-003 | Problem has no testcases | Empty set | Start evaluation | Approved empty-set behavior; never accidental full score | TEST-F001/D09 |
| EVAL-QA-004 | Testcase belongs to another Problem | mixed relation | Load set | Reject/ignore foreign testcase per schema relation | TEST-F001 |
| EVAL-QA-005 | Hidden cases excluded from worker | hidden case exists | Evaluate formal submission | Verify all official hidden cases included if policy requires | TEST-F001/F010 |
| EVAL-QA-006 | Problem edited while queued | testcase added/changed before worker | Submit then edit then run | Snapshot policy determines run; decision-blocked | TEST-F002/D20 |
| EVAL-QA-007 | Testcase deleted before worker | queued submission | delete testcase then evaluate | Version policy; no silent missing case | TEST-F002/D20 |
| EVAL-QA-008 | Testcase order changes before evaluation | queued submission | reorder then run | Approved snapshot/order policy | TEST-F002/F023 |
| EVAL-QA-009 | Problem version unavailable | submission references prior version | run | Fail/recover per version policy | TEST-F002 |
| EVAL-QA-010 | Run result stores testcase set/version | versioning enabled | inspect run | Version/set identity persisted | TEST-F002/D20 |
| EVAL-QA-011 | Build Judge0 payload per testcase | N testcases | worker builds batch | Exactly one execution request per case if that design approved | TEST-F003 |
| EVAL-QA-012 | Source code encoding | Unicode code | build payload | Correct encoding/decoding | TEST-F003 |
| EVAL-QA-013 | Testcase stdin encoding | input with Unicode/newlines | build payload | Input preserved per Judge0 contract | TEST-F003 |
| EVAL-QA-014 | Expected output encoding | output with Unicode/newlines | build payload | Expected output preserved | TEST-F003 |
| EVAL-QA-015 | Missing testcase input | malformed testcase | build payload | Validation/config failure | TEST-F003 |
| EVAL-QA-016 | Missing expected output | malformed testcase | build payload | Follow explicit null/empty rule | TEST-F003/D15 |
| EVAL-QA-017 | Invalid language ID | unsupported language | build batch | Reject before Judge0 request | TEST-F003/Module 6 |
| EVAL-QA-018 | Resource limit invalid | negative/oversized limit | build batch | Reject config; do not use client override | TEST-F003 |
| EVAL-QA-019 | Empty batch | zero cases | batch submit | Explicit empty set handling | TEST-F003/D09 |
| EVAL-QA-020 | Very large batch | many cases | build batch | Approved batch-size/capacity behavior | TEST-F003/D19 |
| EVAL-QA-021 | Batch request accepted | valid N-case batch | POST Judge0 | Token set returned and validated | TEST-F004 |
| EVAL-QA-022 | Judge0 non-2xx response | 500/503 | submit batch | Infrastructure failure; not Wrong Answer | TEST-F004 |
| EVAL-QA-023 | Judge0 rate limit | 429 | submit batch | Retry/backoff per approved policy | TEST-F004/D19 |
| EVAL-QA-024 | Malformed JSON response | invalid response | submit batch | Correlation failure | TEST-F004 |
| EVAL-QA-025 | Token array length too short | N expected, N-1 tokens | submit batch | Incomplete/operational error | TEST-F004 |
| EVAL-QA-026 | Token array length too long | N expected, N+1 tokens | submit batch | Detect mismatch | TEST-F004 |
| EVAL-QA-027 | Duplicate Judge0 token | token repeated | submit batch | Reject/flag invalid mapping | TEST-F004 |
| EVAL-QA-028 | Partial batch acceptance | some accepted, some rejected | submit batch | Track per-token outcomes; do not grade missing cases as WA | TEST-F004 |
| EVAL-QA-029 | Request timeout after Judge0 accepted | timeout with token creation possible | retry | Dedup/reconcile external jobs per policy | TEST-F004/D19 |
| EVAL-QA-030 | Token-to-case mapping | cases A/B/C, tokens x/y/z | inspect run map | Each token mapped to exact case | TEST-F005 |
| EVAL-QA-031 | Judge0 returns tokens reordered | response order differs | correlate | Match by stable mapping; no wrong association | TEST-F005 |
| EVAL-QA-032 | Result token unknown | unexpected token | poll result | Reject/flag | TEST-F005 |
| EVAL-QA-033 | Result missing token | response no token | poll result | Incomplete correlation | TEST-F005 |
| EVAL-QA-034 | Retry uses new tokens | failed first job | retry | Run generation keeps tokens distinct | TEST-F005 |
| EVAL-QA-035 | Duplicate token result | duplicate result item | aggregate | No double scoring | TEST-F005 |
| EVAL-QA-036 | All Judge0 tokens terminal | accepted/wrong outcomes | poll | Stop polling and process full result set | TEST-F006 |
| EVAL-QA-037 | Tokens remain queued | in_queue status | poll | Continue until approved timeout | TEST-F006 |
| EVAL-QA-038 | Tokens remain processing | processing status | poll | Do not treat as terminal verdict | TEST-F006 |
| EVAL-QA-039 | Polling timeout | token still processing | wait deadline | Incomplete evaluation status | TEST-F006/D19 |
| EVAL-QA-040 | Poll request network error | transient failure | poll | Retry/recover per policy | TEST-F006 |
| EVAL-QA-041 | Poll response partial | only some tokens returned | poll | Mark missing results incomplete | TEST-F006 |
| EVAL-QA-042 | Polling backoff | repeated pending | inspect intervals | Approved backoff/timeout applied | TEST-F006 |
| EVAL-QA-043 | Judge0 token expired | expired token | poll | Operational failure, not WA | TEST-F006 |
| EVAL-QA-044 | Judge0 status Accepted (3) | one testcase | map result | Pass according to approved mapping | TEST-F007 |
| EVAL-QA-045 | Wrong Answer status (4) | one testcase | map | Failure outcome | TEST-F007 |
| EVAL-QA-046 | TLE status (5) | one testcase | map | TLE mapping | TEST-F007 |
| EVAL-QA-047 | Compilation Error status (6) | compile failure | map | CE mapping | TEST-F007 |
| EVAL-QA-048 | Runtime status 7–11 | runtime failure | map | RE mapping per approved table | TEST-F007 |
| EVAL-QA-049 | MLE status (12) | memory failure | map | MLE mapping | TEST-F007 |
| EVAL-QA-050 | Unknown Judge0 status ID | future status | map | Unknown/operational; never pass | TEST-F007/D13 |
| EVAL-QA-051 | Judge0 status description conflicts with ID | mismatched response | map | Canonical ID mapping governs per policy | TEST-F007 |
| EVAL-QA-052 | Queued status passed to verdict mapper | nonterminal | map | Not final result | TEST-F007 |
| EVAL-QA-053 | Output exactly matches | expected=actual | evaluate | Accepted | TEST-F008 |
| EVAL-QA-054 | Output mismatch | different output | evaluate | Wrong Answer according to comparator | TEST-F008 |
| EVAL-QA-055 | Trailing whitespace difference | outputs differ whitespace | evaluate | Deployed comparator behavior verified; target policy TBD | TEST-F008/D15 |
| EVAL-QA-056 | Newline/CRLF difference | outputs line ending mismatch | evaluate | Explicit comparator rule | TEST-F008/D15 |
| EVAL-QA-057 | Case-sensitive mismatch | upper/lowercase | evaluate | Comparator policy applies | TEST-F008/D15 |
| EVAL-QA-058 | Floating-point precision mismatch | 1.0 vs 1.00001 | evaluate | No tolerance assumed; policy required | TEST-F008/D15 |
| EVAL-QA-059 | Empty expected output | expected blank | evaluate | Explicit empty output rule | TEST-F008/D15 |
| EVAL-QA-060 | Custom comparator request | unsupported comparator | configure Problem | Reject/disable until approved | TEST-F008/D16 |
| EVAL-QA-061 | Sample testcase executed | isSample true | evaluate | Included per approved evaluation policy | TEST-F009 |
| EVAL-QA-062 | Sample actual/expected shown | sample failure | Student view | Show only approved sample details | TEST-F009 |
| EVAL-QA-063 | isSample vs isHidden conversion | legacy/conceptual schema | map field | Explicit mapping; no silent inverse | TEST-F009/D10 |
| EVAL-QA-064 | Missing visibility flag | testcase malformed | serialize | Restrictive visibility default | TEST-F009/F011 |
| EVAL-QA-065 | Hidden testcase contributes to score | hidden case fails | evaluate | Counts under approved official-case policy | TEST-F010 |
| EVAL-QA-066 | Hidden input returned by Problem API | hidden fixture | GET Problem | Must not be exposed | TEST-F010/F011 |
| EVAL-QA-067 | Hidden expected output returned in result | hidden case | GET result | Must not be exposed | TEST-F011 |
| EVAL-QA-068 | Hidden actual output returned | hidden case | GET result | Must not be exposed | TEST-F011 |
| EVAL-QA-069 | Hidden stderr leaks input | code prints input to stderr | Student result | Mask/filter per approved policy | TEST-F011/D12 |
| EVAL-QA-070 | Hidden compile output leaks details | hidden testcase CE | Student result | Mask/filter per approved policy | TEST-F011/D12 |
| EVAL-QA-071 | Hidden result via WebSocket | grading push | inspect payload | Same masking as REST | TEST-F011 |
| EVAL-QA-072 | Hidden result via direct API | Student bypasses UI | request endpoint | Server masking/authorization | TEST-F011 |
| EVAL-QA-073 | Hidden values in application logs | execution failure | inspect logs | No unauthorized secret payload | TEST-F011/F024 |
| EVAL-QA-074 | Unknown visibility class | missing flag | serialize | Fail closed | TEST-F011 |
| EVAL-QA-075 | Persist JudgeRun row | valid testcase result | worker saves | Row linked to submission/run/case | TEST-F012 |
| EVAL-QA-076 | DB failure persisting JudgeRun | DB outage | save | Recover/retry; aggregate not falsely finalized | TEST-F012 |
| EVAL-QA-077 | Retry duplicate raw run | worker retry | persist | Run attempt identity prevents accidental overwrite | TEST-F012 |
| EVAL-QA-078 | Raw JudgeRun access by Student | Student | raw endpoint | Deny unless safe DTO explicitly allowed | TEST-F012/F019 |
| EVAL-QA-079 | Raw Judge0 token in response | result serialization | inspect | Token not exposed to Student | TEST-F012/F024 |
| EVAL-QA-080 | Result DTO fields | completed testcase | build DTO | Approved fields/units only | TEST-F013 |
| EVAL-QA-081 | Runtime unit conversion | Judge0 time in seconds | DTO | Milliseconds conversion consistent | TEST-F013 |
| EVAL-QA-082 | Memory units | Judge0 memory KB | DTO | KB/MB display conversion correct | TEST-F013 |
| EVAL-QA-083 | Missing runtime value | null time | DTO | Missing/zero representation approved | TEST-F013 |
| EVAL-QA-084 | Case index one-based | N cases | render | Display index matches approved ordinal | TEST-F013/D11 |
| EVAL-QA-085 | Testcase reorder | reorder cases after result | render history | Stable case identity preserved | TEST-F013/F023 |
| EVAL-QA-086 | Weight defaults to 1 | legacy weight omitted | compute | Audit verification; target default decision | TEST-F014/D10 |
| EVAL-QA-087 | All equal-weight cases pass | N accepted | score | 100% under approved formula | TEST-F014 |
| EVAL-QA-088 | Half testcases pass | equal weights | score | Formula/rounding matches approved rule | TEST-F014 |
| EVAL-QA-089 | Unequal testcase weights | weights approved | score | Weighted calculation correct | TEST-F014/D18 |
| EVAL-QA-090 | Zero total weight | all weights zero | score | Explicit error/fallback; not accidental | TEST-F014/D18 |
| EVAL-QA-091 | Negative/invalid weight | invalid schema data | score | Reject config | TEST-F014/D10 |
| EVAL-QA-092 | Missing testcase result | N expected, N-1 results | score | Incomplete; no implicit pass | TEST-F014/F016 |
| EVAL-QA-093 | Duplicate testcase result | duplicate result | score | No duplicate credit | TEST-F014 |
| EVAL-QA-094 | Score rounding boundary | score x.5 | calculate | Rounding follows approved Math policy | TEST-F014/D18 |
| EVAL-QA-095 | Score precision | decimal scoring proposed | calculate | Target precision decision | TEST-F014/D18 |
| EVAL-QA-096 | All accepted statuses | all cases pass | aggregate | ACCEPTED under policy | TEST-F015 |
| EVAL-QA-097 | Single WA | one WA | aggregate | WA or approved aggregate verdict | TEST-F015 |
| EVAL-QA-098 | Mixed TLE and WA | both fail | aggregate | Approved precedence, not accidental array order | TEST-F015/D18 |
| EVAL-QA-099 | Mixed MLE and CE | multiple failure types | aggregate | Approved precedence | TEST-F015/D18 |
| EVAL-QA-100 | Mixed RE and WA | mixed outcomes | aggregate | Approved precedence | TEST-F015/D18 |
| EVAL-QA-101 | Partial score and failed verdict | several pass/fail | aggregate | Partial verdict/score semantics approved | TEST-F015/D18 |
| EVAL-QA-102 | Legacy first-failure behavior | legacy code snapshot | audit test | Verify reported behavior; not target approval | TEST-F015/Audit |
| EVAL-QA-103 | Legacy PARTIAL branch | mixed pass/fail | audit test | Confirm branch reachability from report/source | TEST-F015/Audit |
| EVAL-QA-104 | Unknown status in result set | unknown status | aggregate | Never ACCEPTED | TEST-F015 |
| EVAL-QA-105 | Empty results array | no results | aggregate | Error/incomplete per policy | TEST-F015/D09 |
| EVAL-QA-106 | Infrastructure error | Judge0 unavailable | aggregate | Not student WA/RE by default | TEST-F016 |
| EVAL-QA-107 | Partial polling result | one token incomplete | aggregate | Incomplete state; no full normal grade | TEST-F016 |
| EVAL-QA-108 | Late result arrives after retry | previous run late | aggregate | Current generation/run policy prevents overwrite | TEST-F016 |
| EVAL-QA-109 | Missing Judge0 token | token missing | aggregate | Recoverable/operational error | TEST-F016 |
| EVAL-QA-110 | Result count greater than testcase count | malformed result list | aggregate | Reject correlation | TEST-F016 |
| EVAL-QA-111 | Update aggregate submission | complete valid result | persist | Aggregate fields match testcase outcomes | TEST-F017 |
| EVAL-QA-112 | Stale worker result | old generation | update | Cannot overwrite current run | TEST-F017 |
| EVAL-QA-113 | DB failure after judge complete | final write fails | worker | Retry/recovery without duplicate grading side effect | TEST-F017 |
| EVAL-QA-114 | Event emitted before commit | transaction scenario | worker | Prevent; event follows committed result | TEST-F017/F022 |
| EVAL-QA-115 | Student reads own result | Student-S owns record | GET | Safe DTO returned | TEST-F018 |
| EVAL-QA-116 | Student reads foreign result | Student-F owns record | GET | Deny/not-found safely | TEST-F018 |
| EVAL-QA-117 | Student result while pending | pending submission | GET | Pending distinct from completed | TEST-F018 |
| EVAL-QA-118 | Student result while rerun pending | rejudge underway | GET | Current generation/status shown | TEST-F018 |
| EVAL-QA-119 | HOD reads Department result | authorized | GET | Scope-filtered result | TEST-F019 |
| EVAL-QA-120 | HOD reads foreign Department result | foreign record | GET | Deny | TEST-F019 |
| EVAL-QA-121 | Manager reads assigned Section | permitted record | GET | Allowed fields only | TEST-F019 |
| EVAL-QA-122 | Manager reads foreign Section | foreign record | GET | Deny | TEST-F019 |
| EVAL-QA-123 | Finance views result | Finance role | GET | Deny absent grant | TEST-F019 |
| EVAL-QA-124 | Staff raw output access | role has view but no raw grant | GET raw | Deny raw field | TEST-F019 |
| EVAL-QA-125 | Student requests rerun | Student | POST rerun | Deny | TEST-F020 |
| EVAL-QA-126 | Unauthorized staff rerun | no permission | POST rerun | Deny | TEST-F020 |
| EVAL-QA-127 | Authorized legacy faculty rerun | legacy audit context | audit verify | Legacy permission mapping recorded, not target role equivalence | TEST-F020 |
| EVAL-QA-128 | Rerun active submission | existing run active | rerun | Serialize/reject per decision | TEST-F020 |
| EVAL-QA-129 | Rerun changes aggregate | existing result | rerun | Behavior follows approved overwrite/version policy | TEST-F020 |
| EVAL-QA-130 | Rerun with changed testcase | Problem modified | rerun | Approved version policy | TEST-F020 |
| EVAL-QA-131 | Rerun fails after reset | queue/Judge0 error | rerun | Old/current result recovery policy | TEST-F020 |
| EVAL-QA-132 | Rejudge does not consume attempt | existing attempt | rerun | Attempt count unchanged unless owner says otherwise | TEST-F020 |
| EVAL-QA-133 | Previous JudgeRun rows retained | multiple runs | inspect history | History behavior follows policy; legacy audit says preserved | TEST-F021 |
| EVAL-QA-134 | Run generations distinguish old/new | multiple reruns | retrieve | Correct current result and history labels | TEST-F021 |
| EVAL-QA-135 | Partial rejudge history | incomplete rerun | inspect | No misleading completed run | TEST-F021 |
| EVAL-QA-136 | Grading event published | committed result | inspect outbox | Canonical event generated once | TEST-F022 |
| EVAL-QA-137 | Duplicate SubmissionGraded event | same event replay | consumer | Idempotent side effects | TEST-F022 |
| EVAL-QA-138 | Course progress event mismatch | legacy report | integration test | Target event contract resolves mismatch | TEST-F022 |
| EVAL-QA-139 | Contest score consumer | Contest context | graded event | Module 11 rules apply; no assumption of latest score only | TEST-F022 |
| EVAL-QA-140 | Lab score consumer | Lab context | graded event | Module 8 contract applies | TEST-F022 |
| EVAL-QA-141 | Notification event | policy enabled | grading event | Module 13 trigger/recipient policy | TEST-F022 |
| EVAL-QA-142 | Audit event on rejudge | audit policy | rerun | Required actor/target/reason event | TEST-F022 |
| EVAL-QA-143 | Conceptual orderIndex | TestCase records | evaluate/display | Deterministic ordering | TEST-F023 |
| EVAL-QA-144 | Legacy no orderBy report | audit snapshot | inspect retrieval | Treat as audit observation; target must specify order | TEST-F023 |
| EVAL-QA-145 | failedTestCaseIndex | conceptual schema | result API | Field meaning/index base resolved before use | TEST-F023 |
| EVAL-QA-146 | Stable testcase ID | reorder | inspect result history | Result remains tied to original case identity | TEST-F023 |
| EVAL-QA-147 | Raw output retained | run completed | inspect access policy | Retention/access approved | TEST-F024 |
| EVAL-QA-148 | stdout/stderr large | huge output | execute | Approved output cap/truncation enforced | TEST-F024 |
| EVAL-QA-149 | Student code prints hidden input to stderr | hidden case | view Student result | Hidden details masked | TEST-F024 |
| EVAL-QA-150 | Compiler output includes hidden data | CE hidden case | view Student result | Safe compiler diagnostics only | TEST-F024 |
| EVAL-QA-151 | Raw Judge0 payload includes secret | response contains token | inspect Student API | Token/secret redacted | TEST-F024 |
| EVAL-QA-152 | Log contains source code | execution failure | inspect logs | Logging policy/redaction applied | TEST-F024 |
| EVAL-QA-153 | Stuck pending evaluation | job overdue | monitor | Alert/recovery policy invoked | TEST-F025 |
| EVAL-QA-154 | Stuck processing token | Judge0 never terminal | monitor | Operational recovery | TEST-F025 |
| EVAL-QA-155 | Queue backlog | many jobs | monitor | Threshold and fairness policy | TEST-F025 |
| EVAL-QA-156 | Retry exhaustion | worker repeatedly fails | monitor | Operational status/alert; not WA | TEST-F025 |
| EVAL-QA-157 | Judge0 outage | endpoint unavailable | monitor | Incident visibility/recovery | TEST-F025 |
| EVAL-QA-158 | Monitoring payload contains code | job metrics | inspect | No sensitive source/testcase data in metrics | TEST-F025 |
| EVAL-QA-159 | Runtime CPU limit from Problem | Problem timeLimitMs | execute | Correct units and source | TEST-F003/Module 5 |
| EVAL-QA-160 | Assessment duration mistakenly used as CPU limit | assessment timeLimitMinutes | audit regression test | Must not apply assessment wall duration as testcase CPU seconds | TEST-F003/Module 9 |
| EVAL-QA-161 | Memory limit unit conversion | Problem memory limit | execute | MB/KB conversion approved and correct | TEST-F003 |
| EVAL-QA-162 | Judge0 external sandbox assumption | execution config | inspect | Sandbox responsibility/config documented; not presumed by DEZOLVER alone | TEST-F003 |
| EVAL-QA-163 | Package/import support | Student code imports package | execute | Only supported Judge0 runtime packages; policy documented | TEST-F003 |
| EVAL-QA-164 | Network/filesystem access | Student program | execute | External Judge0 sandbox configuration verified | TEST-F003 |
| EVAL-QA-165 | Custom checker absent | custom comparator request | configure | Not available unless approved and implemented | TEST-F008 |
| EVAL-QA-166 | Float tolerance absent in audit | floating outputs | audit verify | No feature claim unless target supports it | TEST-F008 |
| EVAL-QA-167 | Result push via WebSocket | completion event | client receives | Safe DTO and correct recipient | TEST-F018/F022 |
| EVAL-QA-168 | Poll fallback | socket disconnected | client polls | Eventually consistent result | TEST-F018 |
| EVAL-QA-169 | Result event after rejudge | new generation | socket/poll | Current result identified; stale event ignored | TEST-F021/F022 |
| EVAL-QA-170 | Role changes after grading | actor permission revoked | GET result | Current access policy enforced | TEST-F019 |

## QA review notes
- EVAL-QA-001–170 trace to TEST-F001–F025.
- Decision-blocked tests must be revisited after product/architecture approval.
- Legacy audit verification is not target acceptance.
