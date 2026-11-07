# Module 12 — Certificates QA Test Cases

**Version:** 0.2 | **Status:** Detailed QA draft; decision-dependent scenarios marked  
**Related:** [Overview](./README.md) · [Function-level specs](./FUNCTION-SPECIFICATIONS.md)

> Tests that depend on unresolved decisions must be executed only after the decision is approved. For those cases, expected behavior is “apply approved rule”; it is not a default product requirement.

## Test fixtures
- SUPER_ADMIN global.
- HOD-A College A / Department A; HOD-B Department B in College A.
- Manager-A assigned Section A; Manager-B outside Section A.
- Finance-A assigned College A.
- Student-A College A; Student-B same College but distinct owner; Student-C outside College A.
- Template-A College A created by HOD-A; Template-B outside permitted scope; Template-Referenced has at least one Certificate reference.
- Certificate-A belongs to Student-A / College A / Template-A; Certificate-B belongs to Student-B; Certificate-Foreign belongs to Student-C.
- Source achievement fixtures only when eligibility/issue rules are approved.
- PDF/QR artifacts: available, missing, expired/invalid, and renderer/storage failure variants as appropriate.

## QA cases

| ID | Function | Scenario / steps | Expected result | Trace |
|---|---|---|---|---|
| CERT-QA-001 | CERT-F001 | HOD-A lists templates in Department A | Only authorized scope templates returned | RBAC |
| CERT-QA-002 | CERT-F001 | HOD-A requests Template-B from Department B | Denied or safe not-found; no data leakage | RBAC/scope |
| CERT-QA-003 | CERT-F001 | Manager-A lists templates | View-only authorized-scope result; template-vs-certificate scope decision applies | CERT-D06 |
| CERT-QA-004 | CERT-F001 | Manager-A attempts template mutation from UI/API | Denied | RBAC |
| CERT-QA-005 | CERT-F001 | Finance-A calls Certificate API | Denied | RBAC |
| CERT-QA-006 | CERT-F001 | Client changes College/Department filter to foreign scope | Cannot widen access | RBAC |
| CERT-QA-007 | CERT-F002 | HOD-A creates template in authorized College | Created only if required create UI/contract approved | RBAC |
| CERT-QA-008 | CERT-F002 | Student/Manager/Finance attempts create | Denied; no template row | RBAC |
| CERT-QA-009 | CERT-F002 | HOD-A submits foreign collegeId | Denied or server overrides to authorized scope per approved contract | RBAC |
| CERT-QA-010 | CERT-F002 | Client supplies another creatorId | Server uses authenticated creator; forged value rejected/ignored | Schema/security |
| CERT-QA-011 | CERT-F002 | title exceeds 255 chars | Reject; no record | Schema |
| CERT-QA-012 | CERT-F002 | title empty/whitespace | Requiredness/trim policy currently TBD; test after CERT-D07 | CERT-D07 |
| CERT-QA-013 | CERT-F002 | Duplicate title within College | Apply approved uniqueness/case rule; currently TBD | CERT-D07 |
| CERT-QA-014 | CERT-F002 | malformed stylesConfig JSON | Reject; no invalid JSON persisted | Schema |
| CERT-QA-015 | CERT-F003 | HOD-A edits Template-A | Allowed fields update in scope | RBAC |
| CERT-QA-016 | CERT-F003 | HOD-A edits Template-B | Denied | RBAC |
| CERT-QA-017 | CERT-F003 | Manager edits Template-A | Denied | RBAC |
| CERT-QA-018 | CERT-F003 | HOD-A edits template created by HOD-B | Apply approved creator/department ownership policy; currently TBD | CERT-D23 |
| CERT-QA-019 | CERT-F003 | HOD attempts to change collegeId/createdById | Cannot change system ownership fields absent approved policy | Security/derived |
| CERT-QA-020 | CERT-F003 | Two actors update same template concurrently | Shared Module 18 stale-update/conflict policy; no silent lost update if conflict detection applies | Module 18 |
| CERT-QA-021 | CERT-F004 | Super Admin deletes unreferenced Template-A | Apply approved deletion semantics | RBAC/CERT-D19 |
| CERT-QA-022 | CERT-F004 | HOD deletes Template-A | Denied; HOD RBAC has no delete | RBAC |
| CERT-QA-023 | CERT-F004 | Manager/Student/Finance deletes template | Denied | RBAC |
| CERT-QA-024 | CERT-F004 | Super Admin deletes template referenced by issued Certificate | Apply approved restrict/archive/delete policy; never silently cascade by assumption | CERT-D19 |
| CERT-QA-025 | CERT-F004 | Delete request repeated or template already absent | Shared idempotency/not-found behavior; no collateral delete | Module 18 |
| CERT-QA-026 | CERT-F005 | Authorized actor previews valid template | Preview behavior only if feature approved | CERT-D11 |
| CERT-QA-027 | CERT-F005 | Preview includes untrusted HTML/script | Unsafe script not executed; approved sanitization policy applied | CERT-D08/security |
| CERT-QA-028 | CERT-F005 | Preview contains unknown placeholder | Apply approved placeholder error/fallback policy; TBD | CERT-D10 |
| CERT-QA-029 | CERT-F005 | Preview uses Student personal data | Do not expose real data unless approved; safe sample policy applies | Privacy/CERT-D11 |
| CERT-QA-030 | CERT-F005 | Preview fails to render | Show controlled preview error; no Certificate issuance | Derived |
| CERT-QA-031 | CERT-F006 | Authorized issuer issues certificate for eligible Student/source | Certificate created according to approved issue workflow | CERT-D01–04 |
| CERT-QA-032 | CERT-F006 | Student/Manager/Finance attempts issuance | Deny unless explicit future RBAC decision grants action | RBAC/CERT-D03 |
| CERT-QA-033 | CERT-F006 | HOD issues for Student outside College/Department scope | Denied | RBAC/scope |
| CERT-QA-034 | CERT-F006 | Actor selects unauthorized template | Denied | RBAC |
| CERT-QA-035 | CERT-F006 | Achievement source does not satisfy eligibility | Apply approved eligibility; no issue if ineligible | CERT-D01 |
| CERT-QA-036 | CERT-F006 | Course and Contest IDs both supplied | Apply approved source cardinality rule; currently TBD | CERT-D12 |
| CERT-QA-037 | CERT-F006 | Course/Contest source missing | Apply approved required-source rule; currently TBD | CERT-D12 |
| CERT-QA-038 | CERT-F006 | Source achievement is incomplete/unfinalized | Apply approved dependency policy; do not infer eligibility | CERT-D20 |
| CERT-QA-039 | CERT-F006 | Same issue request retried after timeout | Duplicate/idempotency behavior per approved policy; no accidental duplicate | CERT-D13/Module 18 |
| CERT-QA-040 | CERT-F006 | Two concurrent issue requests for same Student/achievement | Apply uniqueness/duplicate issuance decision | CERT-D13 |
| CERT-QA-041 | CERT-F006 | Template edited between issue request and rendering | Snapshot/current-version policy applies | CERT-D14 |
| CERT-QA-042 | CERT-F006 | Template HTML/style invalid | Issuance fails safely; no false success | CERT-D08/09 |
| CERT-QA-043 | CERT-F006 | PDF generation fails after certificate data prepared | Apply approved transactional/pending/retry policy; no misleading issued status | CERT-D15/26 |
| CERT-QA-044 | CERT-F006 | QR generation/storage fails | Apply approved issue consistency/retry policy | CERT-D15/26 |
| CERT-QA-045 | CERT-F007 | Student-A opens My Certificates | Only Certificate-A / own records returned | RBAC ownership |
| CERT-QA-046 | CERT-F007 | Student-A supplies Student-B studentId | Server derives auth Student; B data not returned | RBAC ownership |
| CERT-QA-047 | CERT-F007 | Student-A opens Certificate-B | Denied/safe not-found | RBAC ownership |
| CERT-QA-048 | CERT-F007 | Student-A opens Certificate-Foreign | Denied/safe not-found | RBAC/tenant |
| CERT-QA-049 | CERT-F007 | Student-A list has no certificates | Empty state; no unrelated records | UX contract TBD |
| CERT-QA-050 | CERT-F007 | Student account suspended/deactivated | Apply Module 1 account-state policy | Module 1 |
| CERT-QA-051 | CERT-F007 | Certificate has missing Course/Contest relation | Certificate ownership still enforced; related display fallback per approved policy | Schema |
| CERT-QA-052 | CERT-F007 | Client alters certificateId during detail request | Server rechecks ownership; no foreign data | RBAC ownership |
| CERT-QA-053 | CERT-F008 | Student-A opens/downloads Certificate-A PDF | Allowed only through approved artifact access flow | CERT-D15 |
| CERT-QA-054 | CERT-F008 | Student-A requests Student-B PDF | Denied; no PDF URL/artifact leakage | RBAC ownership |
| CERT-QA-055 | CERT-F008 | User guesses PDF URL or certificate ID | Access cannot bypass authorization | Security |
| CERT-QA-056 | CERT-F008 | PDF artifact missing | Controlled unavailable state; no false success | CERT-D15 |
| CERT-QA-057 | CERT-F008 | PDF URL expired or storage access denied | Apply approved expiry/refresh policy; no unrestricted fallback | CERT-D15 |
| CERT-QA-058 | CERT-F008 | PDF open/download retried | No unintended certificate reissue/regeneration unless approved | CERT-D15 |
| CERT-QA-059 | CERT-F009 | Verification page/code lookup is attempted before feature approval | No verification behavior assumed enabled | CERT-D17 |
| CERT-QA-060 | CERT-F009 | Valid code supplied after verification feature approved | Return only approved fields/status | CERT-D16/17 |
| CERT-QA-061 | CERT-F009 | Invalid or unknown code | Apply approved response; prevent certificate data enumeration | CERT-D16/17 |
| CERT-QA-062 | CERT-F009 | Code is duplicated in data | Resolve according to uniqueness policy; no arbitrary certificate selection | CERT-D16 |
| CERT-QA-063 | CERT-F009 | QR scanned | QR target and disclosure follow approved contract; URL field alone does not define behavior | CERT-D15/17 |
| CERT-QA-064 | CERT-F009 | Student/private fields requested via public verify | Do not disclose beyond approved public projection | CERT-D17/privacy |
| CERT-QA-065 | CERT-F009 | Verification request rate/guessing attack | Apply approved anti-enumeration/rate-limit controls | Security decision |
| CERT-QA-066 | CERT-F009 | Certificate revoked/expired (if lifecycle added) | Return status per approved lifecycle and disclosure policy | CERT-D18 |
| CERT-QA-067 | CERT-F009 | Verification service logs request | Log only approved metadata; no unnecessary PII/code secrets | Module 15 |
| CERT-QA-068 | CERT-F009 | Foreign tenant certificate code queried | No cross-tenant private disclosure; public verification policy governs | RBAC/CERT-D17 |

