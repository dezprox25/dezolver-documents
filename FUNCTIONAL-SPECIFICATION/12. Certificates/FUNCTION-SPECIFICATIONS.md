# Module 12 — Certificates: Function-Level Specifications

**Document ID:** CERT-FS-FUNCTIONS-001 | **Version:** 0.2 | **Status:** Detailed draft; decisions open  
**Standard:** [Functional Specification Standard](../FUNCTIONAL-SPECIFICATION-STANDARD.md)  
**Parent:** [Module overview](./README.md) · [QA companion](./QA-TEST-CASES.md)

> Source facts, derived requirements, and TBD behavior are separated. These are drafts, not product-approved implementation requirements.

## Permissions summary

| Role | View | Create | Update | Delete | Scope |
|---|---|---|---|---|---|
| SUPER_ADMIN | Allow | Allow | Allow | Allow | Global per RBAC |
| FINANCE_TEAM | Deny | Deny | Deny | Deny | No Certificate permission |
| HOD | Allow | Allow | Allow | Deny | Authorized College/Department |
| MANAGER | Allow | Deny | Deny | Deny | Authorized scope |
| STUDENT | Allow own | Deny | Deny | Deny | Own certificates only |

RBAC resource-level Certificate permissions do not explicitly separate template CRUD from issuance/revocation. Fine-grained permission mapping remains CERT-D03–D06.

## Historical function and route crosswalk

**Evidence status: LEGACY-SPEC-REPORTED.** The old-code Module 12 specification reports these endpoints, consumers and defects from a prior Credentials implementation. This pass reviewed the forensic report artifact only; raw application files were not independently fetched. No item below is a target approval.

| Target function ID | Historical route/behavior reported | Target specification treatment |
|---|---|---|
| CERT-F001 | `GET /certificate-templates`; legacy tenant-scoped list. | Target page is HOD Certificate Templates; apply target College/Department scope and roles. Legacy tenant field and route are not target contract. |
| CERT-F002 | `POST /certificate-templates`; legacy template fields name/kind/design and reported initial draft status. | Target fields are title/htmlTemplate/stylesConfig. Legacy status/default/design format is not imported. |
| CERT-F003 | The report's listed route inventory did not identify a template update route. | Target RBAC permits HOD/SUPER_ADMIN update; exact target endpoint, editable fields and creator ownership remain open. |
| CERT-F004 | The report's listed route inventory did not identify template deletion; it did include DELETE issuance-rule route. | Target Super Admin Delete permission is source-confirmed; restrict/archive/delete with issued references remains CERT-D19. |
| CERT-F005 | `POST /certificate-templates/:id/preview` was reported; renderer allegedly generated a preview payload. | Target preview is a derived candidate only. Template sanitization, stylesheet schema, placeholders and sample-data rules remain CERT-D08–D11. |
| CERT-F006 | Manual `/certificates` and `/certificates/manual`; background `PathCompleted`, `EventCompleted`, `RoomCompleted` consumers; report says templates had to be published/active. | Target Course/Contest source fields exist, but Business Flow defines no issuance workflow. Do not import event triggers, template published/active state, rules table, top-three awards or automatic issue. |
| CERT-F007 | `GET /me/certificates`, `GET /me/certificates/:id`; report says server checked owner. | Target Student My Certificates page and RBAC V OWN support owner-scoped access; fields/summary projection and related-course/contest access remain open. |
| CERT-F008 | `GET /me/certificates/:id/download` reportedly redirected to CloudFront `pdfUrl`; report says stored object was HTML, not PDF. | Target `pdfUrl` field alone does not establish binary PDF, direct URL, redirect or public access. CERT-D15 controls artifact/access contract. |
| CERT-F009 | Public `GET /certificates/verify/:qrCode` and `/verify/c/:certificateId` were reported. | Target `verificationCode`/`qrCodeUrl` fields do not prove public verification is required or public. CERT-D16/17/18 remain open. |

