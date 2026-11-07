# DEZOLVER — 11.1 Contest Participation

**Document ID:** CON-PART-FS-001 | **Version:** 0.2 | **Status:** Detailed draft; decisions open  
**Governing standard:** [Functional Specification Standard v1.0](../../FUNCTIONAL-SPECIFICATION-STANDARD.md)  
**Parent module:** [Module 11 Contests](../README.md)  
**Core contest management:** [Function-Level Specifications](../FUNCTION-SPECIFICATIONS.md)

> Source facts, derived behavior, recommendations, and TBD decisions remain separate. This draft does not approve proposed behavior.

## Evidence
ContestParticipant(id, contestId, studentId, joinedAt, createdAt, updatedAt) is source-confirmed in schema. Page mappings show Student/HOD/Manager Contests pages. RBAC Student View/Participate. Registration/eligibility mechanics are not specified by Business Flow.

## Permissions and scope
Student identity must come from authenticated session. Staff roster visibility is within authorized College/Department/Section/Group scope. Student may view own participation; public roster visibility is TBD. GroupContest does not prove team mode or automatically enroll Group members.

### CPART-F001 — Discover contests
- **Purpose:** Show Student permitted Contests.
- **Entry:** Student Contests page.
- **Preconditions:** authenticated active Student.
- **Inputs:** optional search/status filter; student ID auth-derived.
- **Flow:** request list → server applies role/scope and approved visibility → return allowed Contest fields → render.
- **Rules:** discovery ≠ eligibility to join/submit; status/isActive and pre-start visibility TBD.
- **Validation:** foreign tenant/forged ID must not broaden access.
- **Data:** read Contest and permitted participation state.
- **Success:** authorized list; no private score/code.
- **Failures/edge:** no results, inactive Contest, stale schedule, account status change.
- **Acceptance:** only permitted contests; server identity; no unauthorized disclosure.
- **QA:** CON-QA-039–043.

### CPART-F002 — Validate eligibility
- **Purpose:** Validate Student participation before join/start.
- **Inputs:** contestId; Student from auth.
- **Flow:** validate Contest existence/scope/state/time → Student account → approved academic eligibility → existing participant → return allow/deny.
- **Decision dependencies:** College/Department/Section/Group, enrollment, batch, subscription, capacity, registration dates are CON-D08/10. Do not assume each applies.
- **Validation:** account inactive follows Module 1; time boundary CON-D05; Group membership CON-D09/21.
- **Data:** read only.
- **Acceptance:** consistent server-side eligibility; safe denial without foreign-resource leakage.
- **QA:** CON-QA-044–048.

### CPART-F003 — Join/register
- **Status:** Participant entity exists; student registration operation is not explicit in source.
- **Entry:** Contest detail → Join/Register (proposed).
- **Inputs:** contestId; authenticated student identity.
- **Flow:** request join → revalidate eligibility/state → check duplicate/capacity → create ContestParticipant and joinedAt → return participation.
- **Rules:** self-registration vs staff enrollment, capacity, cutoff, unregister, duplicate semantics CON-D07/10/23. Recommend duplicate prevention. Do not auto-register Group members.
- **Data:** create Participant only; no score/submission/certificate.
- **Failure:** no partial row; retry idempotency Module 18.
- **Audit:** participant joined event recommended; Modules 13/15 own notification/audit.
- **Acceptance:** one participant per approved identity; Group link does not automatically join.
- **QA:** CON-QA-049–054.

### CPART-F004 — View/manage roster
- **Entry:** Contest → Participants / My participation.
- **Inputs:** contestId, optional page/filter; identity auth-derived.
- **Flow:** authorize roster/own view → query participant records → return role-appropriate projection.
- **Rules:** Student full roster visibility and staff roster fields TBD; Ranking owns score/rank.
- **Data:** read only.
- **Failure/edge:** foreign Contest, former participant, removed Student, privacy.
- **Acceptance:** role/scope and own-data rules; no code/score leakage.
- **QA:** CON-QA-055–058.

### Historical cross-check — legacy Event evidence

**Evidence status: LEGACY-SPEC-REPORTED.** The old-code report describes a separate Registration entity with POST registration, DELETE self-unregister and organizer roster routes. Registration included optional teamName, registered/cancelled/pending_payment states, timestamps, reported unique (eventId,userId) and optional maxParticipants. Paid orders/refunds and a registration guard defect were also reported: service accepted only published while the watcher moved to registration_open. These are not target behavior. Target ContestParticipant has contestId/studentId/joinedAt/timestamps only; capacity, withdrawal, team, payment, duplicate and eligibility rules remain CON-D07–D10/D26. Do not copy the status lockout, payment path or teamName semantics. Raw source was not independently retrieved.

**Decision IDs:** CON-D07–D10, D23–D24.
