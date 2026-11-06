# Module 15 — Audit Logging QA Test Cases

**Document ID:** AUDIT-FS-QA-001 | **Version:** 0.1 | **Status:** QA draft; decision-dependent tests blocked  
**Related:** [Overview](./README.md) · [Function specifications](./FUNCTION-SPECIFICATIONS.md)

> Expected behavior that depends on open decisions is written as “apply approved decision.” QA must not treat a recommendation or legacy defect as a desired target result. Security-negative cases should be retained even while product behavior is undecided.

## Test fixtures

- SUPER_ADMIN (global).
- FINANCE_TEAM-A (College A), FINANCE_TEAM-B (College B).
- HOD-A (College A / Department A), HOD-B (College A / Department B).
- MANAGER-A (College A / Department A / Section A), MANAGER-B (Section B).
- STUDENT-A and STUDENT-B.
- Audit events in multiple Colleges/departments/sections with varied actor/target scope.
- Events with multiple actions, target types, timestamps including equal timestamps, security events, success/failure outcome, and safe/sensitive metadata.
- Event fixtures with missing actor, missing target, impersonation actor+subject, null IP/user-agent, and large metadata.
- API pagination dataset with enough records to cross several pages.

## QA cases

| Test ID | Function | Scenario / steps | Expected result | Trace |
|---|---|---|---|---|
| AUD-QA-001 | F001 | Approved sensitive admin operation succeeds | Audit record created per coverage catalogue | AUD-D02 |
| AUD-QA-002 | F001 | Non-audit-required read-only operation occurs | Apply approved event coverage; no assumed log | AUD-D02 |
| AUD-QA-003 | F001 | Audit-required operation emits action name outside canonical naming convention | Reject/normalize according to approved action contract | AUD-D02/D17 |
| AUD-QA-004 | F001 | Operation affects one resource | Event target is the correct resource | AUD-D01/D04 |
| AUD-QA-005 | F001 | Operation affects multiple resources | Apply approved one-event vs multiple-event rule | AUD-D17 |
| AUD-QA-006 | F001 | Sensitive action occurs under scoped tenant | Audit event is persisted with correct tenant association | AUD-D05 |
| AUD-QA-007 | F001 | Audit writer receives unapproved metadata field | Field omitted/rejected according to allowlist | AUD-D06 |
| AUD-QA-008 | F001 | Audit persistence fails during business operation | Apply approved fail-open/fail-closed behavior; failure observable | AUD-D07 |
| AUD-QA-009 | F002 | Normal user performs action | Actor is derived from authenticated principal | Security |
| AUD-QA-010 | F002 | Client submits forged actorId in request body | Stored actor remains authenticated principal; client cannot impersonate | Security |
| AUD-QA-011 | F002 | Admin impersonates user and performs action | Actor and subject are represented per approved policy | AUD-D04/D19 |
| AUD-QA-012 | F002 | Route param contains a resource ID unrelated to affected entity | Target is not incorrectly inferred from unrelated param | AUD-D01 |
| AUD-QA-013 | F002 | Target ID is unavailable | Apply approved null/unknown target policy | AUD-D01 |
| AUD-QA-014 | F002 | Request is behind trusted reverse proxy | IP is extracted using approved proxy trust policy | AUD-D15 |
| AUD-QA-015 | F002 | User-agent is missing/oversized/malformed | Normalize/truncate/reject according to policy; request does not crash | AUD-D15 |
| AUD-QA-016 | F003 | SUPER_ADMIN requests audit list | Global permitted entries returned | RBAC |
| AUD-QA-017 | F003 | FINANCE_TEAM-A requests College A records | Only authorized College A entries returned | RBAC |
| AUD-QA-018 | F003 | FINANCE_TEAM-A supplies College B filter | Deny or return no out-of-scope data; cannot widen scope | Scope |
| AUD-QA-019 | F003 | HOD-A queries Department B events | Apply department scope; no unauthorized records | RBAC |
| AUD-QA-020 | F003 | MANAGER-A queries Section B events | Apply manager’s permitted scope | RBAC |
| AUD-QA-021 | F003 | Student attempts audit endpoint | Deny access | RBAC |
| AUD-QA-022 | F003 | Unauthenticated user requests audit list | Authentication failure; no audit data | Auth |
| AUD-QA-023 | F003 | Scoped user changes actor/tenant query parameters | Query remains bounded by server-derived scope | Scope |
| AUD-QA-024 | F003 | Audit query fails at DB layer | Defined error/telemetry; no accidental data leakage | Module 18 |
| AUD-QA-025 | F004 | Filter by approved action returns matching records | Correct authorized records only | AUD-D12 |
| AUD-QA-026 | F004 | Filter by actor ID | Results match actor and remain scope-protected | AUD-D12 |
| AUD-QA-027 | F004 | Filter by target type and ID | Results match target; scope enforced | AUD-D12 |
| AUD-QA-028 | F004 | Filter by time interval boundaries | Boundary inclusivity follows approved specification | AUD-D12 |
| AUD-QA-029 | F004 | Invalid action/category filter | Validation error; no broad fallback query | AUD-D12/D13 |
| AUD-QA-030 | F004 | Multiple filters are combined | Filter combination follows approved AND/OR semantics | AUD-D12 |
| AUD-QA-031 | F004 | Search for security event category | Category mapping follows approved event taxonomy | AUD-D13 |
| AUD-QA-032 | F005 | Static audit dataset paginated across pages | All records traversed exactly once | Pagination |
| AUD-QA-033 | F005 | Multiple events share timestamp | Stable secondary ordering prevents duplicates/skips | AUD-D12 |
| AUD-QA-034 | F005 | Cursor contains invalid format | Defined cursor error; no unscoped fallback | Module 18 |
| AUD-QA-035 | F005 | Cursor copied from a different user/scope | Cannot reveal data outside current authorization | Security |
| AUD-QA-036 | F005 | Cursor used with changed filters | Apply approved cursor/filter consistency behavior | AUD-D12 |
| AUD-QA-037 | F005 | Page size exceeds maximum | Enforce approved maximum | AUD-D12 |
| AUD-QA-038 | F005 | Concurrent new audit entries are inserted while paging | Behavior matches approved consistency strategy | Module 18 |
| AUD-QA-039 | F006 | Authorized user opens entry detail | Correct permitted event fields are returned | AUD-D16 |
| AUD-QA-040 | F006 | User guesses entry ID from another College | Deny/safe not-found; no data leak | Scope |
| AUD-QA-041 | F006 | Entry metadata contains token-like value | Sensitive value is masked/omitted | AUD-D06 |
| AUD-QA-042 | F006 | Entry target link is clicked | Target module rechecks its own permissions | Security |
| AUD-QA-043 | F006 | Entry has null actor or target | UI renders safely; no undefined/null crash | Robustness |
| AUD-QA-044 | F006 | Metadata is nested/large/unknown shape | Safe rendering and approved size policy | AUD-D06 |
| AUD-QA-045 | F007 | Super Admin dashboard loads recent activity | Widget displays approved recent event set | AUD-D14 |
| AUD-QA-046 | F007 | Scoped role opens dashboard widget | Widget returns only permitted scope | AUD-D05 |
| AUD-QA-047 | F007 | Dashboard has no audit records | Approved empty state displayed | AUD-D14 |
| AUD-QA-048 | F007 | Audit API fails on dashboard | Error/loading state; no crash or stale cross-scope data | AUD-D14 |
| AUD-QA-049 | F007 | Dashboard activity click-through | Navigates to Audit Logs without bypassing access checks | Security |
| AUD-QA-050 | F008 | Login success event exists and viewer filters security events | Event appears under approved taxonomy | AUD-D13 |
| AUD-QA-051 | F008 | Login failure event occurs | Captured or excluded per approved failure coverage | AUD-D03 |
| AUD-QA-052 | F008 | MFA enrollment/failure event occurs | Category and event visibility follow policy | AUD-D13 |
| AUD-QA-053 | F008 | Session revocation event occurs | Correct actor/target/session linkage follows policy | AUD-D13 |
| AUD-QA-054 | F008 | Impersonation start/end events occur | Correct taxonomy and actor/subject attribution | AUD-D19 |
| AUD-QA-055 | F008 | Security tab is filtered by event namespace | No known prefix mismatch; mapping follows canonical taxonomy | AUD-D13 |
| AUD-QA-056 | F008 | Non-security event is queried as security event | Event excluded unless taxonomy explicitly classifies it | AUD-D13 |
| AUD-QA-057 | F009 | Student requests list, detail, dashboard, or export | Deny audit access | RBAC |
| AUD-QA-058 | F009 | Finance College A tries to query College B | No cross-college records disclosed | RBAC |
| AUD-QA-059 | F009 | HOD tries to query another department | No cross-department records disclosed | RBAC |
| AUD-QA-060 | F009 | Manager tries to query another section/group | No cross-section/group records disclosed | RBAC |
| AUD-QA-061 | F009 | Frontend hides audit link but user calls API directly | Backend independently denies if unauthorized | RBAC |
| AUD-QA-062 | F009 | Viewer accesses detail using a list cursor or guessed ID | Authorization and scope rechecked | Security |
| AUD-QA-063 | F009 | Role/scope is revoked while token remains active | Apply Module 1 authorization refresh/revocation policy | Module 1 |
| AUD-QA-064 | F009 | Query has no explicit scope filter | Backend applies scope from principal, not global default | Security |
| AUD-QA-065 | F009 | Scope predicate and user filter conflict | Intersection/narrowing behavior; never widen access | Security |
| AUD-QA-066 | F010 | Audit database is unavailable | Apply fail-open/fail-closed policy and report failure | AUD-D07 |
| AUD-QA-067 | F010 | Audit insert times out after DB may have committed | Retry behavior avoids uncontrolled duplicate events | AUD-D08 |
| AUD-QA-068 | F010 | Audit event insert fails and is retried | Retry count/backoff/terminal failure follows policy | AUD-D07 |
| AUD-QA-069 | F010 | Critical admin operation’s audit write fails | Operation behavior follows critical-action policy | AUD-D07 |
| AUD-QA-070 | F010 | Non-critical operation’s audit write fails | Operation behavior follows fail-open/fail-closed policy | AUD-D07 |
| AUD-QA-071 | F010 | Failure is caught by audit writer | Failure is observable through log/metric/alert policy | AUD-D07 |
| AUD-QA-072 | F010 | Audit writer/outbox/queue is unavailable | Defined durability behavior; no silent loss | AUD-D07 |
| AUD-QA-073 | F010 | Audit event is written in same transaction as business update | Atomicity follows approved transaction policy | AUD-D07 |
| AUD-QA-074 | F011 | Request body contains password | Password is never persisted in metadata | Privacy |
| AUD-QA-075 | F011 | Request body contains access/refresh token | Tokens are omitted/redacted | Privacy |
| AUD-QA-076 | F011 | Request body contains MFA secret/recovery code | Secret is omitted/redacted | Privacy |
| AUD-QA-077 | F011 | Request contains payment gateway secret/payload | Sensitive payment fields excluded | Privacy |
| AUD-QA-078 | F011 | Event metadata contains code/source content | Source code excluded unless specifically approved | Privacy |
| AUD-QA-079 | F011 | Manual writer bypasses HTTP interceptor | Redaction policy applies consistently to manual writer | AUD-D06 |
| AUD-QA-080 | F011 | Metadata exceeds approved size limit | Reject/truncate/summarize according to policy | AUD-D06 |
| AUD-QA-081 | F011 | Viewer is less privileged than event metadata classification | Masked/omitted data returned | AUD-D16 |
| AUD-QA-082 | F012 | Login success event is produced | Event persisted with approved identity attribution | AUD-D13 |
| AUD-QA-083 | F012 | Login failure / invalid credential attempt occurs | Event captured or excluded per explicit policy | AUD-D03 |
| AUD-QA-084 | F012 | Logout or logout-all occurs | Event names and actor are consistent | AUD-D13 |
| AUD-QA-085 | F012 | MFA enrollment or challenge fails | Event captures approved result/context without secret | AUD-D13 |
| AUD-QA-086 | F012 | Admin revokes another user’s session | Actor and target are distinguished | AUD-D04 |
| AUD-QA-087 | F012 | Impersonation begins | Actor admin and subject are distinguishable | AUD-D19 |
| AUD-QA-088 | F012 | Impersonation action triggers interceptor and manual log path | Duplicate behavior matches approved dedupe/ownership policy | AUD-D08 |
| AUD-QA-089 | F012 | Unauthorized operation is rejected (403) | Security failure event handled according to approved failure policy | AUD-D03 |
| AUD-QA-090 | F012 | Authentication fails before principal is established | Anonymous/system actor policy is followed | AUD-D04 |
| AUD-QA-091 | F013 | Authorized user requests export | Export only allowed fields and scoped records | AUD-D11 |
| AUD-QA-092 | F013 | User attempts export without export permission | Deny; no file produced | AUD-D11 |
| AUD-QA-093 | F013 | Export includes metadata with secrets | Sensitive fields masked/omitted | AUD-D06/D11 |
| AUD-QA-094 | F013 | Export dataset exceeds limit | Apply approved limit/async job policy | AUD-D11 |
| AUD-QA-095 | F013 | Retention period expires for an entry | Archive/purge/retain according to approved policy | AUD-D10 |
| AUD-QA-096 | F013 | Legal hold is active on audit records | Held records are not purged | AUD-D10 |
| AUD-QA-097 | F013 | Audit log entry is modified/deleted by normal app user | Integrity policy prevents or records mutation | AUD-D09 |
| AUD-QA-098 | F013 | Admin exports or accesses audit data | Apply audit-of-audit access policy | AUD-D20 |

## Coverage summary

98 cases cover audit creation, actor/target attribution, query/filtering, pagination, dashboard, security events, role/scope isolation, durability, redaction, authentication/impersonation, retention, export, and integrity.

### Decision-dependent QA gates
- AUD-D01–D20: expected results must be updated from decisions after product/security/architecture review.
- Until then, tests whose expected outcome depends on those decisions are **blocked / policy-dependent**, not failed and not approved.
- Security tests for cross-tenant, cross-department, cross-section, and student access denial are mandatory regardless of unresolved presentation/retention choices.