### Historical operations without a target function ID

| Historical operation | Reported legacy behavior | Why not promoted to target function |
|---|---|---|
| Template publish | `POST /certificate-templates/:id/publish`; legacy template status draft → published, plus active flag. | Current target CertificateTemplate has no status/active field. CERT-D27 added. |
| Issuance rules | GET/POST/DELETE `/issuance-rules`; report states IssuanceRule model/table was absent and repository methods returned empty/mock results. | Not target schema/page-supported; keep outside inventory pending target need and schema (CERT-D02/20/28). |
| Student privacy toggle | `PATCH /me/certificates/:id` changed `isPublic`. | Target Certificate has no isPublic field; public directory semantics are not source-defined (CERT-D17/28). |
| Reissue/revoke | POST `/certificates/:id/reissue` and `/revoke`; old record preserved, new record got a new QR; revoke stored reason/time and changed status. | Target schema has no status/revocation fields/history. Lifecycle decision belongs to CERT-D13/18/FSM-D12. |
| Public and staff pages | Public verification pages, staff all-certificates console, issuance-rule page and private WebSocket event were reported. | Target Page–DB Mapping includes HOD Certificate Templates and Student My Certificates only. Additional pages/channels remain CERT-D21/25 and Module 13 decisions. |

### Target schema versus legacy schema — do not conflate

| Target conceptual field | Historical counterpart | Evidence note |
|---|---|---|
| CertificateTemplate.title | `name` | Not same source field; target title length 255. |
| CertificateTemplate.htmlTemplate | `design` JSON | Target has HTML text; old report says design JSON. Rendering/sanitization remains open. |
| CertificateTemplate.stylesConfig | `design` JSON | Target separate JSON config; old model had a single design field. |
| Certificate.verificationCode | `qrCode` | No evidence they share format, entropy, uniqueness constraint or public exposure. |
| Certificate.qrCodeUrl | QR code token/URL behavior | Field exists; target payload/destination semantics not defined. |
| Certificate.courseId / contestId | `contextType` / `contextId` (`course`, `event`, `path`, `room`, etc.) | Target explicitly maps Course and Contest; legacy path/room/event identifiers are not target entity relations. |
| Certificate.studentId | legacy `userId` | Target uses Student semantics; identity/ownership must be resolved via Module 1. |
| Certificate.pdfUrl | legacy URL to S3-hosted `.html` | Do not infer PDF MIME or binary content from field name. |
| No target field | legacy `isPublic`, `status`, `revokedAt`, `revocationReason`, `metadata` | These need separate product/schema decisions; do not write them as current target facts. |

## CERT-F001 — List and view Certificate Templates

**Requirement status:** Source-confirmed page/entity; detailed UI derived  
**Specification status:** Decision required  
**Actors:** HOD, SUPER_ADMIN; MANAGER view-only per RBAC  
**Entry:** HOD Certificate Templates page

**Purpose:** Display templates available in the actor’s authorized scope.

**Preconditions:** Authenticated active user; Certificate view permission; College/Department scope resolved through Module 2.

**Inputs**

| Input | Source/type | Required | Validation |
|---|---|---|---|
| page/filter/sort | Query, contract TBD | Optional | Shared API validation |
| collegeId | Scope context | System | Server-authorized only |
| templateId | Route | Detail request | Must exist and be in scope |

**Main flow**
1. Actor opens Certificate Templates.
2. Server resolves role, scope, and permissions.
3. Server queries only authorized templates.
4. UI renders approved summary fields (columns TBD).
5. On detail request, server rechecks scope and returns permitted fields.

**Rules/data:** Template entity includes id, collegeId, createdById, title, htmlTemplate, stylesConfig, createdAt, updatedAt. Manager is view-only. Student My Certificates is a separate owner-based page.

**Failure/alternate:** Deny unauthorized role; deny/safe-not-found for foreign template; empty state if no records; invalid filters follow shared contract; suspended/deactivated actor follows Module 1.

