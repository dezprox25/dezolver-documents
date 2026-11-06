# Module 3 — Adopted Subscription & Seat Management Decisions

**Document ID:** DEZ-SUB-DEC-003  
**Decision date:** 2026-11-06  
**Authority:** Product owner selected fixed College packages with custom quotes and stated that Super Admin must be able to edit/configure the commercial settings. Remaining minimal decisions use recommended v1 SaaS defaults.  
**Status:** Product baseline adopted; schema, RBAC and cross-module implementation pending.

## Decision precedence

This record is authoritative for SUB-D01–SUB-D40 and supersedes conflicting TBD/Decision Required wording in the earlier Module 3 draft and QA companion. Main source documents remain evidence of what they specify; legacy Module 03 remains historical functional evidence only. Adoption does not claim schema, UI, API, gateway or automated billing functionality has been implemented.

## Adopted decisions

| ID | Area | Adopted target decision |
|---|---|---|
| D01 | Commercial model | Fixed-price College package for a configured duration, including a configured student-seat allowance. No per-active-student subscription billing. Additional seats are separately priced add-ons. |
| D02 | SUPER_ADMIN catalogue control | SUPER_ADMIN can create, view, edit, version, publish, activate and deactivate Plans. Editable fields include name, description, code, package price, currency, duration, included seats, features, add-on pricing/pack, default designation and effective dates. Every change is validated and audited. |
| D03 | DEFAULT/CUSTOM | DEFAULT is a published standard Plan. CUSTOM is a College-specific offer documented through PricingQuote and approved by SUPER_ADMIN; it is not a client-side price override. |
| D04 | Quote authority | FINANCE_TEAM can request a custom quote for its assigned College. SUPER_ADMIN creates, edits, approves, rejects, withdraws or reissues it. Accepted quote terms are immutable. |
| D05 | Subscription cardinality | A College may retain multiple historical Subscription records but only one current effective primary subscription at a time. The College-to-Subscription one-to-many relationship is retained. |
| D06 | Quote requirement/lifecycle | DEFAULT plan purchases do not require a quote. CUSTOM terms and non-standard discounts require an approved quote. Quote states: DRAFT, SENT, ACCEPTED, REJECTED, EXPIRED, WITHDRAWN. Acceptance is single-use and requires validUntil not passed. |
| D07 | Entitlement schema | Plan.features is a validated structured entitlement configuration. SUPER_ADMIN edits supported feature flags/limits; custom quotes can override only supported values with approval. |
| D08 | Pricing calculation | Subtotal is fixed package plus add-ons; discount is deducted; configured tax applies to the discounted taxable amount; total = subtotal - discount + tax. Discount cannot exceed subtotal. Use decimal/minor-unit-safe arithmetic. |
| D09 | Currency | INR is the initial configured default. SUPER_ADMIN may enable other currencies. Each Plan/Quote/Subscription/Invoice/Payment uses one currency; no implicit FX conversion. |
| D10 | Period semantics | Plan.durationMonths sets term. Store UTC timestamps; display in configured billing timezone (initially Asia/Kolkata). Period is half-open [start,end); renewalDate is the next boundary. Use calendar-month arithmetic, not a fixed 30-day month. |
| D11 | Initial activation | Module 2 creates the College and Finance Team assignment. Finance selects DEFAULT or requests CUSTOM. Subscription begins PENDING and becomes ACTIVE only after trusted payment confirmation or SUPER_ADMIN-confirmed bank transfer. |
| D12 | Renewal | Manual renewal is the v1 default. Reminders are configurable. Auto-renewal is disabled by default and may be enabled by SUPER_ADMIN only with explicit College consent and a supported mandate. |
| D13 | Grace period | Unpaid renewal enters GRACE_PERIOD for a configurable duration (initial configured default: 7 calendar days; SUPER_ADMIN can change globally or per College). Existing contracted features remain, but new seat purchases/feature expansion are blocked. Grace expiry moves to SUSPENDED. |
| D14 | Suspension/cancellation | Non-payment suspension follows grace expiry. SUPER_ADMIN can administratively suspend with reason. FINANCE_TEAM cancellation requests are period-end by default; immediate cancellation is SUPER_ADMIN-only with reason. Refunds are separate actions. |
| D15 | College status | College EntityStatus and SubscriptionStatus are independent. Module 2 enforces College status; Module 3 enforces subscription entitlements. Changing one does not silently rewrite the other. |
| D16 | Plan versioning | Plan edits create a new version for future purchases. Existing subscriptions retain accepted price, seat and feature snapshot until renewal. SUPER_ADMIN may explicitly migrate an existing subscription after impact preview, confirmation, effective date, reason and audit. |
| D17 | Invoice rules | Generate invoices for initial subscription, renewal, custom purchase and seat add-on. Issued unpaid invoice is OPEN; verified settlement makes PAID; corrections use VOID/replacement; UNC​​OLLECTIBLE is a SUPER_ADMIN write-off. Overdue is derived from OPEN plus dueAt. |
| D18 | Invoice access | SUPER_ADMIN has global access; FINANCE_TEAM can view/download only its College invoices. HOD sees Department seat summary only, no amount/invoice/payment detail. MANAGER/STUDENT have no Module 3 invoice access in v1. |
| D19 | Seat trigger | A seat is consumed when a Student membership becomes ACTIVE for a College. Pending invites and staff do not count. Count a User once per College. Deactivation/archive releases once; same-College transfer moves Department usage without changing College usage. |
| D20 | Canonical capacity | Plan.maxStudents is included package capacity. Subscription.purchasedSeats is current contracted College capacity including paid add-ons. College.totalSeats mirrors purchasedSeats; College.usedSeats counts active Student seat assignments. |
| D21 | Department allocation | FINANCE_TEAM allocates College capacity to Departments. Sum of allocatedSeats cannot exceed College.totalSeats; Department.usedSeats cannot exceed allocation. Unallocated capacity stays at College level. HOD has read-only view. |
| D22 | Buy More Seats | FINANCE_TEAM purchases an additional quantity for its assigned College. SUPER_ADMIN configures add-on rate/pack and proration. Default is prorated by remaining calendar days; add-on ends with the subscription period. Capacity increases only after verified payment. |
| D23 | Seat reduction | FINANCE_TEAM cannot reduce capacity during an active term. Reduction may be requested for next renewal only if usage fits. SUPER_ADMIN may approve an exceptional change with reason/impact review; never silently remove students. |
| D24 | Seat enforcement | Student activation is blocked when College capacity or Department allocation is exhausted. SUPER_ADMIN may grant an audited reasoned override. Existing students are not auto-deactivated. |
| D25 | Access gating | ACTIVE in-period subscription plus ACTIVE College grants contracted features. GRACE_PERIOD preserves existing features but blocks expansion. SUSPENDED/CANCELLED block paid academic features but retain limited billing/support access. |
| D26 | Subscription permissions | SUPER_ADMIN manages global Plans and all subscriptions. FINANCE_TEAM can view its College subscription, initiate purchase/renewal/add-on and update approved billing contact; it cannot edit global Plans or directly force payment-derived status. |
| D27 | Quote/invoice permissions | SUPER_ADMIN controls quote approvals, invoice correction/void and write-off. FINANCE_TEAM requests custom quotes and views/downloads its own College invoices. System invoices are not edited in place. |
| D28 | Payment permissions | FINANCE_TEAM initiates enabled online payment or submits bank-transfer evidence. SUPER_ADMIN verifies manual transfers and authorizes full/partial refunds with reason. Browser redirect cannot mark payment successful. |
| D29 | Gateway config | SUPER_ADMIN enables/configures gateways. Razorpay is the initial India-first online gateway; Manual Bank Transfer may be enabled with verification. Stripe remains disabled until configured/tested. |
| D30 | Webhook/manual confirmation | Signed gateway webhook is authoritative only when signature verification succeeds with a configured secret; missing/invalid secret fails closed. Manual transfer remains PENDING until SUPER_ADMIN verifies it. |
| D31 | Partial payment/retry | V1 supports full invoice settlement only; partial invoice payments are not supported. Failed/time-out attempts remain traceable and can be retried idempotently. |
| D32 | Refunds | SUPER_ADMIN authorizes full/partial refunds with reason, amount and provider result. Store separate immutable refund records. Payment becomes REFUNDED only when fully refunded; partial refund is explicitly represented/derived. Refund does not auto-reactivate or cancel subscription. |
| D33 | Financial references | Transactions link to same-College, same-currency records according to type: initial purchase to PENDING subscription/quote/invoice, renewal to subscription/invoice, add-on to subscription/invoice/quote as applicable. |
| D34 | Idempotency/reconciliation | Unique idempotency keys for logical payments and unique gateway event IDs. Reconciliation is retryable/auditable and never fabricates success or duplicates financial effects. |
| D35 | Invoice numbering/due date | Invoice number is unique, sequential within configured fiscal-year/legal-entity series, atomic and never reused. SUPER_ADMIN configures prefix/series and due terms. dueAt is explicit; overdue is derived. |
| D36 | PDF/access security | Invoice stores immutable line/tax/billing snapshots. PDF generation is asynchronous/retryable. Download requires role/scope validation and short-lived signed URL; raw storage URL is not public authorization. |
| D37 | Audit/notifications | Audit Plan, quote, subscription, invoice, payment, refund, seat purchase/allocation/override and access-impact actions. Module 15 owns audit structure/retention; Module 13 owns reminders/delivery. Never log secrets/card data. |
| D38 | Retention/correction | Retain settled financial records, accepted quotes, invoice snapshots, payments, refunds, subscription history and seat events per approved financial retention. Never hard-delete/edit a settled record; use void/replacement/adjustment. |
| D39 | Error behavior | Foreign College billing resource returns 404; in-scope unauthorized action returns 403. FINANCE_TEAM retains limited settlement/support access during subscription restriction, but paid academic access remains blocked. |
| D40 | Concurrency | Use Plan version checks; transactions/optimistic locking for renewals, purchases and allocations; idempotency prevents duplicate operations and oversell. Record effective time, actor and reason as applicable. |

