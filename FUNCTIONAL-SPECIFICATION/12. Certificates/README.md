# DEZOLVER — Module 12: Certificates

**Document ID:** CERT-FS-001 | **Version:** 0.1 | **Status:** Detailed draft — product decisions open  
**Governing standard:** [Functional Specification Standard](../FUNCTIONAL-SPECIFICATION-STANDARD.md)

> Source facts, derived requirements, recommendations, and unresolved decisions remain separate. Committing this draft does not approve target behavior.

## Sources
- [Business Flow v2](../../docs/source-text/Business%20Flow%20v2.md)
- [Business Flow original](../../docs/source-text/Business%20Flow.md)
- [Page–DB Mapping v2](../../docs/source-text/Page%20-%20DB%20Mapping%20v2.md)
- [Page–DB Mapping original](../../docs/source-text/Page%20-%20DB%20Mapping.md)
- [RBAC](../../docs/source-text/RBAC.md)
- [Schema](../../docs/source-text/scheema.md)

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

## Evidence labels
- **Source-confirmed:** directly stated in source.
- **Derived:** necessary security/functional implication, not directly stated.
- **Recommendation:** proposed for review only.
- **TBD / Decision required:** unavailable in source.

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
- Module 4: Course source and any completion/eligibility evidence (not specified here).
- Module 11: Contest source/result/award eligibility (not specified here).
- Module 13: notification delivery if approved.
- Module 15: audit event definitions.
- Module 17: lifecycle/state machine if approved.
- Module 18: retry/conflict/storage failures.

## Source gaps
No source defines eligibility, automatic vs manual issuance, template syntax/placeholders, rendering service, PDF/QR generation, verification endpoint/public access, duplicate issuance, revocation, expiration, reissue, template versioning, notifications, or audit event names.

## Decisions
See [CERT decision log](./FUNCTION-SPECIFICATIONS.md#decision-log).

## QA and review
See [QA test cases](./QA-TEST-CASES.md). The module remains a **Detailed draft**, not product-approved. Completion of all modules precedes cross-module review and resolution of these decisions.
