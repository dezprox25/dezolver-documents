# Module 15 — Audit Logging: Function-Level Specifications

**Document ID:** AUDIT-FS-FUNCTIONS-001 | **Version:** 0.1 | **Status:** Detailed draft; decision-dependent behavior open  
**Standard:** [Functional Specification Standard](../FUNCTIONAL-SPECIFICATION-STANDARD.md)  
**Parent:** [Module overview](./README.md) · [QA test cases](./QA-TEST-CASES.md)

> **Evidence rule:** “Legacy-verified (prior forensic record)” references the earlier module audit, not a fresh read of the source repository. “Candidate flow” and “TBD” are not approved target behavior. These documents must not be used to implement open policy as if it were decided.

## Shared authorization, data-scope, and audit invariants

| Role | Source permission | Scope |
|---|---|---|
| SUPER_ADMIN | Full View | Global |
| FINANCE_TEAM | View* | Assigned College |
| HOD | View* | Assigned College + Department |
| MANAGER | View* | Assigned College + Department + permitted Section/Group |
| STUDENT | None | No audit log view access |

- Source-confirmed RBAC says backend authorization and scope validation are mandatory; UI visibility is not a security boundary.
- **TBD:** How an audit record is scoped when actor scope and target-resource scope differ.
- **TBD:** Whether query access is limited to events the viewer initiated or may include all events within the viewer’s authorized organizational scope.
- Any event metadata containing secrets, tokens, passwords, code, payment payloads, or sensitive personal information must not be assumed safe to expose. The exact allowlist/redaction policy remains AUD-D06.
- Audit writes, audit reads, exports, and failure events may have different policy. Do not collapse them into a single “audit permission” assumption.

---

## AUD-F001 — Record a sensitive action

**Evidence/status:** Source-confirmed audit intent; legacy write instrumentation reported in prior forensic record; target coverage catalogue TBD.  
**Actors:** Authenticated user, service/system actor, controller/service producing a sensitive action.  
**Entry:** Successful or failed operation requiring an audit event.

### Purpose and outcome
Record an event that can answer who performed what action, against which target, when, and with what permitted context.

### Preconditions
- Event is classified as audit-required by a yet-to-be-approved coverage catalogue.
- Actor identity is available or event is explicitly classified as system/anonymous.
- Action name and target semantics conform to the approved event contract.

### Inputs
| Field | Source / validation |
|---|---|
| action | Required conceptual field; canonical namespace/length TBD |
| actorId / actorUserId | Actor identity; nullable behavior/anonymous handling TBD |
| targetId / resourceId | Conceptual or legacy field; canonical schema TBD |
| targetType / resourceType | Conceptual or legacy field; canonical schema TBD |
| timestamp | Conceptual createdAt / legacy occurredAt; server time source TBD |
| metadata | JSON context; allowlist, size and redaction TBD |
| ipAddress | Schema field; trusted proxy extraction TBD |
| userAgent | Schema field; truncation/retention TBD |
| outcome/result | RBAC document suggests status/result is relevant; schema does not include a dedicated result column; placement TBD |

### Candidate main flow
1. A sensitive operation begins.
2. The operation’s audit classification determines whether an event is required.
3. Actor and target are resolved from trusted server context, not blindly from client payload.
4. Metadata is constructed from approved fields only.
5. The event is persisted through the approved durable mechanism.
6. The caller receives the operation’s result; audit failure handling follows AUD-D07.

### Business rules and system behavior
- Do not include full secrets, credentials, source code, private payment payloads, or unfiltered request bodies.
- Actor identity must represent the real initiating principal; impersonation actor/subject distinctions remain AUD-D04/AUD-D19.
- **Legacy-verified (prior forensic record):** `@AuditAction` annotations reportedly cover 89 controller endpoints; the global interceptor reportedly writes null target/resource fields and empty metadata. Treat count and exact instrumentation as pending source revalidation.

### Success / failure
- Success response and event acknowledgement semantics are TBD.
- Audit persistence failure must not silently be interpreted as success until fail-open/fail-closed policy is decided.
- Retry/deduplication semantics are AUD-D08 and Module 18.

### Acceptance criteria
- An approved audit-required operation produces a record with correct actor/action/time.
- Target and context follow the approved schema/event mapping.
- No unapproved sensitive values are persisted.
- Audit persistence failure follows an explicit policy and is observable.