**Acceptance:** HOD/Manager see only authorized records; Manager cannot mutate; Finance denied; client-supplied scope cannot widen access; server checks authorization.

**QA:** CERT-QA-001–006.

## CERT-F002 — Create Certificate Template

**Requirement status:** RBAC create permission source-confirmed for SUPER_ADMIN/HOD; UI field details schema-derived  
**Specification status:** Decision required  
**Actors:** SUPER_ADMIN, HOD  
**Entry:** Certificate Templates → Create (page control not explicitly mapped)

**Purpose:** Create a template associated with College and creator.

**Preconditions:** Active authenticated actor; create permission; authorized College; create UI/action confirmed.

**Inputs**

| Field | Source | Type | Required | Validation |
|---|---|---|---|---|
| collegeId | Context | varchar | System/context | Server scope; never trust client |
| createdById | Auth context | varchar | System | Set from authenticated actor |
| title | Form | varchar(255) | TBD | Schema max length; required/trim/unique TBD |
| htmlTemplate | Form/editor | text | TBD | Sanitization/placeholder policy TBD |
| stylesConfig | Form/editor | JSON | TBD | JSON/schema/allowed style validation TBD |

**Main flow**
1. Authorized actor opens create action.
2. UI displays product-approved fields.
3. Actor enters template content.
4. Client may perform basic feedback; server performs all security/business validation.
5. Server derives College and creator from authorized context.
6. Server validates title length, HTML policy, and JSON syntax/schema.
7. Server creates CertificateTemplate and timestamps.
8. UI shows success and navigates according to approved behavior.

**Rules:** Creator is server-derived; College cannot be forged; htmlTemplate must not be unsafely executed; stylesConfig shape is not defined by source. Requiredness of fields remains TBD.

**Failure:** Permission/scope denial creates no row; invalid input rejected; duplicate-title and retry semantics TBD; DB failure must not show false success.

**Acceptance:** Only authorized HOD/Super Admin can create in scope; creator is auth-derived; title respects schema length; invalid JSON rejected; unauthorized attempts create no record.

**QA:** CERT-QA-007–014.

## CERT-F003 — Update Certificate Template

**Requirement status:** RBAC update permission source-confirmed for SUPER_ADMIN/HOD  
**Specification status:** Decision required  
**Actors:** SUPER_ADMIN, HOD  
**Entry:** Template detail/edit (entry control not explicitly mapped)

**Purpose:** Update an existing template in scope.

**Inputs:** templateId; approved editable fields title/htmlTemplate/stylesConfig. collegeId and createdById are not actor-editable in this draft (recommendation; decision pending).

**Main flow**
1. Actor opens authorized template.
2. UI loads current values.
3. Actor edits allowed fields.
4. Server rechecks permission and scope.
5. Server validates fields and updates permitted values plus updatedAt.
6. UI shows updated template.

**Rules:** HOD scope applies; Manager read-only. HOD edit of another HOD’s template is unresolved. Editing template after issuance may change rendered certificates unless snapshot/version policy is approved (CERT-D14).

**Failure/edge:** foreign template denied; invalid HTML/JSON rejected; stale edits follow Module 18; referenced certificates invoke approved snapshot/delete policy.

**Acceptance:** Only approved fields update; server authorization enforced; updatedAt changes on successful update; stale update is not silently lost under shared conflict policy.

**QA:** CERT-QA-015–020.

## CERT-F004 — Delete Certificate Template

**Requirement status:** SUPER_ADMIN delete permission source-confirmed; HOD delete not granted  
**Specification status:** Decision required  
**Actor:** SUPER_ADMIN only under supplied RBAC  
**Entry:** Template row/action (not explicitly mapped)

**Preconditions:** Super Admin authenticated; template exists; deletion semantics and referenced-certificate policy approved.

