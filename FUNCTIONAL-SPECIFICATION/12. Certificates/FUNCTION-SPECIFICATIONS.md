# Module 12 — Certificates: Function-Level Specifications

**Document ID:** CERT-FS-FUNCTIONS-001 | **Version:** 0.1 | **Status:** Detailed draft; decisions open  
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

## Decision log
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
