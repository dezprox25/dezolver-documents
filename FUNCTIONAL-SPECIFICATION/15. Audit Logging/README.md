# DEZOLVER — Module 15: Audit Logging

**Document ID:** AUDIT-FS-001  
**Version:** 0.1  
**Status:** Detailed draft — source-aligned; target audit policy and retention decisions remain open  
**Governing standard:** [Functional Specification Standard v1.0](../FUNCTIONAL-SPECIFICATION-STANDARD.md)  
**Repository inclusion:** Draft only; inclusion/commit does not constitute approval of target behavior.

> **Scope note:** This module is being upgraded from the earlier audit reconstruction. It distinguishes conceptual source requirements from the observed legacy implementation. Legacy evidence is referenced from the existing forensic reconstruction in `functional-spec-from-old code/module-15-audit/SPECIFICATION.md`; the application source tree itself is not present in this repository checkout, so line-level source claims should be revalidated against the actual codebase before implementation sign-off.

## 1. Purpose and module outcome

Audit Logging provides accountability and forensic visibility for sensitive platform actions. The supplied business/schema/RBAC material calls for logging important administrative and data-changing operations and provides an AuditLog data shape. Page–DB Mapping identifies Audit Logs in the Super Admin area, AuditLog data in the Super Admin dashboard, and audit records in Profile & Settings mappings.

This specification captures:
- audit event recording and actor/action/resource/context fields;
- role- and scope-governed audit log viewing;
- Audit Logs page, dashboard activity usage, filtering/search/pagination concepts;
- security and session event visibility;
- legacy implementation findings and known cross-layer defects;
- open product/security decisions about event coverage, failure auditing, data minimization, immutability, retention, export, and tenant visibility.

**Important distinction:** The source documents describe audit requirements and data mapping, not a complete approved target design. The legacy implementation is not automatically the target behavior.

## 2. Scope

### In scope
- AuditLog data model and source/legacy schema comparison.
- Capturing actor, action, target/resource, timestamp, request context, and metadata where defined.
- Auditing sensitive administrative/data-changing actions.
- Audit Logs list/detail view and role-scoped query access.
- Dashboard recent audit activity.
- Security/session action logging and audit category filtering.
- Tenant/college, department, section/group, and ownership scope boundaries.
- Failure auditing, redaction, retention, export, and audit integrity as open policy areas and legacy findings.

### Out of scope
- Authentication and session management mechanics themselves (Module 1); only their audit events are in scope.
- Security session revocation mechanics (security/session module); only corresponding audit event visibility/recording is in scope.
- Ticket audit policy implementation (Module 14), except cross-reference.
- Notification delivery mechanics (Module 13), except whether audit events should trigger notifications.
- Analytics aggregation (Module 16).
- Final retention/legal hold policy, export authorization, and target audit architecture until product/security approval.

## 3. Source documents and evidence

| Source | Relevant evidence | Limitations |
|---|---|---|
| [Business Flow original](../../docs/source-text/Business%20Flow.md) | Overall actor/domain flow | Does not describe audit event lifecycle or log views. |
| [Business Flow v2](../../docs/source-text/Business%20Flow%20v2.md) | Backend is intermediary for coding flows; stakeholder relationships | Audit logging not separately detailed. |
| [Page–DB Mapping original](../../docs/source-text/Page%20-%20DB%20Mapping.md) | Super Admin Dashboard maps AuditLog; Audit Logs page maps AuditLog + User; Profile & Settings maps AuditLog | Page/entity mapping only. |
| [Page–DB Mapping v2](../../docs/source-text/Page%20-%20DB%20Mapping%20v2.md) | Same Audit Logs / dashboard / settings references | No list filters, detail layout, or page interactions. |
| [RBAC](../../docs/source-text/RBAC.md) | Audit Logs: SUPER_ADMIN full view, FINANCE_TEAM/HOD/MANAGER view*; student no access; * scope restrictions | Role/resource-level matrix, not audit-specific filter semantics. |
| [Schema](../../docs/source-text/scheema.md) | Conceptual AuditLog fields: id, action, actorId, targetId, targetType, metadata, ipAddress, userAgent, createdAt; User creates audit logs | Diagram schema, not proof of deployed schema. |
| [Functional Specification Standard](../FUNCTIONAL-SPECIFICATION-STANDARD.md) | Evidence labels, detailed function template, decision log, QA and review gates | Authoring standard, not product behavior. |
| [Earlier Module 15 forensic reconstruction](../../functional-spec-from-old%20code/module-15-audit/SPECIFICATION.md) | Legacy audit schema, interceptor, endpoint, UI/service, RBAC and defects | Secondary forensic artifact; source paths/lines are included there and need source checkout validation before implementation. |