**Main flow**
1. Super Admin selects delete.
2. Confirmation behavior, if any, follows product decision.
3. Server rechecks permission and target.
4. Server checks template references and retention constraints.
5. If permitted, applies approved deletion semantics.
6. UI reports outcome.

**Rules:** Schema links CertificateTemplate to Certificate. Hard delete, soft delete, archive, restrict, or cascade are unspecified. Never assume deleting a template deletes certificates.

**Failure/edge:** all other roles denied; referenced template follows approved policy; concurrent issue/delete follows integrity rules; missing template safe-not-found.

**Acceptance:** non-Super Admin delete denied; issued certificate records not silently removed; approved retention/audit policy applied.

**QA:** CERT-QA-021–025.

## CERT-F005 — Preview/render Certificate Template

**Requirement status:** Derived candidate; preview not explicitly mapped  
**Specification status:** Decision required  
**Actors:** HOD/Super Admin candidate  
**Entry:** Template editor/preview candidate

**Purpose:** Preview a rendered template if product approves the feature.

**Inputs:** template or unsaved draft; sample data; HTML and style configuration.

**Candidate flow**
1. Actor requests preview.
2. System validates content and style configuration.
3. Renderer substitutes approved placeholders using safe sample data.
4. UI displays preview without issuing a certificate.
5. Actor returns to editing or exits.

**Decisions:** preview support CERT-D11; HTML policy CERT-D08; style schema CERT-D09; placeholders/escaping CERT-D10. Real Student data in preview is not approved.

**Acceptance:** invalid content does not execute scripts or expose sensitive data; preview does not create Certificate/issuedAt/PDF unless approved; render error distinct from issue error.

**QA:** CERT-QA-026–030.

## CERT-F006 — Issue/generate Certificate

**Requirement status:** Certificate schema/relationship source-confirmed; workflow details TBD  
**Specification status:** Decision required  
**Actors:** HOD/Super Admin candidate; RBAC Create action is not explicitly mapped to issue  
**Entry:** Issuance UI/event TBD

**Purpose:** Create a Certificate for an eligible Student using an authorized template.

**Preconditions:** approved issue permission; Student/College/template/source in scope; eligibility and trigger approved; rendering and artifact policy defined.

**Inputs**

| Field | Source/type | Required | Notes |
|---|---|---|---|
| studentId | Recipient/source event, varchar | Yes | Server validates scope |
| templateId | Selection, varchar | Yes | Must be valid and usable |
| collegeId | Context, varchar | System | Must match approved context |
| courseId | Source relation, varchar | TBD | Nullable/mutually exclusive policy TBD |
| contestId | Source relation, varchar | TBD | Nullable/mutually exclusive policy TBD |
| verificationCode | System, varchar | TBD | Format/uniqueness CERT-D16 |
| qrCodeUrl | System/service, varchar(1024) | TBD | CERT-D15/17 |
| pdfUrl | System/service, varchar(1024) | TBD | CERT-D15 |
| issuedAt | System timestamp | TBD | Issue semantics TBD |

**Candidate flow (not approved)**
1. Actor/event identifies recipient and achievement source.
2. Server checks permission/scope and approved eligibility.
3. Server validates source achievement and duplicate policy.
4. System loads approved template/version.
5. System renders and creates PDF/QR artifacts under approved architecture.
6. System creates Certificate with College, template, recipient, source refs, code, URLs, issuedAt.
7. UI confirms success; notifications only under Module 13 approved rules.

**Rules/gaps:** Eligibility CERT-D01; manual/automatic trigger D02; role mapping D03/D04; source relationship D12; duplicates/reissue D13; template snapshot D14; artifact architecture D15; verification code D16; audit/notification D21.

**Failure:** Ineligible/foreign recipient denied; invalid template fails; renderer/PDF/QR error must not show false success; timeout/retry duplicate behavior TBD; missing/conflicting course/contest source follows approved policy; partial/pending generation state must be approved.