| CERT-QA-069 | CERT-F002/F003 | Template fields are changed to legacy `name`, `kind`, or single `design` JSON payload | Keep target conceptual fields `title`, `htmlTemplate`, `stylesConfig`; do not silently substitute legacy schema | Target schema / legacy report / CERT-D28 |
| CERT-QA-070 | CERT-F006/F007/F009 | API assumes target Certificate has legacy `contextType`, `contextId`, `metadata`, `qrCode`, `isPublic`, `status`, `revokedAt`, or `revocationReason` | Do not rely on those fields absent explicit target schema decision/migration; keep `verificationCode` distinct from legacy `qrCode` | Target schema / CERT-D16–18/D28 |
| CERT-QA-071 | CERT-F001–F004 | Template list/create/update/delete flow assumes legacy `draft`/`published` status or `active` field | Do not assume template lifecycle/default/active semantics; follow explicit target decision and schema contract | Target schema / CERT-D27 |
| CERT-QA-072 | CERT-F005 | Preview endpoint is added solely because legacy `POST /certificate-templates/:id/preview` existed | Preview remains a derived feature until approved; no new target route/contract is inferred | CERT-D11 |
| CERT-QA-073 | CERT-F006 | Issuance-rule API/table is treated as available because legacy routes existed | Do not assume IssuanceRule exists; historical report itself says repository calls were stubbed and table absent | LEGACY-SPEC-REPORTED / target schema / CERT-D02/D20/D28 |
| CERT-QA-074 | CERT-F006 | Course completion automatically issues a certificate on `completedAt` without checking Module 4 completion policy and CERT eligibility | Module 4 completion is HOD-verified in v1; automatic certificate trigger remains a separate CERT-D01/D02/D20 decision | Module 4 / CERT-D01/D02/D20 |
| CERT-QA-075 | CERT-F006 | Contest end automatically issues certificates to legacy top-three or all registered participants | Do not import legacy Event rules; Module 11 CON-D25 and Module 12 eligibility decision must govern | Module 11 / CERT-D01/D12/D20 |
| CERT-QA-076 | CERT-F006 | Legacy `PathCompleted` event issues target Certificate without a target path/achievement relation and eligibility contract | No target path trigger is confirmed; align Modules 16/17 and CERT-D01/D20/D28 before enabling | LEGACY-SPEC-REPORTED / Modules 16–17 |
| CERT-QA-077 | CERT-F006 | Legacy `RoomCompleted` event issues target Certificate without target room relation or completion guard | No target room trigger is confirmed; do not assume Room is a target Certificate source | LEGACY-SPEC-REPORTED / FSM-D07 / CERT-D01/D28 |
| CERT-QA-078 | CERT-F006 | Legacy `EventCompleted` consumer iterates all registered participants and issues Certificates | Target ContestParticipant/result eligibility and award criteria remain open; no auto-issue from registration alone | LEGACY-SPEC-REPORTED / CON-D25 / CERT-D01/D20 |
| CERT-QA-079 | CERT-F006 | Background issuer and renderer use incompatible payloads or missing recipient/title values | Where asynchronous rendering is selected, producer/consumer contract must be explicit and contract-tested; legacy crash is a regression warning, not target architecture | LEGACY-SPEC-REPORTED / CERT-D26 / Module 18 |
| CERT-QA-080 | CERT-F008 | `.html` artifact with `text/html` is returned as though it were a binary PDF solely because field is named `pdfUrl` | Artifact MIME/type and download behavior must match the approved CERT-D15 contract; do not mislabel HTML as PDF | LEGACY-SPEC-REPORTED / CERT-D15 |
| CERT-QA-081 | CERT-F009 | `verificationCode` is assumed to be a 32-character hex token generated from 16 random bytes | Do not impose legacy token length/encoding/uniqueness or expose code until CERT-D16 is approved | LEGACY-SPEC-REPORTED / CERT-D16 |
| CERT-QA-082 | CERT-F009 | Anonymous verification is enabled using legacy QR or UUID routes without target approval | Public vs authenticated access and public fields remain CERT-D17; no legacy route is assumed enabled | LEGACY-SPEC-REPORTED / CERT-D17 |
| CERT-QA-083 | CERT-F009 | Public verification by predictable/guessable certificate UUID reveals recipient or award metadata | Apply only approved access model; no public lookup/response fields without CERT-D17 and Module 15 review | CERT-D17 / AUD decisions |
| CERT-QA-084 | CERT-F007/F008 | Legacy `isPublic` toggle or public certificate directory is assumed in target Student My Certificates | Target Certificate schema has no isPublic; public visibility is not currently source-confirmed | LEGACY-SPEC-REPORTED / target schema / CERT-D17/D28 |
| CERT-QA-085 | CERT-F006 | Legacy `issued` → `revoked` lifecycle, revoke reason/time, or irreversible revocation is added without target lifecycle approval | Target schema has no status/revocation fields; keep blocked pending CERT-D18 and FSM-D12 | LEGACY-SPEC-REPORTED / CERT-D18 / FSM-D12 |
| CERT-QA-086 | CERT-F006 | Reissue creates a new certificate/QR and preserves original based only on legacy endpoint behavior | Reissue, duplicate issuance, history and original validity are CERT-D13/D18 decisions; do not assume legacy semantics | LEGACY-SPEC-REPORTED / CERT-D13/D18 |
| CERT-QA-087 | CERT-F001/F006/F007 | Staff `GET /certificates` all-records console or `/platform/credentials` page is assumed from legacy frontend despite no target Page–DB mapping | Target page mapping confirms HOD Certificate Templates and Student My Certificates; additional staff issuance pages/permissions remain CERT-D21/D25 | LEGACY-SPEC-REPORTED / page mapping / CERT-D25 |
| CERT-QA-088 | CERT-F006/F007 | WebSocket `certificate:issued` event is assumed to notify Student and authorize a private user channel | Notification trigger/recipient/channel and realtime delivery are Module 13 decisions; no legacy WebSocket event is adopted | LEGACY-SPEC-REPORTED / Module 13 / CERT-D21 |

## Coverage summary
88 test cases cover:
- template read/create/update/delete and role scope;
- template HTML/styles security and preview;
- issuance eligibility, duplicate/idempotency, source relations, renderer/PDF/QR failures;
- Student ownership and certificate detail;
- PDF access controls;
- verification privacy, enumeration, QR, and lifecycle decision dependencies;
- target-versus-legacy schema, lifecycle, milestone triggers, worker payload, HTML/PDF, public verification, privacy, revocation/reissue, and notification regressions.

CERT-QA-069–088 explicitly keep legacy Credentials fields, routes, triggers, bugs, and side effects from being silently imported. Cases are defined only; they have not been executed.

Policy-dependent tests must remain blocked until the corresponding decision is approved.
