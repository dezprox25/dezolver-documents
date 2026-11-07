# Module 6 — Code Submission & Judge0 QA Test Cases

**Version:** 1.0 — Adopted target baseline QA companion  
**Status:** 180 target cases; implementation execution not yet performed  
**Spec:** [Module 6](./README.md)

## Execution conventions
Use SUPER_ADMIN, Finance, HOD-A, Manager-A, Student-A and Student-B with immutable ProblemVersions, five activity contexts, CodeSubmission parent, SubmissionRun and JudgeRun children. Legacy-only verification is historical; target cases verify adopted behavior in Sections 25–29.

| Test ID | Scenario | Preconditions/data | Steps | Expected result | Trace/status |
|---|---|---|---|---|---|
| SUB-QA-001 | Student submits accessible Practice Problem | Student-A eligible | Submit valid code/language | Submission accepted for processing with unique ID | SUBMIT-F001 |
| SUB-QA-002 | Student submits valid Course Problem | Enrolled Student-A | Submit via Course route | Context/ownership retained; accepted for processing | SUBMIT-F001/SUBMIT-F003 |
| SUB-QA-003 | Student submits valid Lab Problem | Lab membership/eligibility | Submit | Module 8 eligibility passes; result linked to Lab context | SUBMIT-F003 |
| SUB-QA-004 | Student submits valid Assessment Problem | Assessment owner approves | Submit | Module 9 eligibility and attempt policy checked | SUBMIT-F003/SUBMIT-F005 |
| SUB-QA-005 | Student submits valid Contest Problem | Registered/eligible participant | Submit | Contest policy checks pass | SUBMIT-F003 |
| SUB-QA-006 | Unauthenticated Submit | No session | POST submit | Reject; no DB submission or Judge0 job | SUBMIT-F002 |
| SUB-QA-007 | Disabled Student submits | User inactive | Submit | Reject per Module 1; no job | SUBMIT-F002 |
| SUB-QA-008 | Finance Team attempts submit | Finance role | Submit | Deny by RBAC | SUBMIT-F002 |
| SUB-QA-009 | HOD submits as Student | HOD actor | Submit | Deny unless submit-on-behalf is explicitly granted | SUBMIT-F002; SUB-D17 |
| SUB-QA-010 | Student sends foreign studentId | Student-A request includes Student-B ID | Submit | Ignore/reject forged owner; authenticated Student-A is authority | SUBMIT-F002 |
| SUB-QA-011 | Student sends foreign tenantId | Student-A request includes College B | Submit | Server derives tenant/context; no cross-tenant job | SUBMIT-F002 |
| SUB-QA-012 | Student submits Problem-B outside scope | Student-A not eligible | Submit | Deny; no row/job | SUBMIT-F003 |
| SUB-QA-013 | Problem ID invalid | Nonexistent Problem | Submit | Safe validation failure | SUBMIT-F003 |
| SUB-QA-014 | Activity ID does not reference Problem | Valid activity, wrong Problem | Submit | Reject association | SUBMIT-F003 |
| SUB-QA-015 | Unsupported activity type | Unknown context type | Submit | Reject; no inferred support | SUBMIT-F003/SUB-D07 |
| SUB-QA-016 | Activity deadline passed | Owner module says closed | Submit | Deny per owner module | SUBMIT-F003 |
| SUB-QA-017 | Membership removed after page opened | Student stale UI | Submit | Backend rechecks current eligibility | SUBMIT-F003 |
| SUB-QA-018 | Same Problem belongs to two activities | two valid contexts | Submit each | Each request uses explicit validated context | SUBMIT-F003/SUB-D07 |
| SUB-QA-019 | Empty code | code empty | Submit | Reject before persistence | SUBMIT-F004 |
| SUB-QA-020 | Whitespace-only code | code spaces only | Submit | Behavior matches approved blank-code rule | SUBMIT-F004/SUB-D08 |
| SUB-QA-021 | Code payload below limit | Valid code | Submit | Accept if other validation passes | SUBMIT-F004 |
| SUB-QA-022 | Code payload above approved limit | Large source | Submit | Reject before queue; clear validation intent | SUBMIT-F004/SUB-D08 |
| SUB-QA-023 | Code payload near limit | Limit boundary | Submit | Boundary behavior matches contract | SUBMIT-F004/SUB-D08 |
| SUB-QA-024 | Invalid language string | Unknown language | Submit | Reject; no Judge0 call | SUBMIT-F004/SUBMIT-F009 |
| SUB-QA-025 | Language alias normalization | cpp/c++ alias | Submit | Canonical mapping matches approved table | SUBMIT-F004/SUBMIT-F009 |
| SUB-QA-026 | Language not allowed by Problem | Valid platform language, not allowed | Submit | Reject | SUBMIT-F004/Module 5 |
| SUB-QA-027 | Judge0 mapping unavailable | Valid Problem language, no mapping | Submit | Operational validation failure; no wrong-answer result | SUBMIT-F009 |
| SUB-QA-028 | Language version changed | Config updated | Submit old/new version | Version policy consistent | SUBMIT-F009/SUB-D04 |
| SUB-QA-029 | Invalid code encoding | Malformed payload | Submit | Reject safely | SUBMIT-F004/SUB-D08 |
| SUB-QA-030 | Client sends privileged CPU limit | Alter request | Submit | Backend ignores/rejects client-controlled limit | SUBMIT-F004/SUBMIT-F010 |
| SUB-QA-031 | Attempt limit reached | Assessment maxAttempts reached | Submit | Reject and do not create attempt/job | SUBMIT-F005 |
| SUB-QA-032 | Unlimited attempts configuration | maxAttempts null if supported | Submit repeatedly | Behavior matches owner module policy | SUBMIT-F005 |
| SUB-QA-033 | Two concurrent submissions at attempt boundary | one slot left | Submit concurrently | At most approved number accepted | SUBMIT-F005/SUB-D09/SUB-D28 |
| SUB-QA-034 | Queue failure after attempt reservation | queue unavailable | Submit | Consistent rollback/recovery; no ghost attempt | SUBMIT-F005/SUBMIT-F008 |
| SUB-QA-035 | Rerun counted as Student attempt | existing submission | Rejudge | Must not increment attempt unless owner module explicitly says so | SUBMIT-F005/SUBMIT-F019 |
| SUB-QA-036 | Persist valid submission | valid request | Submit | Record has correct owner/problem/context/code/language/time | SUBMIT-F006 |
| SUB-QA-037 | Database insert fails | DB unavailable | Submit | No Judge0 request; error surfaced | SUBMIT-F006 |
| SUB-QA-038 | Initial status/verdict | valid request | Inspect created record | Matches approved state design; do not assume enum | SUBMIT-F006/SUBMIT-F013 |
| SUB-QA-039 | Queue dispatch succeeds | submission row exists | Submit | Job references correct submission | SUBMIT-F008 |
| SUB-QA-040 | Queue unavailable | valid request | Submit | No false “queued” success unless durable recovery | SUBMIT-F008 |
| SUB-QA-041 | Duplicate job delivery | worker receives same job twice | Process | Idempotent result or distinct run IDs; no corrupt final state | SUBMIT-F008/SUBMIT-F014 |
| SUB-QA-042 | Job payload contains foreign actor/context | tampered internal payload | Worker handles | Verify trusted correlation; reject/alert | SUBMIT-F008 |
| SUB-QA-043 | Source code in queue storage | code-bearing job | inspect access controls | Sensitive data protected per security policy | SUBMIT-F008/SUB-D23 |
| SUB-QA-044 | Judge0 request uses configured base URL | env configuration | worker submits | Correct approved environment | SUBMIT-F010/SUBMIT-F011 |
| SUB-QA-045 | Judge0 authentication token | secret configured | submit batch | Token server-side only; no browser exposure | SUBMIT-F011 |
| SUB-QA-046 | Judge0 token missing config | empty/missing secret | submit | Operational error; no secret leak | SUBMIT-F011 |
| SUB-QA-047 | Judge0 accepts batch tokens | multiple test cases | submit | Correct token-to-testcase mapping retained | SUBMIT-F011/SUBMIT-F012 |
| SUB-QA-048 | Judge0 response malformed | malformed JSON/token response | worker handles | Correlation failure, not student WA | SUBMIT-F011/SUBMIT-F014 |
| SUB-QA-049 | Judge0 batch token count mismatch | expected N, got fewer | worker handles | Mark incomplete/operational; no false grade | SUBMIT-F011/SUBMIT-F014 |
| SUB-QA-050 | Judge0 returns duplicate token | malformed/duplicate response | worker handles | Detect correlation error | SUBMIT-F012 |
| SUB-QA-051 | Single token vs per-testcase tokens | conceptual model vs audit | compare design | Schema/API adopted target policy before finalization | SUBMIT-F012/SUB-D06/SUB-D21 |
| SUB-QA-052 | Token persistence | token returned | persist | Token tied to submission/run/testcase as approved | SUBMIT-F012 |
| SUB-QA-053 | Token missing on retry | retry context | worker executes | Recovery/deduplication policy applied | SUBMIT-F012/SUBMIT-F014 |
| SUB-QA-054 | Submission remains pending | queue backlog | read status | Pending displayed distinctly; stale threshold TBD | SUBMIT-F013 |
| SUB-QA-055 | Worker begins execution | job active | status update | State/timestamp follows Module 17 | SUBMIT-F013 |
| SUB-QA-056 | Judge0 processing status | token processing | read result | Not mistaken for final verdict | SUBMIT-F013 |
| SUB-QA-057 | Completed verdict | results saved | retrieve | Completed state and verdict are distinct | SUBMIT-F013 |
| SUB-QA-058 | Worker fails after external Judge0 accept | token exists, DB update fails | recovery | Correlate/recover; avoid untracked duplicate run | SUBMIT-F014 |
| SUB-QA-059 | Judge0 429/5xx | external service failure | submit batch | Retry per approved transient policy | SUBMIT-F014 |
| SUB-QA-060 | Judge0 network timeout | timeout | worker | Infrastructure failure; not Wrong Answer | SUBMIT-F014 |
| SUB-QA-061 | Polling timeout with unfinished tokens | tokens remain processing | poll | Incomplete result identified; no false final grade | SUBMIT-F014 |
| SUB-QA-062 | BullMQ retry creates new token batch | failure retry | worker | Duplicate external executions tracked/prevented per policy | SUBMIT-F014/SUB-D11 |
| SUB-QA-063 | Retry exhaustion | max retries reached | worker | Operational failure state + visibility | SUBMIT-F014 |
| SUB-QA-064 | Worker crash | job active | recovery | Job state/retry behavior deterministic | SUBMIT-F014 |
| SUB-QA-065 | Missing Judge0 token result | one result missing | evaluation | Incomplete evaluation not accepted | SUBMIT-F014/Module 7 |
| SUB-QA-066 | Result retrieval by owner | Student-A owns record | GET result | Safe own result returned | SUBMIT-F015/SUBMIT-F016 |
| SUB-QA-067 | Student-B requests Student-A submission | foreign ID | GET result | Deny/not-found safely | SUBMIT-F016 |
| SUB-QA-068 | HOD views in Department | submission in Dept A | GET | Permitted summary only | SUBMIT-F016 |
| SUB-QA-069 | HOD views foreign Department | submission in Dept B | GET | Deny | SUBMIT-F016 |
| SUB-QA-070 | Manager views assigned Section | scoped submission | GET | Allowed fields within scope | SUBMIT-F016 |
| SUB-QA-071 | Manager views foreign Section | out-of-scope submission | GET | Deny | SUBMIT-F016 |
| SUB-QA-072 | Finance reads submission | Finance role | GET | Deny absent explicit permission | SUBMIT-F016 |
| SUB-QA-073 | SUPER_ADMIN reads global record | authorized admin | GET | Allowed according to approved admin policy | SUBMIT-F016 |
| SUB-QA-074 | Scope changed after submission | actor transferred Section | retrieve old record | Historical access follows approved policy | SUBMIT-F016/SUB-D17 |
| SUB-QA-075 | History list own submissions | Student-A | list | Own records only | SUBMIT-F017 |
| SUB-QA-076 | History list foreign scope filter | Student/HOD query foreign scope | list | Scope filter cannot widen data | SUBMIT-F017 |
| SUB-QA-077 | Pagination stable | many records | next cursor | No unexpected duplicates/skips under defined cursor contract | SUBMIT-F017 |
| SUB-QA-078 | Deactivated Student history | inactive account | retrieve | Retention/access follows policy | SUBMIT-F017/SUB-D19 |
| SUB-QA-079 | WebSocket result update | completed submission | receive event | Client updates authorized result only | SUBMIT-F018 |
| SUB-QA-080 | WebSocket disconnected | no socket | polling fallback | Client eventually gets authoritative persisted state | SUBMIT-F018 |
| SUB-QA-081 | Duplicate grading event | repeated event | client/consumer | No duplicate side effects | SUBMIT-F018/SUBMIT-F024 |
| SUB-QA-082 | Event for another user | socket subscription | publish | Unauthorized client receives no submission data | SUBMIT-F018 |
| SUB-QA-083 | Student requests rerun | Student role | rerun endpoint | Deny | SUBMIT-F019 |
| SUB-QA-084 | HOD rerun permission | HOD actor | rerun | Follow approved DEZOLVER mapping; legacy faculty role not assumed | SUBMIT-F019 |
| SUB-QA-085 | Rerun during active job | current job active | rerun | Reject/serialize per policy | SUBMIT-F019 |
| SUB-QA-086 | Rerun updates history | existing result | rerun | Previous result/run history preserved per approved policy | SUBMIT-F019/SUB-D20 |
| SUB-QA-087 | Cancel pending submission | pending record | cancel | Behavior follows approved cancellation state | SUBMIT-F020 |
| SUB-QA-088 | Cancel running Judge0 job | Judge0 token active | cancel | Do not falsely claim remote execution stopped | SUBMIT-F020 |
| SUB-QA-089 | Cancel completed submission | completed record | cancel | Reject/no-op per state machine | SUBMIT-F020 |
| SUB-QA-090 | Custom Run Code feature status | current target scope | inspect UI/API | Do not expose as available unless approved and implemented | SUBMIT-F021 |
| SUB-QA-091 | Custom stdin with formal submission | Problem/testcases exist | submit stdin | Behavior matches contract; do not bypass official testcases | SUBMIT-F021 |
| SUB-QA-092 | Tenant limit exceeded | limit configured | submit burst | Rate limit response; no partial submission | SUBMIT-F022 |
| SUB-QA-093 | Per-user burst | user submits rapidly | request burst | Approved throttle policy applied | SUBMIT-F022/SUB-D09 |
| SUB-QA-094 | Tenant queue fairness | multiple tenants | enqueue load | Approved fairness behavior | SUBMIT-F022 |
| SUB-QA-095 | Client changes tenant | forged tenant context | submit | Server derives tenant; deny/ignore tampering | SUBMIT-F022 |
| SUB-QA-096 | Student response contains hidden stderr | hidden test failure | inspect response | Hidden-sensitive output masked by Module 7 policy | SUBMIT-F023 |
| SUB-QA-097 | Student response contains compiler diagnostics | compilation error | inspect | Only approved sanitized diagnostics | SUBMIT-F023 |
| SUB-QA-098 | Raw Judge0 response requested by Student | Student role | raw endpoint | Deny | SUBMIT-F023 |
| SUB-QA-099 | Raw token/secret appears in API/log | execution result | inspect outputs | Secrets/internal tokens not disclosed | SUBMIT-F023 |
| SUB-QA-100 | Source code retention expires | retention policy configured | lifecycle job | Retention applied without breaking required audit record | SUBMIT-F025 |
| SUB-QA-101 | Student deleted/deactivated | historical submission | access/retention | Approved retention and privacy behavior | SUBMIT-F025 |
| SUB-QA-102 | Problem deleted with submissions | linked history | delete/archive | Submission history preserved per approved policy | SUBMIT-F025 |
| SUB-QA-103 | Audit submit/rerun/staff access | audit catalog approved | perform action | Required audit record exists | SUBMIT-F024 |
| SUB-QA-104 | Grading event triggers course progress | accepted result | inspect consumer | Event names/contract align; legacy mismatch not repeated | SUBMIT-F024 |
| SUB-QA-105 | Grading event updates contest result | contest context | accepted/partial result | Module 11 contract governs; no implicit contest score assumption | SUBMIT-F024 |
| SUB-QA-106 | Lab score consumes result | Lab context | graded result | Module 8 integration contract applies | SUBMIT-F024 |
| SUB-QA-107 | Notification on grading | notification policy approved | graded event | Trigger/recipient follow Module 13 | SUBMIT-F018 |
| SUB-QA-108 | Quiz/MCQ submit | noncoding assessment | submit | Reject or route only according to approved Module 9/quiz feature | SUBMIT-F003 |
| SUB-QA-109 | Plagiarism detection assumption | submit same code across users | inspect outcome | No plagiarism feature assumed without specification/implementation | SUBMIT-F025 |
| SUB-QA-110 | Formal submission vs scratchpad | both workflows proposed | compare side effects | Separate rules and attempt/scoring behavior | SUBMIT-F021 |
| SUB-QA-111 | Code editor default language | UI loads | inspect default | Target default is approved, not inherited from audit | SUBMIT-F009 |
| SUB-QA-112 | Problem time limit maps to Judge0 CPU time | Problem timeLimitMs configured | execution | Correct units/field conversion; not assessment duration × 60 by assumption | SUBMIT-F010/Module 5 |
| SUB-QA-113 | Memory limit conversion | Problem memoryLimitMb | execution | Unit conversion is correct and documented | SUBMIT-F010/Module 5 |
| SUB-QA-114 | Judge0 endpoint environment | staging vs prod | submit | Environment config does not cross-contaminate | SUBMIT-F011 |
| SUB-QA-115 | Request retry after client timeout | submission persisted, response lost | retry same key | Same submission response replayed or approved conflict | SUBMIT-F007 |
| SUB-QA-116 | Same key with different code | same idempotency key | retry changed payload | Reject/conflict per contract | SUBMIT-F007 |
| SUB-QA-117 | Idempotency cache expires | TTL elapsed | repeat request | Policy on expired key documented; attempt semantics still enforced | SUBMIT-F007 |
| SUB-QA-118 | Queue dispatch durable outbox | DB persisted, queue fails | inspect | Approved recovery prevents orphaned pending submission | SUBMIT-F008 |
| SUB-QA-119 | Poll returns stale result after rerun | rerun pending | client fetch | Client displays current generation/version | SUBMIT-F015/SUBMIT-F019 |
| SUB-QA-120 | Staff raw output access audited | staff granted permission | read raw run | Access is logged where required | SUBMIT-F023/SUBMIT-F024 |