**Acceptance:** server validates all scope and eligibility; client cannot forge recipient/College/issuer; issuedAt system-controlled; duplicate handling follows decision; issue, artifact generation, and notification outcomes remain distinguishable.

**QA:** CERT-QA-031–044.

## CERT-F007 — View Student’s own Certificates

**Requirement status:** Student My Certificates page and RBAC ownership source-confirmed  
**Specification status:** Detailed draft  
**Actor:** STUDENT  
**Entry:** My Certificates

**Purpose:** List and open certificate records owned by authenticated Student.

**Preconditions:** authenticated active Student; Module 1 account gate; Certificate read-own permission.

**Inputs:** Student identity from session; optional filters/pagination TBD; certificateId for detail, validated against owner.

**Main flow**
1. Student opens My Certificates.
2. Server derives Student ID from authenticated principal.
3. Server queries Certificate records belonging to that Student and applies tenant checks.
4. Server returns approved summary projection.
5. Student opens a certificate.
6. Server rechecks ownership and returns permitted detail.
7. UI displays certificate information.

**Rules:** Student sees own records only; related Course/Contest details require independent authorization; do not expose another Student’s metadata/PDF/QR URL.

**Failure/edge:** Forged studentId ignored/rejected; foreign certificate denied/safe-not-found; empty state copy TBD; inactive account follows Module 1; missing source relation display fallback TBD.

**Acceptance:** owner filter server-side; detail independently owner-checked; no private URL leakage; empty/error states distinguishable.

**QA:** CERT-QA-045–052.

## CERT-F008 — Open/download Certificate PDF

**Requirement status:** pdfUrl field source-confirmed; download workflow not mapped  
**Specification status:** Decision required  
**Actor:** Student owner; staff access TBD  
**Entry:** My Certificates → PDF/open candidate

**Purpose:** Access PDF artifact if product supports it.

**Inputs:** certificateId; PDF URL retrieved server-side (client URL is not proof of authorization).

**Candidate flow**
1. Student selects certificate.
2. Server checks owner/scope and approved certificate state.
3. System checks PDF artifact availability.
4. System opens/returns PDF through approved access mechanism.
5. UI handles unavailable/expired artifact per policy.

**Security:** Do not accept arbitrary client URL; deny cross-owner downloads; public verification does not automatically make PDF public; avoid unrestricted storage URL exposure if security decision requires protected access.

**Decisions:** artifact access/expiry/regeneration CERT-D15; user status/retention D22; staff certificate UI D25.

**Acceptance:** owner authorization; guessed IDs/URLs do not bypass access; missing/expired PDF produces controlled error; no silent regeneration unless approved.

**QA:** CERT-QA-053–058.

## CERT-F009 — Verify Certificate by code/QR

**Requirement status:** Derived candidate from schema fields; no verification workflow source-confirmed  
**Specification status:** Decision required  
**Actor/entry:** Public or authenticated verifier TBD; QR/code page not mapped

**Purpose:** Potential authenticity/status verification, if approved.

**Inputs:** verificationCode or QR target token; contract TBD.

**Candidate flow**
1. Verifier scans QR or enters code.
2. System validates input and approved verification access policy.
3. System locates certificate under approved code uniqueness policy.
4. System checks status under approved lifecycle (schema currently has no status field).
5. System returns only approved public/authorized fields.
6. Audit is recorded only if Module 15 approves it.

**Security/decisions:** public vs authenticated access CERT-D17; code format/uniqueness D16; status/revocation/expiry D18; public field list, anti-enumeration, rate limiting, response timing, and privacy TBD. QR URL may point to verification or an asset; source does not specify.

**Acceptance:** verification feature is not treated as confirmed; no private Certificate data disclosed; unknown/revoked/expired responses are defined only after decision; status and output contract approved before implementation.

**QA:** CERT-QA-059–068.

## Cross-module contract reconciliation