### Evidence labels used here
- **Source-confirmed:** explicitly present in supplied business flow, Page–DB Mapping, RBAC, or schema documents.
- **Legacy-verified (prior forensic record):** observed in the prior Module 15 forensic reconstruction, with implementation path references.
- **Derived:** conclusion drawn from multiple supplied sources.
- **Recommendation:** proposal only; not approved.
- **TBD / Decision Required:** missing, conflicting, or requiring product/security/technical approval.

## 4. Terminology

| Term | Meaning / evidence |
|---|---|
| Audit event / AuditLog | A record of an action performed in the platform. The conceptual schema calls it AuditLog. |
| Actor | User or system principal that initiated an event; conceptual schema uses actorId. |
| Target / resource | Entity affected by the action; conceptual schema uses targetId and targetType. |
| Action | Event/action name, such as a sensitive administrative action. Exact canonical naming policy is TBD. |
| Metadata | Contextual JSON associated with an audit event. Allowed keys and sensitive-data policy are TBD. |
| Audit Logs page | Super Admin page mapped to AuditLog and User. |
| Scope | College, department, section/group, or global boundary applied to access. |
| Security event | Authentication/session/authorization or other security-relevant event; exact taxonomy is TBD. |
| Immutable/append-only | Audit record integrity expectation; source documents do not explicitly define the enforcement mechanism. |

## 5. Actors, permissions, and data scope

### 5.1 Source RBAC matrix

| Role | Audit Logs permission | Scope |
|---|---|---|
| SUPER_ADMIN | Full View | Global |
| FINANCE_TEAM | View* | Assigned College |
| HOD | View* | Assigned College + Department |
| MANAGER | View* | Assigned College + Department + permitted Section/Group |
| STUDENT | — | No audit log access |

`*` means scope restriction is required. RBAC also states that frontend restrictions are not sufficient; backend authorization and data-scope validation are mandatory.

### 5.2 Security requirements
- **Source-confirmed:** Audit log viewing is role-gated; student access is not granted by the matrix.
- **Source-confirmed:** Role permission alone does not grant unrestricted access; data scope must be applied.
- **Derived:** A user must not query audit entries from a different College/Department/Section merely by supplying a filter or target ID.
- **TBD:** Whether Finance/HOD/Manager can see all audit events in their scope, only events affecting resources in scope, or only events they personally initiated.
- **TBD:** Whether impersonated users, support agents, auditors, or external compliance roles receive separate audit access.

## 6. Module overview and entry points

| Surface | Source reference | Evidence status |
|---|---|---|
| Super Admin Audit Logs page | Page–DB Mapping original/v2: Super Admin → Audit Logs → AuditLog + User | Source-confirmed page/entity mapping |
| Super Admin dashboard recent audit activity | Page–DB Mapping original/v2 maps AuditLog to dashboard | Source-confirmed entity mapping; widget behavior TBD |
| Profile & Settings audit records | Page–DB Mapping original/v2 maps AuditLog + User | Source-confirmed mapping; view/actions TBD |
| Security / Sessions page | Page–DB Mapping maps RefreshSession + User | Source-confirmed page mapping; audit-event linkage TBD |
| Audit query endpoint | Prior forensic reconstruction: `GET /audit/entries` | Legacy-verified in prior forensic record |
| Cross-module write instrumentation | Prior forensic reconstruction: `@AuditAction`, AuditInterceptor, explicit AuthService inserts | Legacy-verified in prior forensic record |

## 7. Functional inventory

