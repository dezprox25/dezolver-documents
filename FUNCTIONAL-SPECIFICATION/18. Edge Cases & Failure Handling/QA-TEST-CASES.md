# Module 18 — Edge Cases & Failure Handling QA Test Cases

**Document ID:** EDGE-FAILURE-QA-001 | **Version:** 0.1 | **Status:** QA draft; policy-dependent  
**Related:** [Overview](./README.md) · [Function specifications](./FUNCTION-SPECIFICATIONS.md)

> Tests dependent on open policy decisions are blocked pending approval. Security, integrity, bounded retries, and safe error disclosure are mandatory quality gates.

## Fixtures
- HTTP requests for validation, auth, permission, conflict, not-found, DB, and unknown exceptions.
- Authenticated and anonymous principals; tenant/plan quotas.
- Redis healthy/down/restarting/flapping; idempotency key records (processing/completed/expired).
- Outbox pending/dispatched/failed/dead-letter events; queue success/failure/ack-loss/duplicate delivery.
- DB pool saturated, Prisma P2002/P2003/P2024/P2025, deadlock and uncertain commit fixtures.
- Judge0, email, payment provider transient/permanent failures and circuit states.
- Frontend token expiry, concurrent 401s, invalid refresh, and component render crash.
- Operational records with age, unresolved failure, legal hold, and archive state.

## Cases