### QA trace
AUD-QA-001–008. **Unresolved:** AUD-D02, D03, D04, D06, D07, D08, D17.

---

## AUD-F002 — Resolve actor, subject, target, and request context

**Evidence/status:** Conceptual schema source-confirmed; legacy implementation defects reported; canonical contract TBD.  
**Actors:** Audit producer/interceptor/service.

### Purpose and outcome
Attach correct identity and resource context to an audit event.

### Inputs
- Authenticated principal/user ID.
- Optional impersonated subject ID.
- Trusted request parameters/resource ID.
- Resource type/action metadata.
- Request IP and user-agent.
- Approved contextual fields.

### Candidate flow
1. Read identity from the server-established principal.
2. If impersonation is active, preserve both acting administrator and impersonated subject if approved.
3. Resolve target from the operation’s authoritative result/context, not only a generic route parameter.
4. Normalize target type and ID to the canonical schema.
5. Extract IP/user-agent only through approved trusted-proxy rules.
6. Apply field allowlist/redaction.
7. Persist the normalized event.

### Rules and gaps
- The conceptual schema uses `actorId`, `targetId`, `targetType`, `createdAt`; legacy schema is reported to use `actorUserId`, `resourceId`, `resourceType`, `occurredAt`.
- Do not assume both field sets coexist in the final model.
- Route parameters are not guaranteed to identify the actual modified resource.
- Actor, subject, and target are separate concepts; exact relationships remain TBD.

### Failure/edge cases
- Missing actor (system event, public endpoint, auth failure): policy TBD.
- Target ID unavailable: nullable/unknown handling TBD.
- Multiple affected resources: one event vs multiple events TBD.
- IP behind proxy, malformed user-agent, and large metadata: normalization/limit TBD.

### Acceptance criteria
- The record does not attribute an event to a client-forged actor.
- Actor/subject/target fields are consistent with approved contract.
- No incorrect target is inferred solely from a coincidental URL param.
- Unknown identity/target cases are represented according to policy.

### QA trace
AUD-QA-009–015. **Unresolved:** AUD-D01, D04, D06, D15, D19.

---

## AUD-F003 — Query and list audit entries

**Evidence/status:** Audit Logs page and AuditLog/User mapping source-confirmed; `GET /audit/entries` reported as legacy-verified.  
**Actors:** SUPER_ADMIN, FINANCE_TEAM, HOD, MANAGER subject to role/scope.

### Purpose
Retrieve authorized audit events for investigation and compliance review.

### Preconditions
- User is authenticated.
- User has Audit Logs View permission.
- Server can determine role and organizational scope.

### Inputs
- Query filters, pagination/cursor, date range, action, actor, target, and category fields as permitted by final design.
- Filter names, optionality, limits, and defaults are TBD.

### Candidate flow
1. User opens Audit Logs.
2. Frontend sends approved query parameters.
3. Backend authenticates and checks audit-view permission.
4. Backend derives scope from principal and approved policy.
5. Query applies scope **in the database query**, before pagination.
6. Results are sorted and paginated under a stable ordering contract.
7. Response is serialized using the approved API envelope.
8. UI renders the entries and pagination state.

### Rules
- Students are denied.
- SUPER_ADMIN global view is source-confirmed.
- FINANCE/HOD/MANAGER scopes are source-confirmed in RBAC, but audit target-vs-actor scope details remain AUD-D05.
- Client-supplied tenant/college IDs must not override server authorization scope.

### Failure handling
- 401 unauthenticated; 403 insufficient permission; invalid query/cursor validation response; safe not-found/empty behavior TBD.
- Query failure must not return partial data as complete without an explicit contract.

### Acceptance
- Role and scope are enforced for every page/cursor.
- Unauthorized audit data is never returned.
- Filter/pagination parameters cannot widen scope.
- Response schema matches frontend DTO and envelope.

### QA trace
AUD-QA-016–024. **Unresolved:** AUD-D05, D12, D16, D20.

---

## AUD-F004 — Filter and search audit entries

**Evidence/status:** Prior forensic reconstruction reports frontend filter tabs/search; source UI behavior needs code revalidation. Target filters TBD.  
**Actors:** Authorized audit viewer.

### Purpose
Narrow audit results to a useful subset without weakening access controls.

### Candidate filters
- Time interval
- Actor
- Action/event namespace
- Target type and target ID
- Security category
- Outcome/result
- Text search (if approved)