| SUB-QA-121 | Canonical parent/run/outbox | Valid Student and ProblemVersion | POST | One parent, INITIAL run, attempt and outbox commit atomically | SUBMIT-F006; SUB-D05/SUB-D06/SUB-D28 |
| SUB-QA-122 | Transaction rollback | Run/outbox write fails | POST | No parent/attempt/Judge0 job committed | SUBMIT-F006; SUB-D05 |
| SUB-QA-123 | Duplicate outbox delivery | Same event twice | Worker handles | One run dispatch; duplicate ignored | SUBMIT-F008; SUB-D11/SUB-D24 |
| SUB-QA-124 | Single context only | Multiple contexts supplied | Submit | Reject; one contextType/contextId only | SUBMIT-F003; SUB-D07 |
| SUB-QA-125 | Practice context ID | PRACTICE with contextId | Submit | Reject; contextId must be null | SUBMIT-F003; SUB-D07 |
| SUB-QA-126 | Course association | CourseProblem/enrollment valid | Submit COURSE | Module 4 validates link and membership | SUBMIT-F003; SUB-D07/SUB-D27 |
| SUB-QA-127 | Lab association | LabAssignment/group valid | Submit LAB | Module 8/10 validates | SUBMIT-F003; SUB-D07/SUB-D27 |
| SUB-QA-128 | Assessment association | AssessmentProblem/window valid | Submit ASSESSMENT | Module 9 validates membership/window/attempt | SUBMIT-F003/SUBMIT-F005; SUB-D07 |
| SUB-QA-129 | Contest association | ContestProblem/participant valid | Submit CONTEST | Module 11 validates window/participant/attempt | SUBMIT-F003/SUBMIT-F005; SUB-D07 |
| SUB-QA-130 | No Problem activity link | Activity lacks Problem link | Submit | Reject before parent/attempt | SUBMIT-F003; SUB-D07 |
| SUB-QA-131 | ProblemVersion pin | Current Problem V4 | Submit | Parent/run use V4; browser cannot set version | SUBMIT-F003/SUBMIT-F006; SUB-D05 |
| SUB-QA-132 | New version while queued | Submission V4; current V5 | Worker loads tests | Use V4 snapshot | SUBMIT-F010/SUBMIT-F013; SUB-D15 |
| SUB-QA-133 | 100KiB boundary | Code exactly 102400 bytes | Submit | Accept if other checks pass | SUBMIT-F004; SUB-D08 |
| SUB-QA-134 | Over 100KiB | Code 102401 bytes | Submit | Reject pre-attempt | SUBMIT-F004; SUB-D08 |
| SUB-QA-135 | Invalid UTF-8 | Malformed source | Submit | Reject; no corrupted source | SUBMIT-F004; SUB-D08 |
| SUB-QA-136 | Client version override | Foreign version in payload | Submit | Ignore/reject; server resolves | SUBMIT-F002/SUBMIT-F003; SUB-D07 |
| SUB-QA-137 | Client runtime override | Higher CPU/memory sent | Submit | Ignore/reject; pinned limits used | SUBMIT-F004/SUBMIT-F010; SUB-D15 |
| SUB-QA-138 | Parent token deprecated | Read CodeSubmission | Inspect | Token only on JudgeRun | SUBMIT-F006/SUBMIT-F012; SUB-D06 |
| SUB-QA-139 | JudgeRun cardinality | N testcases | Dispatch | N JudgeRuns or unresolved markers | SUBMIT-F010/SUBMIT-F012; SUB-D06 |
| SUB-QA-140 | Duplicate token | Judge0 duplicate token | Persist | Constraint catches; run incomplete | SUBMIT-F012/SUBMIT-F014; SUB-D14 |
| SUB-QA-141 | Batch order differs | GET returns shuffled tokens | Persist | Match by token, not order | SUBMIT-F012; SUB-D06 |
| SUB-QA-142 | Partial batch errors | Some row errors | Process | Valid tokens persist; verdict withheld until resolved | SUBMIT-F012/SUBMIT-F014; SUB-D14 |
| SUB-QA-143 | Ambiguous batch timeout | No token after possibly accepted POST | Recover | Dispatch uncertain; no blind repost | SUBMIT-F014; SUB-D11/SUB-D14 |
| SUB-QA-144 | Retry after token | Tokens stored | Retry | Poll only; no second execution | SUBMIT-F008/SUBMIT-F014; SUB-D11 |
| SUB-QA-145 | Polling deadline | Judge0 slow | Test | 120s configurable; recoverable failure | SUBMIT-F014; SUB-D12 |
| SUB-QA-146 | Status In Queue | Native status 1 | Poll | Nonterminal | SUBMIT-F013; SUB-D13 |
| SUB-QA-147 | Status Processing | Native status 2 | Poll | Nonterminal | SUBMIT-F013; SUB-D13 |
| SUB-QA-148 | Status Internal Error | Native status 13 | Poll | Operational, not WA | SUBMIT-F013/SUBMIT-F014; SUB-D13/SUB-D14 |
| SUB-QA-149 | Unknown status | Unknown native status | Process | Fail closed as mapping failure | SUBMIT-F013; SUB-D13 |
| SUB-QA-150 | No MLE status | Generic runtime only | Process | Do not infer MLE | SUBMIT-F013; SUB-D13 |
| SUB-QA-151 | CPU conversion | timeLimitMs 2000 | Build request | 2.0 seconds | SUBMIT-F010; SUB-D15 |
| SUB-QA-152 | Memory conversion | memoryLimitMb 256 | Build request | 262144 KiB | SUBMIT-F010; SUB-D15 |
| SUB-QA-153 | Aggregate metrics | Case runtime 120/300/210 | Finalize | Max 300ms, not sum | SUBMIT-F013; SUB-D15 |
| SUB-QA-154 | Failed index | First failure orderIndex 2 | Finalize | Display index 3; testcase ID canonical | SUBMIT-F013; SUB-D15 |
| SUB-QA-155 | Compile error index | Compilation fails before tests | Finalize | Index null | SUBMIT-F013; SUB-D15 |
| SUB-QA-156 | Infrastructure verdict separation | Queue fails | Read parent | FAILED processing; verdict null | SUBMIT-F013/SUBMIT-F014; SUB-D14 |
| SUB-QA-157 | Student source access | Student owns record | GET | Own only; foreign denied | SUBMIT-F016; SUB-D17 |
| SUB-QA-158 | Manager source access | Manager assigned Section | GET source | Summary only; raw denied | SUBMIT-F016; SUB-D17 |
| SUB-QA-159 | HOD source access | HOD assigned Department | GET source | Summary only; raw denied | SUBMIT-F016; SUB-D17 |
| SUB-QA-160 | SUPER_ADMIN raw audit | Privileged raw read | Inspect | Actor/target/reason/time audited | SUBMIT-F016/SUBMIT-F024; SUB-D17 |
| SUB-QA-161 | Rejudge preserves old run | Run1 complete | Create Run2 | Run1 immutable; Run2 REJUDGE | SUBMIT-F019; SUB-D20 |
| SUB-QA-162 | Rejudge uses original version | Submission V3; current V5 | Rejudge | Use V3 | SUBMIT-F019; SUB-D20 |
| SUB-QA-163 | Rejudge requires reason | No reason | POST | Reject | SUBMIT-F019; SUB-D20 |
| SUB-QA-164 | Score reconciliation | Contest rejudge; no approval | Complete | No silent rank/score change | SUBMIT-F019/SUBMIT-F024; SUB-D20/SUB-D27 |
| SUB-QA-165 | Student cancel disabled | Student queued | Cancel | Unavailable | SUBMIT-F020; SUB-D18 |
| SUB-QA-166 | Ops cancel pre-handoff | No token | Cancel | CANCELLED + audit + worker guard | SUBMIT-F020/SUBMIT-F024; SUB-D18 |
| SUB-QA-167 | Ops cancel post-handoff | Token exists | Cancel | Do not claim remote execution stopped | SUBMIT-F020; SUB-D18 |
| SUB-QA-168 | Polling fallback | Websocket unavailable | Poll | Starts 3s, backs off max 10s | SUBMIT-F018; SUB-D25 |
| SUB-QA-169 | Foreign websocket | Student B subscribes A | Publish | No foreign data | SUBMIT-F018; SUB-D17 |
| SUB-QA-170 | Duplicate grading event | Same event twice | Consume | Dedupe by eventId/runId | SUBMIT-F024; SUB-D24 |
| SUB-QA-171 | Rate limit | 11 submissions/minute | Submit | At most 10 accepted | SUBMIT-F022; SUB-D22 |
| SUB-QA-172 | Concurrency limit | Two active runs | Submit third | Throttle | SUBMIT-F022; SUB-D22 |
| SUB-QA-173 | Tenant fairness | Tenant saturates queue | Observe | Weighted fair scheduling | SUBMIT-F022; SUB-D22 |
| SUB-QA-174 | Idempotency expiry | Same key after 24h | Submit | Expiry + attempt policy | SUBMIT-F007; SUB-D10 |
| SUB-QA-175 | Idempotency scope | Same key by different Students | Submit | No cross-user leak/conflict | SUBMIT-F007; SUB-D10 |
| SUB-QA-176 | Queue source isolation | Job created | Inspect | IDs only; no code/secrets | SUBMIT-F008/SUBMIT-F023; SUB-D23 |
| SUB-QA-177 | Base64 roundtrip | Unicode/control text | Submit | Bytes preserved | SUBMIT-F010/SUBMIT-F011; SUB-D02 |
| SUB-QA-178 | Judge0 max under Problem limit | Problem exceeds instance cap | Dispatch | Config failure; no widening | SUBMIT-F010/SUBMIT-F014; SUB-D15 |
| SUB-QA-179 | No retention policy | Policy absent | Run purge | No automatic delete | SUBMIT-F025; SUB-D19 |
| SUB-QA-180 | History after deactivation | Student disabled; record exists | Read | Academic record retained per policy | SUBMIT-F025; SUB-D19 |

## QA review notes

SUB-QA-001–180 map to SUBMIT-F001–SUBMIT-F025 and SUB-D01–SUB-D28. Product decisions are adopted; implementation execution remains pending. Do not mark cases passed until executed against the implementation with evidence recorded.