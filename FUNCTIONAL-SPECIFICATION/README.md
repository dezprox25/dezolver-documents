# DEZOLVER — Functional Specification

Module-by-module functional specifications, authored sequentially first; cross-module review and decision resolution will follow after all modules are completed.

## Authoring standard

All module upgrades must follow the [Functional Specification Standard](./FUNCTIONAL-SPECIFICATION-STANDARD.md). The standard defines the required module structure, function-level detail, evidence labels, acceptance criteria, QA coverage, decision log, and review gate.

## Upgrade and review plan

1. Establish and validate the common specification standard.
2. Upgrade Modules 1–9 from source-based drafts into detailed function-level specifications.
3. Keep source facts, legacy findings, approved target requirements, recommendations, and unresolved decisions separate.
4. Complete all remaining module specifications first. Then perform the cross-module review and resolve product decisions before treating target behavior as approved.
5. Module 10 (Groups) has been expanded with function-level specifications and QA coverage by explicit user direction. This is a documented sequence exception; unresolved Group decisions remain open.

**Important status distinction:** committed to GitHub / approved for repository inclusion does not mean the functional behavior is product-approved or implementation-ready.

## Module index

| # | Module | Current repository status |
|---|---|---|
| 1 | [Authentication & User Lifecycle](./1.%20Authentication%20%26%20User%20Lifecycle/README.md) | Detailed draft — Standard v1.0 aligned; QA appendix added; product review pending; decisions open |
| 2 | [College / Department / Section Management](./2.%20College%20-%20Department%20-%20Section%20Management/README.md) | Detailed draft — Standard v1.0 aligned; QA appendix added; product review pending; decisions open |
| 3 | [Subscription & Seat Management](./3.%20Subscription%20%26%20Seat%20Management/README.md) | Detailed draft — Standard v1.0 aligned; QA appendix added; product review pending; decisions open |
| 4 | [Course & Enrollment](./4.%20Course%20%26%20Enrollment/README.md) | Detailed draft — Standard v1.0 aligned; function-level spec + QA companion; product review pending; decisions open |
| 5 | [Problem Management](./5.%20Problem%20Management/README.md) | Detailed draft — Standard v1.0 aligned; 25 function specs + QA companion (170 cases); product review pending; decisions open |
| 6 | [Code Submission & Judge0](./6.%20Code%20Submission%20%26%20Judge0/README.md) | Detailed draft — Standard v1.0 aligned; QA companion (120 cases); product/architecture/security review pending; decisions open |
| 7 | [Test Case Evaluation](./7.%20Test%20Case%20Evaluation/README.md) | Detailed draft — Standard v1.0 aligned; QA companion (170 cases); product/architecture/security review pending; decisions open |
| 8 | [Labs & Lab Scoring](./8.%20Labs%20%26%20Lab%20Scoring/README.md) | Detailed draft — Standard v1.0 aligned; 20 function specs + QA companion (158 cases); product/architecture/security review pending; decisions open |
| 9 | [Assessments](./9.%20Assessments/README.md) | Detailed draft — Standard v1.0 aligned; 22 function specs + QA companion (158 cases); product/architecture/security review pending; decisions open |
| 10 | [Groups](./10.%20Groups/README.md) | Detailed draft — Standard v1.0 aligned; 11 function specs + QA companion (132 cases); product/security review pending; decisions open |
| 11 | [Contests](./11.%20Contests/README.md) | Detailed draft — core function specs + six submodules + QA companion (126 cases); product/architecture/security decisions open |
| 11.1 | [Contest Participation](./11.%20Contests/11.1%20Contest%20Participation/README.md) | Detailed draft; eligibility/registration decisions open |
| 11.2 | [Contest Submission](./11.%20Contests/11.2%20Contest%20Submission/README.md) | Detailed draft; contest-submission linkage decision open |
| 11.3 | [Contest Scoring](./11.%20Contests/11.3%20Contest%20Scoring/README.md) | Detailed draft; scoring formula decisions open |
| 11.4 | [Contest Penalty](./11.%20Contests/11.4%20Contest%20Penalty/README.md) | Detailed draft; penalty formula/unit decisions open |
| 11.5 | [Contest Ranking](./11.%20Contests/11.5%20Contest%20Ranking/README.md) | Detailed draft; rank ordering and visibility decisions open |
| 11.6 | [Contest Tie Breaking](./11.%20Contests/11.6%20Contest%20Tie%20Breaking/README.md) | Detailed draft; tie-break policy not source-defined |
| 12 | [Certificates](./12.%20Certificates/README.md) | Detailed draft — 9 function specs + QA companion (68 cases); eligibility, issuance, PDF/QR, verification decisions open |
| 13 | [Notifications](./13.%20Notifications/README.md) | Detailed draft — 9 function specs + QA companion (65 cases); trigger, sender/recipient, read-state, delivery, and channel decisions open |
| 14 | [Support Tickets](./14.%20Support%20Tickets/README.md) | Detailed draft — 13 function specs + QA companion (92 cases); lifecycle, assignment, comment visibility, escalation and verification decisions open |
| 15 | [Audit Logging](./15.%20Audit%20Logging/README.md) | Detailed draft — 13 function specs + QA companion (98 cases); audit coverage, failure capture, privacy/redaction, retention, export, and scope decisions open |
| 16 | [Student Analytics](./16.%20Student%20Analytics/README.md) | Detailed draft — 14 function specs + QA companion (104 cases); metrics, progress/certificate semantics, rating, privacy, and analytics architecture decisions open |
| 17 | [Functional State Machines](./17.%20Functional%20State%20Machines/README.md) | Detailed draft — 12 state-machine function specs + QA companion (96 cases); transition guards, lifecycle ownership, reversal, concurrency, and side-effect decisions open |
| 18 | [Edge Cases & Failure Handling](./18.%20Edge%20Cases%20%26%20Failure%20Handling/README.md) | Detailed draft — 12 function specs + QA companion (96 cases); error contract, retry/idempotency, Redis/DB failure, rate limits, DLQ, circuit breakers, and operations decisions open |

**Current sequence:** Initial detailed drafts and QA companions are now present for Modules 1–18. Next: cross-module consistency review, decision-log reconciliation, traceability validation, and product/security/architecture/QA review. Committing a draft does not approve its recommendations.
