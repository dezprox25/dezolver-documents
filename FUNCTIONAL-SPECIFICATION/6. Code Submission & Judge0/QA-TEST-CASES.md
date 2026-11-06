# Module 6 — Code Submission & Judge0 QA Test Cases

**Version:** 0.1 — QA companion to Functional Specification v0.2  
**Status:** Draft; target decisions remain open  
**Spec:** [Module 6](./README.md)

## Execution conventions
Use SUPER_ADMIN, Finance, HOD-A (College A/Department A), Manager-A (College A/Department A/Section A1), Student-A and Student-B. Use Problem-A and Problem-B, valid/invalid activities, and conceptual submission records. Legacy-only behavior tests are explicitly marked audit verification; target acceptance is blocked until decisions are approved.

| Test ID | Scenario | Preconditions/data | Steps | Expected result | Trace/status |
|---|---|---|---|---|---|
| SUB-QA-001 | Student submits accessible Practice Problem | Student-A eligible | Submit valid code/language | Submission accepted for processing with unique ID | F001 |
| SUB-QA-002 | Student submits valid Course Problem | Enrolled Student-A | Submit via Course route | Context/ownership retained; accepted for processing | F001/F003 |
| SUB-QA-003 | Student submits valid Lab Problem | Lab membership/eligibility | Submit | Module 8 eligibility passes; result linked to Lab context | F003 |
| SUB-QA-004 | Student submits valid Assessment Problem | Assessment owner approves | Submit | Module 9 eligibility and attempt policy checked | F003/F005 |
| SUB-QA-005 | Student submits valid Contest Problem | Registered/eligible participant | Submit | Contest policy checks pass | F003 |
| SUB-QA-006 | Unauthenticated Submit | No session | POST submit | Reject; no DB submission or Judge0 job | F002 |
| SUB-QA-007 | Disabled Student submits | User inactive | Submit | Reject per Module 1; no job | F002 |
| SUB-QA-008 | Finance Team attempts submit | Finance role | Submit | Deny by RBAC | F002 |
| SUB-QA-009 | HOD submits as Student | HOD actor | Submit | Deny unless submit-on-behalf is explicitly granted | F002; D17 |
| SUB-QA-010 | Student sends foreign studentId | Student-A request includes Student-B ID | Submit | Ignore/reject forged owner; authenticated Student-A is authority | F002 |
| SUB-QA-011 | Student sends foreign tenantId | Student-A request includes College B | Submit | Server derives tenant/context; no cross-tenant job | F002 |
| SUB-QA-012 | Student submits Problem-B outside scope | Student-A not eligible | Submit | Deny; no row/job | F003 |
| SUB-QA-013 | Problem ID invalid | Nonexistent Problem | Submit | Safe validation failure | F003 |
| SUB-QA-014 | Activity ID does not reference Problem | Valid activity, wrong Problem | Submit | Reject association | F003 |
| SUB-QA-015 | Unsupported activity type | Unknown context type | Submit | Reject; no inferred support | F003/D07 |
| SUB-QA-016 | Activity deadline passed | Owner module says closed | Submit | Deny per owner module | F003 |
| SUB-QA-017 | Membership removed after page opened | Student stale UI | Submit | Backend rechecks current eligibility | F003 |
| SUB-QA-018 | Same Problem belongs to two activities | two valid contexts | Submit each | Each request uses explicit validated context | F003/D07 |
| SUB-QA-019 | Empty code | code empty | Submit | Reject before persistence | F004 |
| SUB-QA-020 | Whitespace-only code | code spaces only | Submit | Behavior matches approved blank-code rule | F004/D08 |
| SUB-QA-021 | Code payload below limit | Valid code | Submit | Accept if other validation passes | F004 |
| SUB-QA-022 | Code payload above approved limit | Large source | Submit | Reject before queue; clear validation intent | F004/D08 |
| SUB-QA-023 | Code payload near limit | Limit boundary | Submit | Boundary behavior matches contract | F004/D08 |
| SUB-QA-024 | Invalid language string | Unknown language | Submit | Reject; no Judge0 call | F004/F009 |
| SUB-QA-025 | Language alias normalization | cpp/c++ alias | Submit | Canonical mapping matches approved table | F004/F009 |
| SUB-QA-026 | Language not allowed by Problem | Valid platform language, not allowed | Submit | Reject | F004/Module 5 |
| SUB-QA-027 | Judge0 mapping unavailable | Valid Problem language, no mapping | Submit | Operational validation failure; no wrong-answer result | F009 |
| SUB-QA-028 | Language version changed | Config updated | Submit old/new version | Version policy consistent | F009/D04 |
| SUB-QA-029 | Invalid code encoding | Malformed payload | Submit | Reject safely | F004/D08 |
| SUB-QA-030 | Client sends privileged CPU limit | Alter request | Submit | Backend ignores/rejects client-controlled limit | F004/F010 |
| SUB-QA-031 | Attempt limit reached | Assessment maxAttempts reached | Submit | Reject and do not create attempt/job | F005 |
| SUB-QA-032 | Unlimited attempts configuration | maxAttempts null if supported | Submit repeatedly | Behavior matches owner module policy | F005 |
| SUB-QA-033 | Two concurrent submissions at attempt boundary | one slot left | Submit concurrently | At most approved number accepted | F005/D09/D28 |
| SUB-QA-034 | Queue failure after attempt reservation | queue unavailable | Submit | Consistent rollback/recovery; no ghost attempt | F005/F008 |
| SUB-QA-035 | Rerun counted as Student attempt | existing submission | Rejudge | Must not increment attempt unless owner module explicitly says so | F005/F019 |
| SUB-QA-036 | Persist valid submission | valid request | Submit | Record has correct owner/problem/context/code/language/time | F006 |
| SUB-QA-037 | Database insert fails | DB unavailable | Submit | No Judge0 request; error surfaced | F006 |
| SUB-QA-038 | Initial status/verdict | valid request | Inspect created record | Matches approved state design; do not assume enum | F006/F013 |
| SUB-QA-039 | Queue dispatch succeeds | submission row exists | Submit | Job references correct submission | F008 |
| SUB-QA-040 | Queue unavailable | valid request | Submit | No false “queued” success unless durable recovery | F008 |
| SUB-QA-041 | Duplicate job delivery | worker receives same job twice | Process | Idempotent result or distinct run IDs; no corrupt final state | F008/F014 |
| SUB-QA-042 | Job payload contains foreign actor/context | tampered internal payload | Worker handles | Verify trusted correlation; reject/alert | F008 |
| SUB-QA-043 | Source code in queue storage | code-bearing job | inspect access controls | Sensitive data protected per security policy | F008/D23 |
| SUB-QA-044 | Judge0 request uses configured base URL | env configuration | worker submits | Correct approved environment | F010/F011 |
| SUB-QA-045 | Judge0 authentication token | secret configured | submit batch | Token server-side only; no browser exposure | F011 |
| SUB-QA-046 | Judge0 token missing config | empty/missing secret | submit | Operational error; no secret leak | F011 |
| SUB-QA-047 | Judge0 accepts batch tokens | multiple test cases | submit | Correct token-to-testcase mapping retained | F011/F012 |
| SUB-QA-048 | Judge0 response malformed | malformed JSON/token response | worker handles | Correlation failure, not student WA | F011/F014 |
| SUB-QA-049 | Judge0 batch token count mismatch | expected N, got fewer | worker handles | Mark incomplete/operational; no false grade | F011/F014 |
| SUB-QA-050 | Judge0 returns duplicate token | malformed/duplicate response | worker handles | Detect correlation error | F012 |
| SUB-QA-051 | Single token vs per-testcase tokens | conceptual model vs audit | compare design | Schema/API decision required before finalization | F012/D06/D21 |
| SUB-QA-052 | Token persistence | token returned | persist | Token tied to submission/run/testcase as approved | F012 |
| SUB-QA-053 | Token missing on retry | retry context | worker executes | Recovery/deduplication policy applied | F012/F014 |
| SUB-QA-054 | Submission remains pending | queue backlog | read status | Pending displayed distinctly; stale threshold TBD | F013 |
| SUB-QA-055 | Worker begins execution | job active | status update | State/timestamp follows Module 17 | F013 |
| SUB-QA-056 | Judge0 processing status | token processing | read result | Not mistaken for final verdict | F013 |
| SUB-QA-057 | Completed verdict | results saved | retrieve | Completed state and verdict are distinct | F013 |
| SUB-QA-058 | Worker fails after external Judge0 accept | token exists, DB update fails | recovery | Correlate/recover; avoid untracked duplicate run | F014 |
| SUB-QA-059 | Judge0 429/5xx | external service failure | submit batch | Retry per approved transient policy | F014 |
| SUB-QA-060 | Judge0 network timeout | timeout | worker | Infrastructure failure; not Wrong Answer | F014 |
| SUB-QA-061 | Polling timeout with unfinished tokens | tokens remain processing | poll | Incomplete result identified; no false final grade | F014 |
| SUB-QA-062 | BullMQ retry creates new token batch | failure retry | worker | Duplicate external executions tracked/prevented per policy | F014/D11 |
| SUB-QA-063 | Retry exhaustion | max retries reached | worker | Operational failure state + visibility | F014 |
| SUB-QA-064 | Worker crash | job active | recovery | Job state/retry behavior deterministic | F014 |
| SUB-QA-065 | Missing Judge0 token result | one result missing | evaluation | Incomplete evaluation not accepted | F014/Module 7 |
| SUB-QA-066 | Result retrieval by owner | Student-A owns record | GET result | Safe own result returned | F015/F016 |
| SUB-QA-067 | Student-B requests Student-A submission | foreign ID | GET result | Deny/not-found safely | F016 |
| SUB-QA-068 | HOD views in Department | submission in Dept A | GET | Permitted summary only | F016 |
| SUB-QA-069 | HOD views foreign Department | submission in Dept B | GET | Deny | F016 |
| SUB-QA-070 | Manager views assigned Section | scoped submission | GET | Allowed fields within scope | F016 |
| SUB-QA-071 | Manager views foreign Section | out-of-scope submission | GET | Deny | F016 |
| SUB-QA-072 | Finance reads submission | Finance role | GET | Deny absent explicit permission | F016 |
| SUB-QA-073 | SUPER_ADMIN reads global record | authorized admin | GET | Allowed according to approved admin policy | F016 |
| SUB-QA-074 | Scope changed after submission | actor transferred Section | retrieve old record | Historical access follows approved policy | F016/D17 |
| SUB-QA-075 | History list own submissions | Student-A | list | Own records only | F017 |
| SUB-QA-076 | History list foreign scope filter | Student/HOD query foreign scope | list | Scope filter cannot widen data | F017 |
| SUB-QA-077 | Pagination stable | many records | next cursor | No unexpected duplicates/skips under defined cursor contract | F017 |
| SUB-QA-078 | Deactivated Student history | inactive account | retrieve | Retention/access follows policy | F017/D19 |
| SUB-QA-079 | WebSocket result update | completed submission | receive event | Client updates authorized result only | F018 |
| SUB-QA-080 | WebSocket disconnected | no socket | polling fallback | Client eventually gets authoritative persisted state | F018 |
| SUB-QA-081 | Duplicate grading event | repeated event | client/consumer | No duplicate side effects | F018/F024 |
| SUB-QA-082 | Event for another user | socket subscription | publish | Unauthorized client receives no submission data | F018 |
| SUB-QA-083 | Student requests rerun | Student role | rerun endpoint | Deny | F019 |
| SUB-QA-084 | HOD rerun permission | HOD actor | rerun | Follow approved DEZOLVER mapping; legacy faculty role not assumed | F019 |
| SUB-QA-085 | Rerun during active job | current job active | rerun | Reject/serialize per policy | F019 |
| SUB-QA-086 | Rerun updates history | existing result | rerun | Previous result/run history preserved per approved policy | F019/D20 |
| SUB-QA-087 | Cancel pending submission | pending record | cancel | Behavior follows approved cancellation state | F020 |
| SUB-QA-088 | Cancel running Judge0 job | Judge0 token active | cancel | Do not falsely claim remote execution stopped | F020 |
| SUB-QA-089 | Cancel completed submission | completed record | cancel | Reject/no-op per state machine | F020 |
| SUB-QA-090 | Custom Run Code feature status | current target scope | inspect UI/API | Do not expose as available unless approved and implemented | F021 |
| SUB-QA-091 | Custom stdin with formal submission | Problem/testcases exist | submit stdin | Behavior matches contract; do not bypass official testcases | F021 |
| SUB-QA-092 | Tenant limit exceeded | limit configured | submit burst | Rate limit response; no partial submission | F022 |
| SUB-QA-093 | Per-user burst | user submits rapidly | request burst | Approved throttle policy applied | F022/D09 |
| SUB-QA-094 | Tenant queue fairness | multiple tenants | enqueue load | Approved fairness behavior | F022 |
| SUB-QA-095 | Client changes tenant | forged tenant context | submit | Server derives tenant; deny/ignore tampering | F022 |
| SUB-QA-096 | Student response contains hidden stderr | hidden test failure | inspect response | Hidden-sensitive output masked by Module 7 policy | F023 |
| SUB-QA-097 | Student response contains compiler diagnostics | compilation error | inspect | Only approved sanitized diagnostics | F023 |
| SUB-QA-098 | Raw Judge0 response requested by Student | Student role | raw endpoint | Deny | F023 |
| SUB-QA-099 | Raw token/secret appears in API/log | execution result | inspect outputs | Secrets/internal tokens not disclosed | F023 |
| SUB-QA-100 | Source code retention expires | retention policy configured | lifecycle job | Retention applied without breaking required audit record | F025 |
| SUB-QA-101 | Student deleted/deactivated | historical submission | access/retention | Approved retention and privacy behavior | F025 |
| SUB-QA-102 | Problem deleted with submissions | linked history | delete/archive | Submission history preserved per approved policy | F025 |
| SUB-QA-103 | Audit submit/rerun/staff access | audit catalog approved | perform action | Required audit record exists | F024 |
| SUB-QA-104 | Grading event triggers course progress | accepted result | inspect consumer | Event names/contract align; legacy mismatch not repeated | F024 |
| SUB-QA-105 | Grading event updates contest result | contest context | accepted/partial result | Module 11 contract governs; no implicit contest score assumption | F024 |
| SUB-QA-106 | Lab score consumes result | Lab context | graded result | Module 8 integration contract applies | F024 |
| SUB-QA-107 | Notification on grading | notification policy approved | graded event | Trigger/recipient follow Module 13 | F018 |
| SUB-QA-108 | Quiz/MCQ submit | noncoding assessment | submit | Reject or route only according to approved Module 9/quiz feature | F003 |
| SUB-QA-109 | Plagiarism detection assumption | submit same code across users | inspect outcome | No plagiarism feature assumed without specification/implementation | F025 |
| SUB-QA-110 | Formal submission vs scratchpad | both workflows proposed | compare side effects | Separate rules and attempt/scoring behavior | F021 |
| SUB-QA-111 | Code editor default language | UI loads | inspect default | Target default is approved, not inherited from audit | F009 |
| SUB-QA-112 | Problem time limit maps to Judge0 CPU time | Problem timeLimitMs configured | execution | Correct units/field conversion; not assessment duration × 60 by assumption | F010/Module 5 |
| SUB-QA-113 | Memory limit conversion | Problem memoryLimitMb | execution | Unit conversion is correct and documented | F010/Module 5 |
| SUB-QA-114 | Judge0 endpoint environment | staging vs prod | submit | Environment config does not cross-contaminate | F011 |
| SUB-QA-115 | Request retry after client timeout | submission persisted, response lost | retry same key | Same submission response replayed or approved conflict | F007 |
| SUB-QA-116 | Same key with different code | same idempotency key | retry changed payload | Reject/conflict per contract | F007 |
| SUB-QA-117 | Idempotency cache expires | TTL elapsed | repeat request | Policy on expired key documented; attempt semantics still enforced | F007 |
| SUB-QA-118 | Queue dispatch durable outbox | DB persisted, queue fails | inspect | Approved recovery prevents orphaned pending submission | F008 |
| SUB-QA-119 | Poll returns stale result after rerun | rerun pending | client fetch | Client displays current generation/version | F015/F019 |
| SUB-QA-120 | Staff raw output access audited | staff granted permission | read raw run | Access is logged where required | F023/F024 |

## QA review notes
- SUB-QA-001–120 map to SUBMIT-F001–F025.
- Tests marked decision-dependent are blocked until their relevant decision is approved.
- Legacy verification does not imply product behavior approval.
