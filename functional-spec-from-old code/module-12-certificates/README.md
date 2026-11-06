# Module 12 — Certificates & Credentials

## 1. Module Executive Summary

Module 12 governs the digital credentialing, certificate templating, automated event-driven issuance, public cryptographic verification, and revocation subsystem of the Dezolver platform.

A thorough forensic investigation of the legacy codebase (`prisma/credentials/`, `src/credentials/`, `src/worker/consumers/certificate-generator.consumer.ts`, and `frontend/src/pages/credentials/`) reveals key functional strengths alongside significant architectural defects:

1. **Dual Verification Architecture (Public & Non-Authenticated):** The platform exposes two public verification ingress routes: `GET /certificates/verify/:qrCode` (by 32-character hex QR code) and `GET /verify/c/:certificateId` (by primary certificate UUID). Both endpoints bypass JWT authentication and return verification status and metadata.
2. **Event-Driven Automatic Issuance Pipeline:** `CredentialConsumers` automatically triggers certificate generation across three platform milestones:
   - `PathCompleted`: On completion of a curated learning track.
   - `EventCompleted`: For all registered participants upon conclusion of a technical event or competition.
   - `RoomCompleted`: On completion of a practical curriculum room.
3. **Critical Worker Payload Mismatch (Job Failure):** `CredentialsService.issue` enqueues jobs to BullMQ queue `certificate` with payload `{ certificateId, templateId, userId, qrCode }`. However, `CertificateGeneratorConsumer` expects `{ recipientName, title, issuedAt, ... }`. Because `recipientName` is undefined, `this.escHtml(data.recipientName)` crashes with `TypeError: Cannot read properties of undefined (reading 'replace')`, preventing automated background generation!
4. **HTML Storage Instead of Binary PDF:** Despite Swagger annotations promising binary PDF generation and database column named `pdf_url`, `CertificateGeneratorConsumer` renders a static HTML string via template literals and uploads it to S3 as `certificates/${id}.html` (`Content-Type: text/html`). No binary PDF compilation (via Puppeteer or WeasyPrint) exists.
5. **Missing Database Table for Issuance Rules:** The API exposes endpoints `GET /issuance-rules`, `POST /issuance-rules`, and `DELETE /issuance-rules/:id`. However, table `issuance_rules` **does not exist in `prisma/credentials/schema.prisma`**. `CertificatesRepository` uses `(this.prisma.credentials as any).issuanceRule`, returning empty arrays or stub IDs.
6. **QR Code Security:** Each issued certificate receives a cryptographically random 16-byte hex token (`crypto.randomBytes(16).toString('hex')`) enforced by a database unique constraint `@@unique([qrCode])`.
7. **Complete Revocation Lifecycle:** Staff can revoke certificates via `POST /certificates/:id/revoke`, recording `revocationReason` and `revokedAt`, which immediately flips public verification to invalid.

---

## 2. Directory Layout & Document Navigation

```
docs/functional-spec/module-12-certificates/
├── README.md               # Executive summary, findings, and navigational guide (this file)
└── SPECIFICATION.md        # Comprehensive 17-Section (A–Q) legacy specification & traceability matrix
```

- **Detailed Functional Specification:** [`SPECIFICATION.md`](./SPECIFICATION.md)
- **Authoring Standard:** [`FUNCTIONAL-SPECIFICATION-STANDARD.md`](../../FUNCTIONAL-SPECIFICATION-STANDARD.md)
- **Master Index & Registry:** [`INDEX.md`](../INDEX.md)

---

## 3. Subsystem Architecture & Issuance Flow

```
   Milestone Events:
   - PathCompleted
   - EventCompleted
   - RoomCompleted
   - Manual Staff Call (POST /certificates)
         │
         ▼
   ┌────────────────────────────────────────────────────────┐
   │                  CredentialsService                    │
   │  1. Validate template (prisma.certificateTemplate)     │
   │  2. Generate 16-byte random hex qrCode                 │
   │  3. Insert into credentials.certificates               │
   │  4. Emit outbox event: CertificateIssued               │
   │  5. Enqueue BullMQ job: queue 'certificate'            │
   └───────────────────────────┬────────────────────────────┘
                               │
            ┌──────────────────┴──────────────────┐
            ▼                                     ▼
   ┌──────────────────────────────┐     ┌──────────────────────────────┐
   │ RealtimeCertificatesConsumer │     │ CertificateGeneratorConsumer │
   │ (src/websocket/consumers)    │     │ (src/worker/consumers)       │
   │                              │     │                              │
   │ Emits WS 'certificate:issued'│     │ 1. Renders HTML template     │
   │ to user:${userId}            │     │ 2. Uploads HTML to S3 bucket │
   └──────────────────────────────┘     │ 3. Updates cert.pdfUrl       │
                                        └──────────────────────────────┘
```

---

## 4. Key Forensic Discoveries Table

| Architectural Area | Conceptual Platform Expectation | Legacy Reality in Source Code | Forensic Status |
|---|---|---|:---:|
| **Public Verification** | Public link / QR code verification | Supported via `/certificates/verify/:qrCode` & `/verify/c/:id`. | `CONFIRMED LEGACY IMPLEMENTATION` |
| **PDF Generation** | Headless Chrome compiles binary PDF | Renders HTML string, stores as `.html` in S3; no PDF engine. | `PARTIALLY IMPLEMENTED (HTML ONLY)` |
| **Worker Queue Payload**| Producer passes required render data | Producer omits `recipientName`, `title`; worker crashes on null. | `DEFECT / WORKER FAILURE` |
| **Issuance Rules DB** | Configurable automated trigger rules | Model `IssuanceRule` missing from Prisma schema; returns stubs. | `DEFECT / SCHEMA MISMATCH` |
| **Template Storage** | Granular HTML/CSS/Orientation columns | Single JSON column `design Json @default("{}")` in database. | `CONFIRMED (JSON STORE)` |
| **Student Privacy** | Toggle public/private visibility | Supported via `PATCH /me/certificates/:id` (`isPublic` boolean).| `CONFIRMED LEGACY IMPLEMENTATION` |
| **Direct Download** | 302 redirect to signed CloudFront URL | Supported via `GET /me/certificates/:id/download`. | `CONFIRMED LEGACY IMPLEMENTATION` |
| **Admin List All API** | `GET /certificates` lists tenant certs | Endpoint does NOT exist in backend controller (frontend 404). | `CONFLICTING / MISSING ENDPOINT` |

---

## 5. Source File Traceability Index

| Subsystem / Layer | Source File Path | Primary Functionality |
|---|---|---|
| **Database Schema** | `prisma/credentials/schema.prisma` | Defines `CertificateTemplate` and `Certificate` models |
| **Controller** | `src/credentials/controllers/credentials.controller.ts` | 17 REST endpoints for templates, issuance, verification, rules |
| **Service** | `src/credentials/services/credentials.service.ts` | Issuance logic, QR generation, verification, and revocation |
| **Repository** | `src/credentials/repositories/certificates.repository.ts` | Prisma queries for templates, certs, and stubbed issuance rules |
| **Event Consumer** | `src/credentials/consumers/credentials.consumers.ts` | Listens to `PathCompleted`, `EventCompleted`, `RoomCompleted` |
| **Worker Consumer** | `src/worker/consumers/certificate-generator.consumer.ts` | BullMQ worker uploading certificate HTML to S3 |
| **WebSocket Consumer** | `src/websocket/consumers/realtime-certificates.consumer.ts`| Real-time WebSocket notification to `user:${userId}` |
| **Frontend Service** | `frontend/src/services/api/certificate.service.ts` | API client methods for certificates and public verification |