| ID | Function | Evidence / status |
|---|---|---|
| AUD-F001 | Record audit events for sensitive actions | Source-confirmed intent; event catalogue/coverage TBD; legacy interceptor coverage reported |
| AUD-F002 | Record actor/action/target/time/context fields | Source-confirmed schema fields; legacy resource/context capture defect reported |
| AUD-F003 | Query and list audit logs | Page/entity mapping source-confirmed; legacy endpoint reported |
| AUD-F004 | Filter/search audit logs | UI legacy filters reported; target filter vocabulary TBD |
| AUD-F005 | Cursor pagination and ordering | Legacy endpoint reported; stable ordering defect reported |
| AUD-F006 | View audit entry details/context | Derived from audit log purpose; detail view not explicit |
| AUD-F007 | Display recent audit activity on dashboard | Source-confirmed mapping; exact limit/columns TBD |
| AUD-F008 | View security/session events | Security/session page source-confirmed; event taxonomy and filter mapping TBD |
| AUD-F009 | Enforce audit access permissions and data scope | Source-confirmed RBAC rules |
| AUD-F010 | Handle audit write failures | Prior legacy failure mode reported; target durability policy TBD |
| AUD-F011 | Redact/minimize sensitive metadata | Prior legacy dead redaction decorator reported; policy TBD |
| AUD-F012 | Audit authentication and impersonation events | Prior forensic record reports explicit login/impersonation events; desired coverage TBD |
| AUD-F013 | Retain, export, or purge audit records | No source-defined export/retention/purge behavior; capability/policy TBD |

## 8. Data and state reference

### 8.1 Conceptual source schema (Schema document)
| Field | Type | Source description |
|---|---|---|
| id | varchar | Record identifier |
| action | varchar(100) | Action/event name |
| actorId | varchar | Actor user identifier |
| targetId | varchar(255) | Target record identifier |
| targetType | varchar(100) | Target entity type |
| metadata | json | Context |
| ipAddress | varchar(255) | Request IP |
| userAgent | varchar(1000) | Client user-agent |
| createdAt | timestamp | Event timestamp |

### 8.2 Legacy schema recorded in prior forensic reconstruction
The previous reconstruction describes `audit.audit_entries` with `id`, `tenantId`, `actorUserId`, `action`, `resourceType`, `resourceId`, `ipAddress`, `userAgent`, `metadata`, and `occurredAt`.

**Schema divergence:** conceptual `AuditLog.actorId/targetId/targetType/createdAt` differs from legacy `actorUserId/resourceId/resourceType/occurredAt`. This is not resolved by this draft. See AUD-D01.

### 8.3 State/lifecycle
Audit events are modeled as historical records. The source schema contains no status/state field. **Derived:** events should not be treated as mutable workflow records. Whether update/delete are technically prevented is TBD.

## 9. Cross-module consistency

| Module | Relationship | Issue / coordination |
|---|---|---|
| 1 Authentication & User Lifecycle | Login, logout, MFA, session revocation, impersonation | Define event names, actor/subject distinction, failure coverage, and identity fields. |
| 2 College / Department / Section Management | Organizational scope changes | Audit tenant/department/section mutations and scope changes. |
| 3 Subscription & Seat Management | Billing and subscription actions | Sensitive financial event coverage and data minimization. |
| 4 Course & Enrollment | Academic resource mutations | Audit create/update/delete and enrollment changes where policy requires. |
| 5 Problem Management | Problem/test-case administration | Audit administrative changes; avoid recording secret test case contents. |
| 6 Code Submission & Judge0 | Student submissions/execution | Decide whether code submissions are audit events; avoid source-code/secret leakage. |
| 7 Test Case Evaluation | Verdict/grading updates | Define audit vs operational logs and regrade event capture. |
| 9 Assessments | Assessment create/update/review | Audit publication, configuration, access, and anti-cheat review. |
| 10 Groups | Membership/access boundary changes | Audit membership changes where policy requires. |
| 11 Contests | Registration, scoring, ranking/admin actions | Audit sensitive contest administration. |
| 12 Certificates | Issuance/revocation | Audit credential issuance and revocation. |
| 13 Notifications | Audit event notification delivery | Event-to-notification policy is not defined. |
| 14 Support Tickets | Ticket/comment/verification actions | Module 14 decision TICKET-D17 references audit policy; keep linked. |
| 16 Student Analytics | Audit data may feed reports | Do not treat raw audit logs as analytics without approved privacy/aggregation rules. |
| 17 Functional State Machines | Lifecycle changes | Define which transitions must emit audit events. |
| 18 Edge Cases & Failure Handling | retries, duplicates, consistency | Define audit behavior on retries, concurrency, and partial failures. |

