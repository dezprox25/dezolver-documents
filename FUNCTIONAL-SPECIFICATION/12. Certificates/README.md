# DEZOLVER — Module 12: Certificates

**Document ID:** CERT-FS-001 | **Version:** 0.2 | **Status:** Detailed draft — source-first legacy cross-check; product decisions open  
**Governing standard:** [Functional Specification Standard](../FUNCTIONAL-SPECIFICATION-STANDARD.md)

> Source facts, derived requirements, recommendations, and unresolved decisions remain separate. Committing this draft does not approve target behavior.

## Sources
- [Business Flow v2](../../docs/source-text/Business%20Flow%20v2.md)
- [Business Flow original](../../docs/source-text/Business%20Flow.md)
- [Page–DB Mapping v2](../../docs/source-text/Page%20-%20DB%20Mapping%20v2.md)
- [Page–DB Mapping original](../../docs/source-text/Page%20-%20DB%20Mapping.md)
- [RBAC](../../docs/source-text/RBAC.md)
- [Schema](../../docs/source-text/scheema.md)
- [Historical Module 12 forensic specification](../../functional-spec-from-old%20code/module-12-certificates/SPECIFICATION.md) and [legacy summary](../../functional-spec-from-old%20code/module-12-certificates/README.md) — LEGACY-SPEC-REPORTED; raw application code not independently retrieved during this pass.

## Purpose
The source-mapped areas are HOD **Certificate Templates** and Student **My Certificates**. The schema defines CertificateTemplate and Certificate entities, with template content/configuration, certificate recipient/source references, verification code, QR URL, PDF URL, and issue timestamp.

Business Flow v2 and the original Business Flow do not describe certificate-specific issuance or verification workflows. This module therefore documents supported entity/page/permission facts and identifies unknown behavior rather than inventing it.

## Scope
- Certificate Template view/create/update/delete permissions from RBAC.
- Student My Certificates and ownership validation.
- Template and Certificate fields and relationships.
- Certificate issue/generation as a decision-dependent workflow.
- PDF/QR/verification handling as decision-dependent behavior.
- QA, source traceability, dependencies, and open decisions.

Out of scope: deciding certificate award criteria, completion rules, HTML rendering architecture, public verification policy, revocation/expiry, or changing the schema.

## Evidence summary
| Source | Evidence | Limitation |
|---|---|---|
| Business Flow v2 / original | No Certificate-specific workflow | Trigger, eligibility, and automation not defined |
| Page–DB Mapping v2 | HOD Certificate Templates → CertificateTemplate, User, College. Student My Certificates → Certificate, CertificateTemplate, Course, Contest, User, College | Does not specify UI behavior or issuance criteria |
| Original Page–DB Mapping | HOD Certificate Templates → CertificateTemplate, User, College. Student My Certificates → Certificate, CertificateTemplate, Course, Contest, College | No full lifecycle/ownership workflow |
| RBAC page/API matrices | SUPER_ADMIN V/C/U/D; FINANCE_TEAM none; HOD V/C/U*; MANAGER V*; STUDENT V OWN | Does not distinguish template CRUD vs issue/revoke action |
| Schema | CertificateTemplate and Certificate entities/relations/fields | No state machine, nullable rules, issuance eligibility, revocation, or verification contract |
| Functional Specification Standard | Function-level specifications, evidence labels, QA, decisions, review status | Does not define Certificate behavior |

## Legacy evidence crosswalk — reported Credentials subsystem

**Evidence status: LEGACY-SPEC-REPORTED.** The historical Module 12 forensic report labels itself `COMPLETED & FORENSICALLY VERIFIED` and cites `src/credentials/`, `prisma/credentials/`, worker, S3/CDN and frontend files. This review inspected the report artifacts in the repository; it did not independently retrieve the raw application source files. The behavior below is attributed to that report and is not fresh code verification or target approval.

The historical implementation is described as a **Credentials** subsystem. Its schema and service flows differ materially from the current conceptual **Certificate / CertificateTemplate** model. Keep the two evidence sets separate:

| Area | Current target source | Historical Credentials behavior reported | Required treatment in this review |
|---|---|---|---|
| Template model | `CertificateTemplate`: id, collegeId, createdById, title, htmlTemplate, stylesConfig, createdAt, updatedAt. | `CertificateTemplate`: tenantId, name, kind, design JSON, status, active, createdByUserId, timestamps. | Do not import `tenantId`, `name`, `kind`, `design`, `status` or `active` into the target model without a schema/product decision (CERT-D27/28). |
| Certificate model | id, verificationCode, collegeId, templateId, studentId, courseId, contestId, qrCodeUrl, pdfUrl, issuedAt. | id, tenantId, userId, templateId, contextId/contextType, metadata, qrCode, pdfUrl, isPublic, status, issuedAt, revokedAt, revocationReason. | Target `verificationCode` is not proven to equal legacy `qrCode`; `qrCodeUrl` is not necessarily a file URL or verification URL. Legacy context/status/privacy/revocation fields are not target source fields (CERT-D16–D18, D28). |
| Template operations | HOD Certificate Templates page; RBAC V/C/U* for HOD, V* for Manager, Super Admin Full CRUD. No target publish status is shown. | GET/POST templates; preview and publish route reported. Old templates moved draft → published and automated issuance filtered published+active. | List/create/update/delete functions are specified from target page/RBAC/schema, but preview/publish/active lifecycle remain decision-dependent. Old route existence is historical only. |
| Issuance | Certificate entity has Course and Contest source fields. Business Flow contains no Certificate issuance flow. | Manual issue routes and event-driven handlers for `PathCompleted`, `EventCompleted`, `RoomCompleted`; report says issued for registered Event participants. | Do not assume manual/automatic issuance, milestone triggers, Path/Room contexts, Contest top-three eligibility, or a rules engine. Course/Contest source and criteria remain CERT-D01/02/12/20/28. |
| Issuance rules | No IssuanceRule entity/page is present in the target schema or page mapping supplied for this module. | `/issuance-rules` GET/POST/DELETE routes were reported, but the report says the `issuance_rules` model/table was absent and repository calls were stubs. | Treat issuance-rule UI/API as neither target-confirmed nor working legacy behavior; it remains outside the target function inventory pending decision. |
| Artifact generation | Target Certificate has `pdfUrl` and `qrCodeUrl` fields; no renderer, storage provider, queue or binary format is specified. | BullMQ `certificate` worker reportedly rendered HTML and uploaded `.html` to S3 as `text/html`, then wrote its URL into `pdfUrl`; no binary PDF compilation was reported. | A field named `pdfUrl` does not prove a PDF exists. Do not claim binary PDF generation or adopt S3/CDN/BullMQ without CERT-D15/26. |
| Worker handoff | No worker/job contract is specified in target sources. | Report says producer passed `{certificateId, templateId, userId, qrCode}` while consumer expected recipient/title/date values; worker failed on undefined `recipientName`. | This is a reported historical defect. Any future async pipeline must define a complete producer/consumer contract, but no queue implementation is prescribed here (CERT-D26; Module 18). |
| QR and verification | `verificationCode` plus `qrCodeUrl` fields exist; no format, uniqueness, endpoint, public access or disclosure contract is defined. | Report says a 16-byte random value was hex-encoded to 32 characters and unique; public routes verified by QR token or certificate UUID. | Do not import token length/encoding, uniqueness constraint, public routes or response fields as target behavior (CERT-D16/17). |
| Privacy and revocation | Target schema has no `isPublic`, status, revokedAt or revocationReason fields. | Student privacy toggle and issued → revoked lifecycle were reported; revocation reason/time recorded; reissue created a new record/QR. | Public directory visibility, revocation, expiry, reissue and irreversible lifecycle are not target-confirmed; keep decision-bound (CERT-D13/18/22/27/28; Module 17 FSM-D12). |
| Admin/real-time UI | Target Page Mapping confirms HOD Certificate Templates and Student My Certificates only. | Report describes public verification pages, Student detail/download/privacy toggle, admin issued-certificate console, rules UI and WebSocket `certificate:issued`. | Historical routes/screens and WebSocket event do not expand target page scope. HOD/Manager issued-certificate management and real-time delivery remain decision items (CERT-D21/25, Module 13). |

### Legacy function/route crosswalk