### Candidate flow
1. Viewer selects filters.
2. Frontend validates filter shape and submits request.
3. Backend validates values and applies them after/alongside scope predicates.
4. Backend returns matching authorized records.
5. UI updates results, count/empty state, and cursor.

### Rules/gaps
- No filter names/default ranges are approved by the supplied sources.
- Filter combinations, case sensitivity, text search semantics, and maximum date range are TBD.
- Security tab taxonomy is AUD-D13.
- Legacy forensic record reports a Security Events tab using `auth.` while event names reportedly use `user.*`; treat this as a legacy defect pending code verification.

### Acceptance
- Filters only narrow results.
- Scope predicate is always applied.
- Invalid filters produce validation errors.
- Empty results are not misrepresented as access errors.
- Security-category results match the approved event taxonomy.

### QA trace
AUD-QA-025–031. **Unresolved:** AUD-D12, D13.

---

## AUD-F005 — Paginate and order audit results

**Evidence/status:** Prior forensic record reports cursor pagination and an unstable tie-order issue. Target contract TBD.  
**Actors:** Authorized viewer; API client.

### Purpose
Navigate large audit datasets without omissions or duplicates.

### Inputs
- Cursor token (shape TBD).
- Page size/limit (default and maximum TBD).
- Ordering fields and direction (TBD).

### Candidate flow
1. Initial list request is made without a cursor.
2. Backend applies authorization scope and filters.
3. Backend uses deterministic order and returns page + next cursor.
4. Client requests next page with cursor.
5. Backend validates cursor and repeats scope/filter conditions.
6. Client appends results without duplicates.

### Rules/gaps
- Timestamp-only order can be ambiguous when events share a timestamp; stable tie-breaker policy required.
- Cursor must not expose raw sensitive data or be usable across user scopes unless permitted.
- Limit defaults, maximums, and snapshot consistency are TBD.

### Failure handling
Malformed/expired cursor should return a defined validation error; no fallback to an unscoped query.
Concurrent inserts and changes in sort keys require a chosen consistency strategy.

### Acceptance
- Stable order.
- No duplicate/omitted records for static dataset under pagination.
- Cursor reuse cannot bypass filters/scope.
- Cursor validation and invalid-cursor response are defined.

### QA trace
AUD-QA-032–038. **Unresolved:** AUD-D12, D05, Module 18.

---

## AUD-F006 — View audit entry detail and metadata

**Evidence/status:** AuditLog schema has metadata; detailed entry page is not explicitly specified.  
**Actors:** Authorized audit viewer.

### Purpose
Inspect a single event with permitted actor, target, timestamp, and contextual details.

### Candidate flow
1. Viewer selects an event.
2. Backend verifies audit permission and scope for that event.
3. Backend returns approved fields.
4. UI renders event summary and metadata with masking.
5. Target navigation is available only if the viewer is independently authorized for the target resource.

### Rules/gaps
- The existence of metadata does not imply it should be rendered raw.
- Raw metadata may contain secrets or personal information.
- Target-link behavior must not bypass the target module’s access checks.
- Actor email/role and other joined identity fields are not in the conceptual AuditLog field list; whether they are joined/displayed is TBD.

### Acceptance
- Entry detail respects same or stricter scope than list.
- Sensitive metadata is masked/omitted according to policy.
- Clicking target cannot grant access to a resource otherwise forbidden.
- Missing target/actor does not crash the page.

### QA trace
AUD-QA-039–044. **Unresolved:** AUD-D06, D05, D16.

---

## AUD-F007 — Display recent audit activity on dashboard

**Evidence/status:** Page–DB Mapping maps AuditLog to Super Admin Dashboard; exact widget contract TBD.  
**Actors:** SUPER_ADMIN and any additional approved roles.

### Purpose
Show recent platform activity at a glance.

### Candidate inputs
- Dashboard context and authorized scope.
- Recent entries query, with count/limit, sort, and fields TBD.

### Candidate flow
1. Dashboard loads.
2. Frontend requests recent activity.
3. Backend checks permission/scope and returns authorized events.
4. Widget renders a summary.
5. User may navigate to Audit Logs.

### Rules/gaps
- The mapping only establishes AuditLog entity association, not exact recent count, sort, refresh interval, fields, or role access.
- Dashboard should not reveal events outside the viewer’s scope.
- Whether the widget reuses audit list endpoint or a dedicated endpoint is TBD.