## 10. Decision log

| ID | Decision required | Options / considerations | Status |
|---|---|---|---|
| AUD-D01 | Canonical audit field naming and schema | Keep legacy `actorUserId/resourceId/resourceType/occurredAt`; adopt conceptual `actorId/targetId/targetType/createdAt`; or map both with explicit DTO contract | Open |
| AUD-D02 | Audit event coverage catalogue | All sensitive operations; role/action matrix; or event catalogue by module | Open |
| AUD-D03 | Success and failure event coverage | Success only; all relevant 4xx/5xx; selected security failures; avoid noise/PII | Open |
| AUD-D04 | Actor vs subject/target representation | Actor only; actor + subject + target; impersonation actor/subject separation | Open |
| AUD-D05 | Tenant/department/section scope semantics | Scope by target resource, actor, both, or a special global-audit permission | Open |
| AUD-D06 | Metadata allowlist and PII redaction | Explicit per-event allowlist, structured schema, redaction policy, secret exclusion | Open |
| AUD-D07 | Audit durability on DB/queue failure | Synchronous transaction, outbox, durable queue, fail-closed vs fail-open | Open |
| AUD-D08 | Duplicate audit event policy | Idempotency key/event ID; accept duplicate attempts; dedupe rules | Open |
| AUD-D09 | Record immutability and tamper evidence | DB constraints/permissions, append-only storage, hash chain/WORM/other | Open |
| AUD-D10 | Retention, purge, legal hold, and archival | Retention period, archive, deletion/hold authority | Open |
| AUD-D11 | Export capability | No export; CSV/JSON; privileged export; redaction and audit of export itself | Open |
| AUD-D12 | Audit Logs UI filters and search | Time range, actor, action, target type/id, result, security category, text search | Open |
| AUD-D13 | Security event taxonomy | Canonical event prefixes and mapping for auth/session/MFA/impersonation/authorization | Open |
| AUD-D14 | Dashboard recent activity | Count, sort, fields, scope, refresh, empty/error states | Open |
| AUD-D15 | IP/user-agent handling | trusted proxy rules, normalization, retention, privacy limits | Open |
| AUD-D16 | Audit record detail and metadata visibility | Role-based masking; internal sensitive metadata; target link behavior | Open |
| AUD-D17 | Cross-module event contract | Common schema, event names, producer ownership, versioning | Open |
| AUD-D18 | Logging/audit separation | What belongs in audit records vs application logs/metrics/traces | Open |
| AUD-D19 | Audit event actor attribution during impersonation | Acting admin vs impersonated identity and session linkage | Open |
| AUD-D20 | Audit data access auditing | Whether every audit query/export must itself be recorded | Open |

## 11. Document review status

- **Draft completion:** Function inventory, evidence mapping, legacy discrepancies, and decision register prepared.
- **Product approval:** Pending.
- **Security/privacy approval:** Pending.
- **Target behavior approval:** Pending.
- **Implementation readiness:** Not ready until decisions AUD-D01–D20 are reviewed.
- **Repository status:** Draft; commit does not imply approval.

## 12. Readiness checklist

- [x] Business Flow original/v2 reviewed; lack of audit workflow noted.
- [x] Page–DB Mapping original/v2 reviewed.
- [x] RBAC audit role/scope matrix captured.
- [x] Conceptual AuditLog schema captured.
- [x] Earlier Module 15 forensic reconstruction reviewed as legacy evidence.
- [x] Schema mismatch and prior implementation defects explicitly separated.
- [x] Function-level specifications and QA companion created.
- [ ] Canonical schema/DTO approved.
- [ ] Event coverage and failure audit policy approved.
- [ ] Retention/export/privacy/security policy approved.
- [ ] Target requirements approved.

## 13. Review and approval workflow

1. Product reviews AUD-D01–D20 and records decisions.
2. Security/privacy reviews event coverage, metadata, failure capture, actor attribution, and retention.
3. Architecture reviews durability, event schema, and cross-module producer contract.
4. QA updates blocked tests with approved expected results.
5. Only approved decisions may be converted into target requirements.