| Target function | Historical analogue reported | Boundary / unresolved item |
|---|---|---|
| CERT-F001 — List templates | `GET /certificate-templates` | Historical route only; target role/scope remains RBAC-driven. |
| CERT-F002 — Create template | `POST /certificate-templates` | Historical route; target fields follow conceptual schema, not legacy `name/kind/design`. |
| CERT-F003 — Update template | No update route identified in the report's listed route inventory. | Target RBAC grants HOD update; exact target endpoint/fields remain a technical contract. |
| CERT-F004 — Delete template | No template-delete route identified in the report's route inventory. | Target Super Admin Delete permission exists; referenced-template behavior remains CERT-D19. |
| CERT-F005 — Preview/render | `POST /certificate-templates/:id/preview` | Derived target candidate; renderer, preview data and safety rules remain CERT-D08–D11/15. |
| CERT-F006 — Issue/generate | `POST /certificates`, `/certificates/manual`, `reissue`, `revoke`; milestone consumers | Manual/automatic issue, role, sources, artifact and lifecycle are not target-approved. |
| CERT-F007 — View own Certificates | `GET /me/certificates`, `GET /me/certificates/:id` | Historical owner flow; target Student own-record access is supported by RBAC/page mapping. |
| CERT-F008 — Open/download PDF | `GET /me/certificates/:id/download` reportedly redirected to CDN | `pdfUrl` exists in target schema, but format, access mechanism and URL privacy remain CERT-D15. |
| CERT-F009 — Verify by code/QR | Public `/certificates/verify/:qrCode` and `/verify/c/:certificateId` | Derived target candidate only; target public verification is not source-confirmed (CERT-D16/17/18). |

**Historical routes/features without a mapped target function:** template publish; issuance-rule CRUD; Student public-visibility toggle; reissue/revoke endpoints as explicit state-changing operations; staff all-certificates console; public verification pages/routes; WebSocket notification. Do not create target functions from these report-only features without decisions and source/page/schema support.
## Evidence labels
- **Source-confirmed:** directly stated in source.
- **Derived:** necessary security/functional implication, not directly stated.
- **Recommendation:** proposed for review only.
- **TBD / Decision required:** unavailable in source.
- **LEGACY-SPEC-REPORTED:** a behavior stated in the historical Module 12 forensic report; it has not been independently verified against raw application code in this review.

## Actors and permissions
| Role | Certificate permission | Scope / interpretation |
|---|---|---|
| SUPER_ADMIN | V/C/U/D; API Full CRUD | Global scope per RBAC |
| FINANCE_TEAM | None | Deny Certificate access under current RBAC |
| HOD | V/C/U* | Authorized College/Department scope; no delete permission shown |
| MANAGER | V* | View only, authorized scope |
| STUDENT | V OWN | Own certificate records only; server ownership check required |

RBAC uses `*` to indicate scope restriction. Frontend visibility is not authorization. Certificate ownership is explicitly included in the RBAC security rules. Resource-level permissions do not clarify which actions apply to template versus issued certificate; this is a decision.

## Functional inventory
See [Function-level specifications](./FUNCTION-SPECIFICATIONS.md).
| ID | Function | Actor | Type | Status |
|---|---|---|---|---|
| CERT-F001 | List/view templates | HOD/Super Admin; Manager scoped view | Read | Draft |
| CERT-F002 | Create template | HOD/Super Admin | Create | Draft |
| CERT-F003 | Update template | HOD/Super Admin | Update | Draft |
| CERT-F004 | Delete template | Super Admin per RBAC | Delete | Decision required |
| CERT-F005 | Preview/render template | HOD/Super Admin candidate | Derived | Decision required |
| CERT-F006 | Issue/generate certificate | HOD/Super Admin candidate under Create permission | Workflow | Decision required |
| CERT-F007 | View own certificates | Student | Read | Source-confirmed page + ownership |
| CERT-F008 | Open/download PDF | Student owner; staff scope TBD | Read/download | PDF URL field exists; behavior TBD |
| CERT-F009 | Verify certificate by code/QR | Actor/page TBD | Verification | Derived candidate only; no workflow source |

## Data and schema
### CertificateTemplate
| Field | Type | Source meaning | Unknowns |
|---|---|---|---|
| id | varchar | Template identifier | ID generation/unique constraint |
| collegeId | varchar | College relation | Nullable/FK behavior |
| createdById | varchar | Creator User relation | Creator edit rights |
| title | varchar(255) | Template title | Requiredness, uniqueness, trim/case policy |
| htmlTemplate | text | HTML template | Allowed markup, sanitization, placeholders |
| stylesConfig | json | Style config | JSON schema, allowed styles |
| createdAt | timestamp | Created time | timezone/default |
| updatedAt | timestamp | Updated time | update semantics |