## Super Admin commercial configuration

SUPER_ADMIN owns the editable commercial control plane:

- Plan catalogue: names, descriptions, stable codes, visibility, active state and default designation.
- Commercial terms: fixed package price, currency, duration, included seats, add-on seat price/pack and proration.
- Product entitlements: supported feature flags and limits in a validated structured configuration.
- Custom offers: College-specific price, seat quantity, duration, feature bundle, discount and expiry through an approved quote.
- Billing policy: enabled gateways, supported currencies, tax rules, invoice series, due terms, renewal reminders, grace duration and billing timezone.
- Existing subscriptions: explicit migration/override workflow with impact preview, effective date, reason, confirmation, audit and preservation of prior terms.

**Default edit behavior:** a catalogue edit creates a new Plan version for future purchases. It does not silently change existing contracted amounts, features or seat capacity. SUPER_ADMIN can explicitly apply a new version through a separate controlled action.

## Required schema and data-contract changes

1. **Plan/versioning:** add stable unique code, version/effective-date metadata, included-seat/add-on pricing fields or PlanVersion entity, supported feature schema and immutable price/feature snapshots.
2. **PricingQuote:** support both pre-onboarding collegeOnboardingId and existing-College collegeId; add quote number/version, status enum, approval/acceptance/rejection metadata and immutable accepted terms.
3. **Subscription:** add PENDING to SubscriptionStatus; persist contracted price/currency/duration/seat/feature snapshot, selected plan version, renewal/cancellation effective dates, cancellation reason and concurrency version.
4. **Billing settings:** platform/College-scoped settings for gateways, currencies, tax, invoice series/due terms, reminder schedule, grace period, timezone and add-on pricing/proration.
5. **Invoice:** add issuedAt, dueAt, line-item snapshot, tax breakdown, billing party/address/tax identifiers, series reference and correction/void linkage; add DRAFT if invoice draft stage is used.
6. **PaymentTransaction:** add payment method/provider metadata, confirmed timestamp and safe provider references; validate type-specific reference combinations.
7. **WebhookEvent:** durable provider event ID, verification/processing state and received/processed timestamps.
8. **Refund:** immutable PaymentRefund records for full/partial refunds, amount, reason, actor, provider refund ID, status and timestamps; add PARTIALLY_REFUNDED or derive from refund totals.
9. **Seat ledger:** unique active College Student seat assignment keyed to User + College, linked to Module 1 membership and Department, with immutable events/idempotency keys.
10. **Department allocation:** enforce sum(Department.allocatedSeats) <= College.totalSeats and Department.usedSeats <= Department.allocatedSeats transactionally.
11. **Integrity/indexes:** foreign keys, same-College checks, invoice-number uniqueness per series/fiscal year, unique payment idempotency key, unique gateway event ID and indexes for College/Subscription/status/period lookups.
12. Do not hard-delete settled invoices, payments, refunds, accepted quotes, subscription history or seat-ledger events.