- **Module 1 — Authentication & User Lifecycle:** derive Student identity and active-account state from authenticated principal. Never accept a client-supplied studentId, issuer identity or College scope as authority. Certificate owner and status/role changes require Module 1 rules.
- **Module 2 — College / Department / Section:** enforce SUPER_ADMIN global, HOD College+Department, Manager assigned Section and Student own-record boundaries. Certificate source relationships do not widen scope.
- **Module 4 — Course & Enrollment:** CourseEnrollment.completedAt exists, but adopted Module 4 v1 completion is HOD-verified. Completion of a Course does not automatically trigger issuance; Module 12 must approve eligibility and trigger. Course completion is not implied by finishing an individual Problem/Lab/Assessment/Contest.
- **Module 11 — Contests:** Contest results and certificate eligibility are not the same rule. CON-D25 remains open; the historical top-three certificate flag is not target policy.
- **Module 13 — Notifications:** certificate-issued/revoked notification triggers, recipients, channels and retry are not defined in current sources. Do not infer a WebSocket or email event from the legacy report.
- **Module 15 — Audit Logging:** audit coverage for template changes, manual/automatic issue, reissue and revoke must align with AUD decisions; exact event catalog and sensitive metadata handling remain open.
- **Module 16/17 — Learning progress and state machines:** historical PathCompleted/RoomCompleted triggers require a target milestone definition and guard; FSM-D07/D12 remain open. Skip-to-completion fraud is reported as a critical legacy risk, not a target certificate rule.
- **Module 18 — Edge Cases & Failure Handling:** define idempotent issuance, event replay, renderer/QR/storage failure, partial artifacts, retry, dead-letter and recovery. The reported worker payload mismatch is historical evidence only; no BullMQ/S3 architecture is selected.

## Decision log## Decision log
| ID | Question | Status |
|---|---|---|
| CERT-D01 | Certificate eligibility and achievement source | Open |
| CERT-D02 | Manual vs automatic issue | Open |
| CERT-D03 | Issue permission by role | Open |
| CERT-D04 | HOD create/update: template vs certificate | Open |
| CERT-D05 | College/Department/Student scope details | Open |
| CERT-D06 | Manager view access: template, certificate, or both | Open |
| CERT-D07 | Title requiredness/uniqueness/normalization | Open |
| CERT-D08 | HTML allowlist/sanitization/script policy | Open |
| CERT-D09 | stylesConfig schema | Open |
| CERT-D10 | Placeholders/value resolution/escaping | Open |
| CERT-D11 | Preview support and preview data | Open |
| CERT-D12 | Course/Contest source and mutual exclusion | Open |
| CERT-D13 | Duplicate issue/reissue/idempotency | Open |
| CERT-D14 | Template snapshot/versioning | Open |
| CERT-D15 | PDF/QR generation, storage, access, expiry | Open |
| CERT-D16 | Verification code format/uniqueness/visibility | Open |
| CERT-D17 | Public vs authenticated verification and response fields | Open |
| CERT-D18 | Revocation/expiry/cancellation/status model | Open |
| CERT-D19 | Delete template with issued certificate references | Open |
| CERT-D20 | Course/Contest eligibility integration | Open |
| CERT-D21 | Audit and notification requirements | Open |
| CERT-D22 | Access after Student/College deactivation | Open |
| CERT-D23 | HOD editing own vs other HOD template | Open |
| CERT-D24 | Super Admin cross-College operations | Open |
| CERT-D25 | HOD/Manager issued-certificate pages | Open |
| CERT-D26 | Rendering/PDF/QR failure and retry behavior | Open |
| CERT-D27 | Does the target CertificateTemplate have a draft/published/archived lifecycle or active flag, and what actions/roles can transition it? The target conceptual schema contains no status/active fields. | Open |
| CERT-D28 | Reconcile target Certificate/Credential data model and supported achievement contexts: Course/Contest source fields vs legacy contextType/contextId, Path/Room/Event triggers, QR/privacy/revocation metadata. Define any schema delta/migration explicitly. | Open |