### Certificate
| Field | Type | Source meaning | Unknowns |
|---|---|---|---|
| id | varchar | Certificate identifier | ID format |
| verificationCode | varchar | Verification code | Unique? format? public or private? |
| collegeId | varchar | College relation | Nullability/FK behavior |
| templateId | varchar | Template relation | Snapshot/version behavior |
| studentId | varchar | Recipient Student | Owner check required |
| courseId | varchar | Course association | Nullable? mutual exclusion with contestId? |
| contestId | varchar | Contest association | Nullable? mutual exclusion with courseId? |
| qrCodeUrl | varchar(1024) | QR URL | Payload/target/expiry/privacy |
| pdfUrl | varchar(1024) | PDF URL | Storage/access/expiry/regeneration |
| issuedAt | timestamp | Issue time | Trigger, timezone, immutability |

Schema relations: College→CertificateTemplate; User→CertificateTemplate creator; CertificateTemplate→Certificate; College→Certificate; User→Certificate recipient.

Schema does not show Certificate status, revocation/expiry, issuer field, template version/snapshot, issuance history, verification-code uniqueness constraint, or PDF/QR processing state. Do not assume these exist.

## Business rules
| ID | Rule | Evidence |
|---|---|---|
| CERT-BR-001 | Template is College-associated and creator-attributed | Schema |
| CERT-BR-002 | Certificate references College, template, Student, and Course/Contest fields | Schema |
| CERT-BR-003 | Student certificate records are owner-controlled | RBAC |
| CERT-BR-004 | Backend enforces role permission and scope | RBAC |
| CERT-BR-005 | Finance Team has no Certificate permission | RBAC |
| CERT-BR-006 | HOD V/C/U*, Manager V*, Student V OWN, Super Admin Full CRUD | RBAC; resource/action ambiguity open |
| CERT-BR-007 | title max length 255; htmlTemplate text; stylesConfig JSON | Schema |
| CERT-BR-008 | verificationCode, qrCodeUrl, pdfUrl fields exist | Schema; semantics TBD |

## Dependencies
- Module 1: authentication, active user, Student identity/ownership.
- Module 2: College/Department/Section authorization scope.
- Module 4: CourseEnrollment/Course completion source. CourseEnrollment.completedAt exists, but Module 4's adopted v1 completion is HOD-verified; Course completion does not automatically issue a Certificate. Module 12 must decide eligibility/trigger separately.
- Module 11: Contest/ContestScoreRank; CON-D25 is open and explicitly does not approve the legacy top-three certificate rule. Contest finalization and eligibility belong to Module 11.
- Module 13: notification triggers, recipients and channels; certificate-issued/ revoked notifications are not source-defined and remain NOTIF decisions.
- Module 15: credential issue/revoke/template audit coverage; exact events, fields, redaction and retention remain AUD decisions.
- Module 17: certificate issue/revoke/reissue state machine; FSM-D12 remains open. Do not treat historical issued/revoked as target states.
- Module 18: renderer/worker/storage retries, idempotency, partial issuance, outbox and recovery; do not transfer reported BullMQ/S3 settings or payload defect as target architecture.

## Source gaps
No target source defines eligibility criteria, manual vs automatic issuance, achievement-source cardinality, template publication/active state, HTML placeholder syntax, rendering service, binary PDF/QR generation, verification endpoint/public access, duplicate issuance, revocation, expiration, reissue, template versioning, or notification/audit event names. The target schema contains Course and Contest source fields only; legacy Path/Room/Event triggers are not target evidence. The historical report also describes legacy-only fields/operations and implementation defects; see the LEGACY-SPEC-REPORTED crosswalk. The conceptual schema does not prove deployed constraints or artifact generation.

## Decisions
See [CERT decision log](./FUNCTION-SPECIFICATIONS.md#decision-log).

## QA and review
See [QA test cases](./QA-TEST-CASES.md). The module remains a **Detailed draft**, not product-approved. Completion of all modules precedes cross-module review and resolution of these decisions. CERT-D01–D28 remain open in this source-first pass; no ADOPTED-DECISIONS file is created.