### Failure handling
Loading, empty, forbidden, and service-error UI states are TBD; the UI must not crash or expose stale unauthorized results.

### Acceptance
- Widget access and data scope are independently enforced.
- Only approved fields are shown.
- Widget limit/order/refresh behavior is approved before implementation.
- Failure states are distinct from an empty audit stream.

### QA trace
AUD-QA-045–049. **Unresolved:** AUD-D05, D14, D16.

---

## AUD-F008 — View security and session events

**Evidence/status:** Security / Sessions page mapping source-confirmed; audit security taxonomy TBD. Prior forensic record reports auth namespace mismatch.  
**Actors:** Authorized staff roles; student denied audit-view access under RBAC.

### Purpose
Inspect security-related actions such as authentication, MFA, session revocation, and impersonation where included by policy.

### Candidate flow
1. Viewer opens security activity view/tab.
2. UI applies the approved security event category.
3. Backend authorizes viewer and scope.
4. Query returns events mapped to the approved security taxonomy.
5. UI presents actor, event, time, outcome, and safe context.

### Rules/gaps
- RBAC source mentions security-sensitive operations and says failed requests may be recorded where applicable.
- Exact events, prefixes, and mapping from action strings to security category are AUD-D13.
- Login failures, MFA failures, token revocations, impersonation, permission denials, and session expiry inclusion is TBD.
- Legacy prior-audit claim: the frontend filter uses `auth.` but some backend event labels are `user.*`; revalidate before accepting as a defect.

### Acceptance
- Security filter uses approved canonical taxonomy.
- Events are neither silently excluded nor misclassified.
- Security event detail is masked and scope-protected.
- Failure event policy is explicit.

### QA trace
AUD-QA-050–056. **Unresolved:** AUD-D03, D13, D19.

---

## AUD-F009 — Enforce audit access permission and scope

**Evidence/status:** Source-confirmed RBAC. Prior forensic record reports backend permission and tenant filtering.  
**Actors:** Backend authorization layer.

### Purpose
Prevent disclosure of audit data to unauthorized users or across organizational boundaries.

### Preconditions
- Request is authenticated where endpoint requires auth.
- Principal role/scope is available from trusted auth context.

### Candidate flow
1. Authenticate request.
2. Check Audit Logs View permission.
3. Resolve principal’s authorized scope.
4. Apply scope predicates at query time.
5. Return only permitted audit records.
6. Deny access if authorization fails.

### Rules
- Frontend role guards are not sufficient.
- User-supplied college/department/section filters may only further narrow permitted scope.
- Super Admin global access is source-confirmed.
- FINANCE_TEAM/HOD/MANAGER restrictions are source-confirmed but exact audit-event visibility boundary is open.

### Acceptance
- Student/unauthorized roles cannot read audit data.
- Scoped users cannot access out-of-scope events via list, detail, cursor, filters, or dashboard.
- Permission and scope failures do not mutate application state.
- Role changes are reflected according to token/session authorization policy (Module 1).

### QA trace
AUD-QA-057–065. **Unresolved:** AUD-D05, D16.

---

## AUD-F010 — Handle audit write failures and durability

**Evidence/status:** Audit persistence exists in prior forensic record; legacy code reportedly catches insert errors silently. Target policy TBD.  
**Actors:** Audit writer, database/queue/outbox infrastructure.

### Purpose
Define how the application behaves when it cannot persist a required audit event.

### Candidate failure cases
- Database unavailable.
- Connection pool exhausted.
- Transaction rolled back.
- Queue/outbox unavailable.
- Timeout or network interruption.
- Event inserted but acknowledgement lost.

### Candidate flow
1. Audit producer emits a normalized event.
2. Durable mechanism attempts persistence.
3. Success is confirmed or accepted under approved async contract.
4. Failure is recorded in telemetry/retry mechanism.
5. Business request proceeds or fails according to approved fail-open/fail-closed policy.

### Rules/gaps
- Fail-open vs fail-closed must be decided by operation class (e.g., financial/admin vs ordinary activity) or a global policy.
- No silent catch-and-ignore without approved observability.
- Retry/deduplication behavior is AUD-D08 and Module 18.
- Transactional atomicity vs asynchronous durability is AUD-D07.

### Acceptance
- Audit failures are observable.
- Critical events are not silently lost.
- Business operation behavior matches approved failure policy.
- Retries do not create uncontrolled duplicates.