| Test ID | Function | Scenario | Preconditions / steps | Expected result | Trace |
| ERR-QA-001 | ERR-F001 | Known validation error | DTO invalid → Submit invalid request | Correct status/code/envelope returned | ERR-F001 |
| ERR-QA-002 | ERR-F001 | Unauthorized error mapping | JWT missing/expired → Call protected API | 401 normalized response | ERR-F001 |
| ERR-QA-003 | ERR-F001 | Feature plan gate | Feature not entitled → Call feature API | 402/code contract matches approved mapping | ERR-F001 |
| ERR-QA-004 | ERR-F001 | Permission denial | Role lacks permission → Call route | 403/404 anti-enumeration behavior per policy | ERR-F001 |
| ERR-QA-005 | ERR-F001 | Prisma unique constraint | Duplicate unique record → Create duplicate | Conflict response; no SQL leakage | ERR-F001 |
| ERR-QA-006 | ERR-F001 | Prisma missing record | Update/delete nonexistent record → Mutate | Not-found response | ERR-F001 |
| ERR-QA-007 | ERR-F001 | Prisma FK error | Invalid relation reference → Mutate | Defined 4xx mapping; no generic opaque error if policy says map | ERR-F001 |
| ERR-QA-008 | ERR-F001 | Unknown internal exception | Unexpected server exception → Call endpoint | Safe 500 envelope; stack/secret only in protected logs | ERR-F001 |
| ERR-QA-009 | ERR-F002 | Request ID propagation | Request has trace ID → Call API and worker | Correlation retained per propagation contract | ERR-F002 |
| ERR-QA-010 | ERR-F002 | No trace ID supplied | Request lacks ID → Call API | Server generates trace/request ID | ERR-F002 |
| ERR-QA-011 | ERR-F002 | Healthy dependency probes | DB/Redis available → GET health/readiness | Statuses accurately indicate health | ERR-F002 |
| ERR-QA-012 | ERR-F002 | Redis unavailable | Redis down → Health/readiness query | Public info safe; readiness/degraded status matches policy | ERR-F002 |
| ERR-QA-013 | ERR-F002 | DB unavailable | DB down → Health/readiness query | Readiness fails appropriately; no sensitive topology exposed | ERR-F002 |
| ERR-QA-014 | ERR-F002 | Time endpoint clock | GET /time → Compare timestamp | UTC/ISO time and precision match contract | ERR-F002 |
| ERR-QA-015 | ERR-F003 | First mutation with key | Unique key, authorized user → Submit mutation | Mutation executes once; key record created | ERR-F003 |
| ERR-QA-016 | ERR-F003 | Replay completed mutation | Same actor/key/payload → Repeat request | Cached response replay; no second side effect | ERR-F003 |
| ERR-QA-017 | ERR-F003 | Concurrent in-flight request | Same key while first active → Submit concurrently | Conflict/wait behavior per policy | ERR-F003 |
| ERR-QA-018 | ERR-F003 | Same key different payload | Existing key used with different request → Submit different body | Rejected as key mismatch or approved behavior | ERR-F003 |
| ERR-QA-019 | ERR-F003 | Two users same key | Distinct principals → Send identical key | No cross-user response replay | ERR-F003 |
| ERR-QA-020 | ERR-F003 | Anonymous same key collision | Unauthenticated clients → Use same key from two clients | Namespace isolation or route restriction; no response leak | ERR-F003 |
| ERR-QA-021 | ERR-F003 | Handler throws | Mutation fails → Request with idempotency key | Key release/retention follows retry policy | ERR-F003 |
| ERR-QA-022 | ERR-F003 | Long-running handler exceeds processing TTL | Mutation runs longer than TTL → Concurrent retry | No duplicate execution due to lock expiry | ERR-F003 |
| ERR-QA-023 | ERR-F003 | Redis unavailable for idempotency | Redis outage → Submit mutation | Fail-open/closed policy is explicit; critical operation protected | ERR-F003 |
| ERR-QA-024 | ERR-F004 | Within tenant quota | Requests under limit → Submit requests | Requests allowed | ERR-F004 |
| ERR-QA-025 | ERR-F004 | Exceed quota | Requests reach limit+1 → Submit request | 429 response and approved retry metadata | ERR-F004 |
| ERR-QA-026 | ERR-F004 | Boundary burst | Burst at adjacent windows → Submit across boundary | Algorithm prevents unintended quota burst | ERR-F004 |
| ERR-QA-027 | ERR-F004 | Concurrent rate-limit requests | Many parallel requests → Hit quota simultaneously | Atomic count; no over-admission beyond policy | ERR-F004 |
| ERR-QA-028 | ERR-F004 | Plan-specific quota | Tenant has plan tier → Send requests | Correct configured quota applied | ERR-F004 |
| ERR-QA-029 | ERR-F004 | Plan upgrade/downgrade | Plan changes during rate window → Request | Quota transition behavior matches policy | ERR-F004 |
| ERR-QA-030 | ERR-F004 | Rate-limit Redis outage | Redis unavailable → Request | Route-specific fail-open/closed policy applied | ERR-F004 |
| ERR-QA-031 | ERR-F004 | Retry-After metadata | Request limited → Inspect response | Retry timing/headers match API contract | ERR-F004 |
| ERR-QA-032 | ERR-F004 | Exempt health/public route | Route exemption configured → Request repeatedly | Exemption follows documented rule; not overly broad | ERR-F004 |
| ERR-QA-033 | ERR-F005 | Tenant reaches concurrency cap | Tenant A has max active jobs → Enqueue more jobs | Excess tenant jobs queued/deferred | ERR-F005 |
| ERR-QA-034 | ERR-F005 | Two tenants share workers | A batch-heavy, B interactive jobs → Process queue | Tenant fairness prevents starvation | ERR-F005 |
| ERR-QA-035 | ERR-F005 | Worker crashes holding slot | Job worker crashes → Observe slot recovery | Stale slot released/recovered | ERR-F005 |
| ERR-QA-036 | ERR-F005 | Missing tenant ID in job | Job has no tenant context → Process job | Reject/quarantine or global policy applied | ERR-F005 |
| ERR-QA-037 | ERR-F005 | Tenant priority fairness | Jobs with priorities configured → Run queue | Priority/fairness policy respected | ERR-F005 |
| ERR-QA-038 | ERR-F005 | Poison job repeatedly fails | Job always throws → Worker runs | Retries bounded and job reaches recovery/DLQ | ERR-F005 |
| ERR-QA-039 | ERR-F006 | Outbox event dispatch success | Pending event → Poll and publish | Event marked dispatched only after accepted delivery | ERR-F006 |
| ERR-QA-040 | ERR-F006 | Queue unavailable | Pending outbox, queue down → Poll | Failure/retry status recorded | ERR-F006 |
| ERR-QA-041 | ERR-F006 | Publish ack lost | Broker accepted, poller times out → Retry dispatch | Duplicate delivery safe/idempotent | ERR-F006 |
| ERR-QA-042 | ERR-F006 | Consumer handler failure | Event delivery throws → Process event | Retry count/error recorded | ERR-F006 |
| ERR-QA-043 | ERR-F006 | DLQ threshold reached | Event repeatedly fails → Run DLQ processor | Failure record moved to DLQ and source no longer loops | ERR-F006 |
| ERR-QA-044 | ERR-F006 | DLQ cron overlap | Multiple DLQ workers → Run concurrently | No duplicate archival or event loss | ERR-F006 |
| ERR-QA-045 | ERR-F006 | Replay dead-letter | Operator authorized → Replay item | Replay validated and side effects idempotent | ERR-F006 |
| ERR-QA-046 | ERR-F006 | Unauthorized DLQ replay | Unprivileged user → Attempt replay | Denied and audited | ERR-F006 |
| ERR-QA-047 | ERR-F006 | Poison payload | Malformed event → Dispatch/consume | Dead-letter safely; sensitive payload handled per policy | ERR-F006 |
| ERR-QA-048 | ERR-F006 | Outbox retention | Old dispatched event → Purge job | Retention and hold conditions respected | ERR-F006 |
| ERR-QA-049 | ERR-F006 | DLQ recovery failure | Archive insert fails → Run transfer | Original event remains recoverable; alert/error recorded | ERR-F006 |
| ERR-QA-050 | ERR-F007 | Redis transient disconnect | Redis connection drops briefly → Observe service | Reconnection retry policy restores connection | ERR-F007 |
| ERR-QA-051 | ERR-F007 | Redis restart | Redis restarts → API continues/reconnects | System recovers without manual container restart if target policy requires | ERR-F007 |
| ERR-QA-052 | ERR-F007 | Rate limiter during Redis outage | Redis down → Send API requests | Fail-open/closed policy for endpoint applied | ERR-F007 |
| ERR-QA-053 | ERR-F007 | Idempotency during Redis outage | Redis down → Retry mutation | No unsafe duplicate mutation | ERR-F007 |
| ERR-QA-054 | ERR-F007 | Redis degraded mode telemetry | Redis unavailable → Inspect health/logs | Alert and degraded status observable | ERR-F007 |
| ERR-QA-055 | ERR-F007 | Redis recovery cache state | Redis reconnects → Resume requests | Stale/partial cache invalidation or reconciliation occurs | ERR-F007 |
| ERR-QA-056 | ERR-F007 | Redis connection flapping | Repeated drops → Monitor retry behavior | Backoff avoids reconnect storm and alerts | ERR-F007 |
| ERR-QA-057 | ERR-F007 | Redis failure for distributed lock | Lock-dependent action → Redis down | Safety-critical action is denied/serialized per policy | ERR-F007 |
| ERR-QA-058 | ERR-F008 | DB pool exhausted | Connections saturated → Request query | Bounded timeout and safe mapped response | ERR-F008 |
| ERR-QA-059 | ERR-F008 | Prisma P2024 timeout | Pool wait exceeds timeout → Run query | Database timeout mapped and observable | ERR-F008 |
| ERR-QA-060 | ERR-F008 | Prisma P2003 FK violation | Invalid foreign key → Write record | Defined error response; no generic hidden issue | ERR-F008 |
| ERR-QA-061 | ERR-F008 | DB unavailable | DB connection fails → Request | No unsafe retry; service failure is observable | ERR-F008 |
| ERR-QA-062 | ERR-F008 | Deadlock/transaction conflict | Concurrent writes → Execute transaction | Retry only if safe; no partial state | ERR-F008 |
| ERR-QA-063 | ERR-F008 | Pool limit total across clients | Multiple Prisma clients → Load system | Connection budget stays within deployment limit | ERR-F008 |
| ERR-QA-064 | ERR-F008 | DB timeout on mutation | Mutation timeout/uncertain commit → Retry request | Idempotency/commit ambiguity handled safely | ERR-F008 |
| ERR-QA-065 | ERR-F008 | DB error payload privacy | Constraint/query failure → Inspect client response | SQL/schema/internal details not leaked | ERR-F008 |
| ERR-QA-066 | ERR-F009 | Judge0 transient timeout | Judge0 slow → Submit execution | Timeout bounded; retry/terminal state follows Module 6 | ERR-F009 |
| ERR-QA-067 | ERR-F009 | Judge0 unavailable | Network error → Worker executes | Retry policy bounded; submission marked appropriately | ERR-F009 |
| ERR-QA-068 | ERR-F009 | Judge0 malformed response | Invalid result JSON/status → Process response | Validation fails safely; no invalid verdict stored | ERR-F009 |
| ERR-QA-069 | ERR-F009 | Email/SES unavailable | Mail provider down → Send notification | Retry/fallback policy; notification not falsely delivered | ERR-F009 |
| ERR-QA-070 | ERR-F009 | Payment provider timeout | Payment request times out → Retry operation | Provider idempotency prevents duplicate charge | ERR-F009 |
| ERR-QA-071 | ERR-F009 | Circuit breaker opens | Failure threshold reached → Call external dependency | Fail-fast/degraded response per policy | ERR-F009 |
| ERR-QA-072 | ERR-F009 | Circuit breaker half-open | Reset interval elapsed → Probe dependency | Controlled probe; state closes only after success | ERR-F009 |
| ERR-QA-073 | ERR-F009 | Permanent provider error | Invalid request/credential → Call provider | No endless retry; actionable safe error | ERR-F009 |
| ERR-QA-074 | ERR-F009 | External service recovery | Dependency restored → Process queued work | Recovery drains safely without overload | ERR-F009 |
| ERR-QA-075 | ERR-F010 | Single 401 token refresh | Access token expired → Call API | Refresh once and replay eligible request | ERR-F010 |
| ERR-QA-076 | ERR-F010 | Concurrent 401 responses | Several API requests fail auth → Run refresh queue | One refresh flow; requests replay safely | ERR-F010 |
| ERR-QA-077 | ERR-F010 | Refresh token revoked | Refresh fails → API request | Session cleared and user redirected/re-auth prompted | ERR-F010 |
| ERR-QA-078 | ERR-F010 | 401 refresh loop | Refresh endpoint also returns 401 → Request flow | No infinite refresh/replay loop | ERR-F010 |
| ERR-QA-079 | ERR-F010 | Mutation retry | POST fails transiently → Retry through frontend | Same idempotency key preserved | ERR-F010 |
| ERR-QA-080 | ERR-F010 | 403/validation error | Permission/invalid input → API request | Not retried as auth refresh | ERR-F010 |
| ERR-QA-081 | ERR-F010 | ErrorBoundary render crash | Component throws → Render page | Boundary isolates component and recovery action available | ERR-F010 |
| ERR-QA-082 | ERR-F010 | Error envelope mismatch | Backend error shape differs → Frontend parses error | Safe fallback; no crash or lost request correlation | ERR-F010 |
| ERR-QA-083 | ERR-F011 | Auth brute-force edge | Repeated invalid login → Attempt beyond lockout threshold | Lockout/error policy applied and observable | ERR-F011 |
| ERR-QA-084 | ERR-F011 | Revoked refresh session | Refresh request → Use revoked token | 401; session cannot be revived | ERR-F011 |
| ERR-QA-085 | ERR-F011 | Duplicate tenant subdomain | Create tenant duplicate → Submit | 409 conflict; no duplicate tenant | ERR-F011 |
| ERR-QA-086 | ERR-F011 | Subscription past due | Billing dunning reaches threshold → Apply dunning | Tenant entitlement/status follows Module 3 policy | ERR-F011 |
| ERR-QA-087 | ERR-F011 | Student room not in path | Student attempts room action → Request room state mutation | 403/no progress side effect | ERR-F011 |
| ERR-QA-088 | ERR-F011 | Empty/invalid problem test case | Author submits invalid testcase → Save problem | Validation rejects invalid content | ERR-F011 |
| ERR-QA-089 | ERR-F011 | Assessment deadline boundary | Submission arrives at end time → Submit | Deadline timezone/inclusivity rule applied | ERR-F011 |
| ERR-QA-090 | ERR-F011 | Certificate render missing recipient | Credential payload incomplete → Render certificate | Worker handles safely; no unhandled crash or false issue | ERR-F011 |
| ERR-QA-091 | ERR-F012 | Purge old dispatched outbox event | Record beyond retention → Run cleanup | Only approved eligible records purged | ERR-F012 |
| ERR-QA-092 | ERR-F012 | Purge unresolved failed event | DLQ item unresolved → Run cleanup | Record retained/recoverable | ERR-F012 |
| ERR-QA-093 | ERR-F012 | Legal hold on operational record | Record under hold → Run cleanup | No purge | ERR-F012 |
| ERR-QA-094 | ERR-F012 | Archive batch partial failure | Some rows fail archive → Run batch | Successful/failed counts and retries tracked | ERR-F012 |
| ERR-QA-095 | ERR-F012 | Cleanup cron overlap | Two cleanup instances → Run simultaneously | No duplicate destructive action | ERR-F012 |
| ERR-QA-096 | ERR-F012 | Restore/replay after archive | Archived event selected for recovery → Authorized restore/replay | Recovery access and integrity preserved | ERR-F012 |

## Coverage / gates
- **Total:** 96 sequential cases, ERR-QA-001–096.
- Coverage: API error envelope, trace/health, idempotency, rate limits, tenant fairness, outbox/DLQ, Redis, DB pool, external providers, frontend refresh/retry, cross-module edge cases, archive/purge.
- ERR-D01–D24 must be resolved before target acceptance where they affect expected results.
- Modules 1–17 must confirm domain-specific edge cases and compensation semantics.
