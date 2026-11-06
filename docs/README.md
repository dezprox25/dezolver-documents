# DEZOLVER Documentation

This folder contains developer-facing documentation extracted from the source PDFs maintained in the DEZOLVER Google Drive folder.

## Source document inventory

| Document | Pages | Size | Purpose |
|---|---:|---:|---|
| `scheema.pdf` | 3 | 2,948,330 bytes | Database schema / ER model covering the core DEZOLVER entities and relationships. |
| `RBAC (1).pdf` | 25 | 316,107 bytes | Role-based access-control model, permission matrices, data-scope rules, backend/API authorization, tenant isolation, ownership checks, and Judge0-related authorization flow. |
| `Page - DB Mapping.pdf` | 3 | 4,654,643 bytes | Original stakeholder-oriented mapping from Figma/application pages to supporting database entities. |
| `Page - DB Mapping v2.pdf` | 3 | 4,789,520 bytes | Updated page-to-database mapping for Super Admin, Finance Team, HOD, Manager, and Student screens. |
| `Business Flow.pdf` | 5 | 241,237 bytes | Original end-to-end business-flow diagrams for platform stakeholders and academic activities. |
| `Business Flow v2.pdf` | 5 | 464,575 bytes | Updated business-flow diagrams with explicit DEZOLVER backend and Judge0 evaluation stages and expanded lab/contest scoring flow. |

## System model captured by the PDFs

DEZOLVER is organized around five primary roles: `SUPER_ADMIN`, `FINANCE_TEAM`, `HOD`, `MANAGER`, and `STUDENT`.

The business flow starts with platform-level college onboarding and subscription setup, then moves into department/section academic administration, course/problem/lab/contest setup, student participation and code submission, code execution/evaluation through Judge0, and resulting scores/ranks.

The RBAC model applies authorization in layers: authentication → role/permission check → tenant/data-scope check → resource/ownership validation → database operation → audit logging where required. Data scope narrows from global to college, department, section/group, and finally ownership.

The schema connects platform administration and billing (`College`, `Department`, `Section`, `User`, `Plan`, `Subscription`, `Invoice`, `PaymentTransaction`) with academic delivery (`Course`, `Problem`, `TestCase`, `LabAssignment`, `Assessment`, `Contest`, `Group`) and student execution/results (`CodeSubmission`, `LabSubmission`, contest participation/ranking, certificates, tickets, notifications).

## Version notes

### Business Flow v2
Compared with the original Business Flow, v2 makes the code-execution path more explicit by inserting the DEZOLVER backend and Judge0 between code submission and evaluation. It also expands lab and contest paths to show Judge0 evaluation, lab scoring, contest scoring, and score/rank generation.

### Page - DB Mapping v2
Both mapping PDFs contain the same high-level stakeholder structure in extractable text; most of the detailed mappings are visual elements in the PDF. Keep both versions for design/history reference.

## Text extracts

Text equivalents of the PDFs are stored beside this index where extraction is meaningful. Visual-heavy page-mapping PDFs have limited extractable text, so the original PDF remains the authoritative representation for those diagrams.

## Original source

Google Drive folder: `dezolver`

Source folder ID: `1U0pxh_AKb59VZS_8-AGOKVCdrFaWC6bc`