### QA trace
AUD-QA-066–073. **Unresolved:** AUD-D07, D08, D17.

---

## AUD-F011 — Redact and minimize sensitive audit metadata

**Evidence/status:** Schema has metadata/IP/user-agent; prior forensic record reports dormant LogRedact utility. Target privacy policy TBD.  
**Actors:** Audit producer, redaction/normalization layer, authorized viewer.

### Purpose
Preserve useful forensic context while avoiding unnecessary sensitive-data capture.

### Candidate flow
1. Producer supplies event context.
2. Event-specific allowlist is applied.
3. Sensitive values are removed, masked, hashed, or excluded according to policy.
4. Metadata size and field types are validated.
5. Event is persisted.
6. Viewer-side masking is applied when displaying sensitive fields.

### Rules/gaps
- Request bodies must not be logged wholesale.
- Passwords, access/refresh tokens, MFA secrets, payment secrets, and private source code must not be included.
- IP/user-agent collection and retention are AUD-D15/D10.
- Exact redaction rules and metadata schema are AUD-D06.

### Acceptance
- Known secret fields never persist in clear text.
- Redaction works across interceptor and manual service writers.
- Metadata limits prevent oversized payloads.
- Authorized display still masks fields not approved for that viewer.

### QA trace
AUD-QA-074–081. **Unresolved:** AUD-D06, D15, D16.

---

## AUD-F012 — Record authentication, session, and impersonation events

**Evidence/status:** Business/RBAC source says audit security events may be required; prior forensic record reports login-success and impersonation event writes. Exact coverage and attribution TBD.  
**Actors:** Auth service, session service, security controller, admin initiating impersonation.

### Purpose
Provide traceability for security-sensitive identity lifecycle actions.

### Event categories to review (not approved coverage)
- Login success/failure.
- Logout / logout-all.
- MFA enrollment, challenge, failure, recovery.
- Session/token revocation.
- Impersonation start/end.
- Role/permission changes.
- Access denied / authorization failures.
- Account suspension/deletion.

### Candidate flow
1. Identity/security operation occurs.
2. Event category and outcome are determined.
3. Actor, subject, session reference, and target are resolved.
4. Sensitive authentication material is excluded.
5. Event is persisted according to AUD-D07.
6. Security event UI maps it to approved taxonomy.

### Rules/gaps
- Actor vs impersonated user must not be conflated.
- Never store raw token, password, MFA secret, or recovery code.
- Failure event inclusion and authentication-before-principal behavior are TBD.
- Legacy prior-audit report claims login success is manually logged and impersonation may be double-logged; validate and decide expected behavior.

### Acceptance
- Security events have consistent taxonomy and attribution.
- Secrets are excluded.
- Duplicate events are controlled or explicitly accepted.
- Failed authentication/security events follow approved coverage.

### QA trace
AUD-QA-082–090. **Unresolved:** AUD-D03, D04, D08, D13, D19.

---

## AUD-F013 — Retain, export, archive, or purge audit records

**Evidence/status:** Conceptual schema provides records; retention/export/purge behavior not defined.  
**Actors:** Authorized administrator/compliance actor; scheduled system process if approved.

### Purpose
Define lifecycle and portability of audit data.

### Preconditions
A retention/export policy and authorization matrix must be approved before behavior can be implemented.

### Candidate capabilities
- Retention period and archival.
- Legal hold.
- CSV/JSON export.
- Export redaction and limits.
- Audit of audit-log access/export.
- Purge or anonymization after expiry.

### Rules/gaps
- No export or retention behavior is source-confirmed.
- Audit logs should not be deleted/modified without a clear integrity/legal policy; this is a recommendation, not an approved requirement.
- Export itself can expose sensitive activity and may require a separate permission and an audit event.
- Legal hold and tenant deletion interactions are TBD.

### Acceptance
- No export/purge is enabled before policy approval.
- Export includes only authorized records and approved fields.
- Retention and legal hold are enforced consistently.
- Export/purge activity is traceable if approved.

### QA trace
AUD-QA-091–098. **Unresolved:** AUD-D09, D10, D11, D20.

---

## Function coverage and review status

| Function | Status |
|---|---|
| AUD-F001–F013 | Detailed functional drafts prepared |
| Approved target requirements | None inferred from legacy behavior |
| Product/security decisions | Open; see AUD-D01–D20 |
| Implementation readiness | Blocked until applicable decisions are approved |