## Canonical role/permission alignment required

- **SUPER_ADMIN:** global Plan/quote/subscription/invoice/payment/refund/seat administration and billing configuration.
- **FINANCE_TEAM:** assigned-College billing, plan selection, custom quote request, renewal, additional-seat purchase, invoice/payment history and approved billing-contact updates; no global Plan edits or payment-derived state mutation.
- **HOD:** assigned-Department seat allocation/usage summary only; no plan price, quotes, invoices, payment details or billing mutations.
- **MANAGER:** no Module 3 billing/seat administration.
- **STUDENT:** no Module 3 billing page or College billing-record access in v1. The RBAC "View own" billing cell requires canonical RBAC cleanup because no Student billing page is mapped.

These alignment changes must be applied to the canonical RBAC document; this commit does not silently edit source-text/RBAC.md.

## Cross-module contracts

| Module | Required alignment |
|---|---|
| 1 — Authentication & User Lifecycle | Finance Team membership/scope; Student activation/deactivation/transfer events; idempotent identity events for seat assignment. |
| 2 — College / Department / Section Management | College/Department hierarchy and College status stay Module 2-owned; capacity and counters are Module 3-owned; Module 2 displays them read-only. |
| 4 — Course & Enrollment | Enrollment is not the seat trigger; it validates that Student has an active seat assignment and eligible subscription. |
| 13 — Notifications | Quote, invoice, payment, renewal reminder, grace, suspension, refund and capacity notifications. |
| 15 — Audit Logging | Plan, quote, subscription, payment, invoice, refund, seat allocation and override events. |
| 17 — Functional State Machines | Plan version, quote, subscription, invoice, payment and refund transitions. |
| 18 — Edge Cases & Failure Handling | Gateway failures, duplicate/out-of-order webhooks, reconciliation, proration, capacity races and retry/idempotency contracts. |

## Implementation gate

Product decisions SUB-D01–SUB-D40 are adopted. Engineering still must implement schema changes, permissions, Plan versioning, billing settings, payment gateway configuration, invoice/PDF pipeline, seat ledger/enforcement, state machines, audit/notification contracts and cross-module integrations. QA cases waiting on these prerequisites are implementation-blocked, not product-decision-blocked.
