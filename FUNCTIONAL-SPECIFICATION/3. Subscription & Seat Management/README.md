# 3. Subscription & Seat Management

**Document type:** Functional Specification  
**Document ID:** DEZ-SUB-FS-003  
**Version:** 0.2 — Functional Specification Standard v1.0 compliance pass  
**Status:** Detailed draft — ready for product review; target behavior not approved  
**Scope:** Plan catalogue, college subscription, subscription periods and renewal, pricing quotes, invoices, payment transactions, and seat purchase / allocation / usage  
**QA companion:** [SUB QA Test Cases](./QA-TEST-CASES.md)

> **Specification principle:** This module is grounded in Business Flow v2, RBAC, Page-to-DB Mapping v2, and the schema PDF. The source documents identify entities, roles, screens, and high-level flows, but do not define all billing rules. Missing behavior is marked **TBD / Decision Required** rather than invented.

---

## 1. Purpose

Define how DEZOLVER offers plans, assigns a subscription to a College, manages the subscription period and renewal, records billing and payment information, and tracks seats purchased and used.

Subscription & Seat Management is closely coupled to College / Department / Section Management and user onboarding. It must preserve a consistent relationship between:

- Plan capacity and features
- Subscription purchased seats
- College seat totals and usage
- Department seat allocation and usage
- Pricing quotes, invoices, and payment transactions
- Subscription state and access to platform services

## 2. Scope

### Included

- Plan catalogue and default/custom plan distinction
- College subscription record
- Subscription start/end dates, active period, renewal date, and grace-period date
- Finance Team subscription and billing pages
- SUPER_ADMIN subscription, plan, invoice, and payment management
- Buy More Seats flow
- Seat totals, allocation, and usage fields
- Pricing quote structure
- Invoice structure and status
- Payment transaction structure, gateways, and status
- Role-based access and college-level data scope
- Billing and subscription validation rules
- Source discrepancies and decisions required

### Excluded / owned by other modules

| Topic | Detailed specification owner |
|---|---|
| College onboarding form, approval, and creation | Module 2 — College / Department / Section Management |
| User creation/invitation and account activation | Module 1 — Authentication & User Lifecycle |
| College/Department/Section CRUD | Module 2 — College / Department / Section Management |
| Course enrollment and student eligibility | Module 4 — Course & Enrollment |
| Notifications for invoices, renewals, payment failures, etc. | Module 13 — Notifications |
| Audit event catalogue and retention | Module 15 — Audit Logging |
| Subscription/College canonical state-transition diagrams | Module 17 — Functional State Machines |
| Payment and subscription edge-case policy | Module 18 — Edge Cases & Failure Handling |

---

## 3. Source documents and interpretation

### 3.1 Business Flow v2

The Business Flow v2 describes the following Finance Team flow:

- Finance Team views the plan.
- Subscription is associated with a Plan.
- Plan selection includes **DEFAULT** and **CUSTOM**.
- Subscription includes an **ACTIVE PERIOD**.
- Subscription includes **RENEWAL**.

The diagram does not provide a detailed billing/payment sequence or define automatic renewal, grace-period duration, or subscription expiry behavior.

### 3.2 RBAC

The RBAC source describes:

- SUPER_ADMIN: platform-wide subscription and billing administration.
- FINANCE_TEAM: college-level subscription and billing operations.
- HOD: View subscription/billing data as shown in the permission matrix.
- MANAGER: no subscription/billing permission shown.
- STUDENT: View own subscription/billing as shown in one RBAC matrix; the page mapping does not show a student billing page (discrepancy to resolve).

RBAC also requires tenant isolation: Finance Team queries must be scoped to the assigned College.

### 3.3 Page - DB Mapping v2

Mapped screens and entities include:

| Role | Page / module | Entities mapped |
|---|---|---|
| SUPER_ADMIN | Dashboard | College, User, Department, Plan, Subscription, Invoice, PaymentTransaction, AuditLog |
| SUPER_ADMIN | Plan Management | Plan, Subscription |
| SUPER_ADMIN | Subscription Management | College, Plan, Subscription, Invoice, PaymentTransaction |
| SUPER_ADMIN | Invoice Management | Invoice, Subscription, College, PaymentTransaction |
| SUPER_ADMIN | Payment Management | PaymentTransaction, Subscription, Invoice, College |
| FINANCE_TEAM | Dashboard | College, Plan, Subscription, Department, Invoice, User |
| FINANCE_TEAM | Billing & Subscription | College, Plan, Subscription, Invoice, PaymentTransaction |
| FINANCE_TEAM | Invoice History | Invoice, Subscription, PaymentTransaction, College |
| FINANCE_TEAM | Buy More Seats | Plan, Subscription, College |

These mappings confirm the intended page/entity relationships; they do not define detailed UI controls, payment integration, or calculations.

### 3.4 Schema PDF

The schema includes Plan, CollegeOnboarding, PricingQuote, Subscription, Invoice, PaymentTransaction, College, and Department records. Their fields are documented in Section 8.

---

## 4. Roles and permissions

### 4.1 Source-based permissions

| Billing / subscription operation | SUPER_ADMIN | FINANCE_TEAM | HOD | MANAGER | STUDENT |
|---|---|---|---|---|---|
| Subscription & Billing | Full CRUD | View / Create / Update | View | Not listed | View own (RBAC matrix) |
| Plan Management | Plan Management page; permissions inferred as platform admin | View Plan in Business Flow | Not listed | Not listed | Not listed |
| Invoice Management | Page explicitly mapped | Invoice History page mapped | Not explicitly listed | Not listed | Not explicitly listed |
| Payment Management | Page explicitly mapped | Billing & Subscription page mapped | Not explicitly listed | Not listed | Not explicitly listed |
| Buy More Seats | Platform-level management (derived) | Page explicitly mapped | Not listed | Not listed | Not listed |

**Source limitation:** the RBAC matrix does not define every action for Invoice Management, Payment Management, or Buy More Seats separately. Confirm detailed action permissions before implementation.

### 4.2 Scope enforcement

- **SUPER_ADMIN:** platform-wide subscription and billing administration.
- **FINANCE_TEAM:** assigned College only.
- **HOD:** assigned College + Department; RBAC shows subscription/billing View.
- **MANAGER:** no billing permissions shown.
- **STUDENT:** RBAC mentions viewing own billing/subscription records, but the Page-to-DB Mapping does not show a Student billing screen.

Every backend operation must validate role, permission, and College scope. A Finance Team user must not view or modify another College's subscription, invoices, payment transactions, or seat information.

---

## 5. Functional requirements

### SUB-001 — Maintain a Plan catalogue

**Status:** Source-confirmed schema and page mapping.

DEZOLVER must represent plans using the Plan entity.

Schema fields:

- `id`
- `name`
- `description`
- `price`
- `currency`
- `durationMonths`
- `features` (JSON)
- `maxStudents`
- `isActive`
- `createdAt`
- `updatedAt`

SUPER_ADMIN has a Plan Management page mapped to Plan and Subscription.

**TBD:** whether plan price is per month, per term, per seat, or per plan; exact features; plan visibility; versioning; and whether existing subscriptions retain the plan terms that were active at purchase.

### SUB-002 — Support DEFAULT and CUSTOM plan selection

**Status:** Source-confirmed in Business Flow v2.

The Business Flow shows Plan types/choices labeled **DEFAULT** and **CUSTOM**.

**TBD:** whether CUSTOM means a custom-priced plan, custom feature bundle, seat count adjustment, negotiated quote, or a separate plan record; who may create/approve custom plans; and whether a custom plan is represented by Plan or PricingQuote.

### SUB-003 — Associate a College with a Subscription

**Status:** Source-confirmed schema.

Subscription contains:

- `collegeId`
- `planId`
- `status`
- `purchasedSeats`
- `currentPeriodStart`
- `currentPeriodEnd`
- `renewalDate`
- `gracePeriodEndsAt`

A Subscription must reference a College and Plan.

**TBD:** whether a College can have multiple simultaneous subscriptions, multiple subscriptions over time, add-on subscriptions, or more than one active subscription.

### SUB-004 — Display subscription plan and period

**Status:** Source-confirmed business flow and schema.

Finance Team must be able to view the plan and subscription details for its assigned College, including the subscription period and renewal information represented in the schema.

The schema fields `currentPeriodStart`, `currentPeriodEnd`, and `renewalDate` provide the date data.

**TBD:** date/timezone display convention, whether period dates are inclusive/exclusive, and whether renewalDate equals currentPeriodEnd or is distinct.

### SUB-005 — Subscription lifecycle status

**Status:** Source-confirmed schema values; operational behavior TBD.

The schema's `SubscriptionStatus` enum contains:

- `ACTIVE`
- `GRACE_PERIOD`
- `SUSPENDED`
- `CANCELLED`

The subscription also has `gracePeriodEndsAt`.

**TBD:** exact status transition conditions, who can change status, grace-period duration, and the capabilities available in each status. Do not infer system access restrictions solely from the enum names.

### SUB-006 — Renewal

**Status:** Source-confirmed as a Business Flow step; detailed workflow TBD.

The Business Flow explicitly includes Renewal.

The system must represent renewal-related information using Subscription dates/status and relevant Invoice/PaymentTransaction records.

**TBD:**
- Manual renewal vs automatic renewal
- Renewal reminder schedule
- Renewal price calculation
- Renewal invoice generation
- Payment required before renewal or post-payment confirmation
- Renewal success/failure behavior
- Whether renewal changes the Plan or purchased seat quantity
- Whether a renewal extends dates from currentPeriodEnd or from payment date

### SUB-007 — Grace period

**Status:** Source-confirmed schema field/status; policy TBD.

The schema includes `GRACE_PERIOD` and `gracePeriodEndsAt`.

**TBD:** duration, when it starts, whether it is entered automatically or manually, whether code execution/course access remains enabled during grace period, and what happens when gracePeriodEndsAt passes.

### SUB-008 — Subscription suspension and cancellation

**Status:** Source-confirmed status values; behavior TBD.

Subscription supports `SUSPENDED` and `CANCELLED`.

**TBD:** who can suspend/cancel; whether cancellation is immediate or end-of-period; refund/credit policy; whether suspension can be reversed; what user-facing access is disabled; and how historical subscription/invoice data is retained.

### SUB-009 — Create and maintain a PricingQuote

**Status:** Source-confirmed schema.

PricingQuote fields:

- `id`
- `collegeOnboardingId`
- `planId`
- `studentCount`
- `baseAmount`
- `discountAmount`
- `taxAmount`
- `totalAmount`
- `currency`
- `status`
- `validUntil`
- `createdById`
- `createdAt`
- `updatedAt`

The schema links PricingQuote to CollegeOnboarding and Plan, and identifies the user who created the quote.

**TBD:** quote creation permissions, quote approval, quote status values, quote expiry behavior, whether quotes can be revised, and whether a quote is required for all subscriptions or only initial onboarding/custom plans.

### SUB-010 — Pricing calculation

**Status:** Source-confirmed fields; calculation rules TBD.

PricingQuote represents:

`baseAmount - discountAmount + taxAmount = totalAmount`

This arithmetic is a natural interpretation of the field names, **not a documented pricing formula**.

**TBD:** discount eligibility and calculation, tax rates and jurisdiction, rounding precision, currency conversion, per-seat vs fixed pricing, and whether tax is calculated before or after discount. The final formula must be explicitly approved.

### SUB-011 — Buy More Seats

**Status:** Source-confirmed Finance Team page and schema references.

The Page-to-DB Mapping includes a Finance Team **Buy More Seats** page associated with Plan, Subscription, and College.

The Subscription schema has `purchasedSeats`; College has `totalSeats` and `usedSeats`; Department has `allocatedSeats` and `usedSeats`.

**TBD:** exact flow:
- Does Finance Team choose an additional seat quantity or a target total?
- Is an additional-seat purchase prorated?
- Does it create a new PricingQuote, Invoice, and PaymentTransaction?
- Does purchasedSeats update immediately after payment or after manual confirmation?
- Does the subscription period change?
- Can seats be reduced?
- Can a College exceed its current Plan's maxStudents through a custom plan or seat add-on?

### SUB-012 — College seat totals and usage

**Status:** Source-confirmed schema.

College contains:

- `totalSeats`
- `usedSeats`

Subscription contains `purchasedSeats`.

**Derived consistency requirement:** the system must keep College seat capacity and Subscription purchased seat quantity consistent, unless the approved business rules explicitly define a different relationship.

**TBD:** whether `College.totalSeats` is a denormalized copy of Subscription.purchasedSeats, an independently managed capacity, or a calculated total across subscriptions.

### SUB-013 — Department seat allocation and usage

**Status:** Source-confirmed schema; calculation policy TBD.

Department contains:

- `allocatedSeats`
- `usedSeats`

**TBD:** how seats are allocated to departments, whether the sum of Department allocatedSeats must be less than or equal to College totalSeats, whether unallocated seats remain available at College level, and whether allocation can be changed after students are assigned.

Detailed seat accounting and enforcement must be coordinated with Module 2 and user/enrollment behavior in Modules 1 and 4.

### SUB-014 — Seat usage definition

**Status:** TBD / Decision Required.

The source includes seat usage fields but does not define exactly what counts as a used seat.

Possible triggering events (not yet approved) include user creation, account activation, enrollment, or assignment to a College/Department. Do not implement seat increments/decrements until the canonical seat-counting event is approved.

### SUB-015 — Generate and manage Invoices

**Status:** Source-confirmed schema and page mapping.

Invoice fields:

- `id`
- `invoiceNumber`
- `subscriptionId`
- `collegeId`
- `amount`
- `taxAmount`
- `currency`
- `status`
- `pdfUrl`
- `paidAt`
- `createdAt`
- `updatedAt`

Invoice statuses from schema:

- `PAID`
- `OPEN`
- `VOID`
- `UNCOLLECTIBLE`

SUPER_ADMIN has Invoice Management; Finance Team has Invoice History.

**TBD:** invoice numbering, generation timing, due date (no dueDate field shown), invoice line items, PDF generation, download permissions, correction/void process, and whether an invoice is generated for initial subscription, renewal, and seat add-on purchases.

### SUB-016 — Record Payment Transactions

**Status:** Source-confirmed schema.

PaymentTransaction fields:

- `id`
- `idempotencyKey`
- `gatewayTransactionId`
- `gateway`
- `subscriptionId`
- `collegeId`
- `invoiceId`
- `pricingQuoteId`
- `amount`
- `currency`
- `status`
- `payload`
- `createdAt`

Payment gateway enum:

- `STRIPE`
- `RAZORPAY`
- `MANUAL_BANK_TRANSFER`

Payment status enum:

- `SUCCESS`
- `FAILED`
- `PENDING`
- `REFUNDED`

**TBD:** gateway selected for production, hosted checkout vs direct API, bank transfer confirmation process, webhook validation, transaction reconciliation, refund rules, and which payload fields may be stored (sensitive data must not be stored unnecessarily).

### SUB-017 — Link payments, invoices, quotes, and subscriptions

**Status:** Source-confirmed schema relationships.

PaymentTransaction can reference a Subscription, College, Invoice, and PricingQuote. Invoice references Subscription and College.

**Derived requirement:** billing records must remain traceable across the purchase/renewal flow.

**TBD:** which references are mandatory for each payment type, whether one invoice can have multiple payment transactions, partial payments, split payments, and how failed/retried payments relate to previous transactions.

### SUB-018 — Payment result handling

**Status:** Source-confirmed status values; flow TBD.

PaymentTransaction supports PENDING, SUCCESS, FAILED, and REFUNDED.

**TBD:** authoritative source of payment status (gateway callback, webhook, manual finance confirmation, or reconciliation), when Invoice becomes PAID, when subscription becomes ACTIVE, and how duplicate or delayed payment notifications are handled.

### SUB-019 — Subscription and seat access gating

**Status:** TBD / Decision Required.

The source documents do not define which DEZOLVER functionality is available when:
- Subscription is ACTIVE
- Subscription is in GRACE_PERIOD
- Subscription is SUSPENDED
- Subscription is CANCELLED
- College is separately SUSPENDED/DEACTIVATED
- Seats are fully used or exceeded

This policy must be defined before access checks, user provisioning, enrollment, or execution limits are finalized. Subscription status must not be treated as equivalent to College status without an approved rule.

### SUB-020 — Billing data access and isolation

**Status:** Source-confirmed RBAC principle.

All subscription, invoice, payment, quote, and seat-related queries must be scoped appropriately.

- Finance Team can access only billing data for its assigned College.
- SUPER_ADMIN can access platform-wide billing records.
- HOD's billing/subscription View permission must be restricted to its permitted scope.
- Any Student “View own” billing permission requires clarification because no student billing page is listed in Page-to-DB Mapping.

---

## 6. High-level workflows

### 6.1 Initial college subscription — source-grounded skeleton

The Business Flow v2 shows:

`SUPER_ADMIN → College Onboarding → College Created → Setup → Finance Team → Subscription`

The schema indicates relevant records may include CollegeOnboarding, Plan, PricingQuote, Subscription, Invoice, and PaymentTransaction.

A possible record relationship is visible in the schema, but the exact sequence of quote approval, invoice creation, payment, and subscription activation is **not defined** in the source documents.

**Do not treat the following as a finalized sequence until approved:**

1. Select Plan (DEFAULT or CUSTOM).
2. Determine pricing and seat quantity.
3. Create/approve PricingQuote if applicable.
4. Create invoice.
5. Record payment.
6. Activate Subscription and set period dates.

The exact ordering and whether any step is optional are TBD.

### 6.2 Finance Team views billing

Source page mapping supports:

1. Finance Team opens Billing & Subscription for its College.
2. Views Plan and Subscription details.
3. Views Invoice History.
4. Accesses Buy More Seats.

The specific page filters, buttons, payment action, and error states remain TBD.

### 6.3 Renewal — source-grounded skeleton

The Business Flow identifies Renewal, and Subscription stores current period dates and renewalDate.

The following are required to be specified, but not yet source-defined:
- Reminder timing
- Invoice creation timing
- Payment deadline
- Renewal success transition
- Renewal failure and grace period
- Seat/plan changes during renewal

---

## 7. Data model reference

### 7.1 Plan

| Field | Schema type | Meaning |
|---|---|---|
| `id` | varchar | Plan identifier |
| `name` | varchar(255) | Plan name |
| `description` | text | Plan description |
| `price` | decimal(10,2) | Price value; billing unit TBD |
| `currency` | varchar(10) | Currency |
| `durationMonths` | int | Plan duration in months |
| `features` | json | Feature configuration |
| `maxStudents` | int | Student capacity value |
| `isActive` | boolean | Plan active flag |
| `createdAt`, `updatedAt` | timestamp | Record timestamps |

### 7.2 PricingQuote

| Field | Schema type | Meaning |
|---|---|---|
| `id` | varchar | Quote identifier |
| `collegeOnboardingId` | varchar | Onboarding reference |
| `planId` | varchar | Plan reference |
| `studentCount` | int | Quoted student count |
| `baseAmount` | decimal(10,2) | Base amount |
| `discountAmount` | decimal(10,2) | Discount amount |
| `taxAmount` | decimal(10,2) | Tax amount |
| `totalAmount` | decimal(10,2) | Total amount |
| `currency` | varchar(10) | Currency |
| `status` | varchar(30) | Quote status; values TBD |
| `validUntil` | timestamp | Quote expiry timestamp |
| `createdById` | varchar | Creator |
| `createdAt`, `updatedAt` | timestamp | Record timestamps |

### 7.3 Subscription

| Field | Schema type | Meaning |
|---|---|---|
| `id` | varchar | Subscription identifier |
| `collegeId` | varchar | College reference |
| `planId` | varchar | Plan reference |
| `status` | SubscriptionStatus | ACTIVE / GRACE_PERIOD / SUSPENDED / CANCELLED |
| `purchasedSeats` | int | Purchased seat quantity |
| `currentPeriodStart` | timestamp | Period start |
| `currentPeriodEnd` | timestamp | Period end |
| `renewalDate` | timestamp | Renewal date |
| `gracePeriodEndsAt` | timestamp | Grace-period end |
| `createdAt`, `updatedAt` | timestamp | Record timestamps |

### 7.4 Invoice

| Field | Schema type | Meaning |
|---|---|---|
| `id` | varchar | Invoice identifier |
| `invoiceNumber` | varchar(100) | Invoice number |
| `subscriptionId` | varchar | Subscription reference |
| `collegeId` | varchar | College reference |
| `amount` | decimal(10,2) | Invoice amount |
| `taxAmount` | decimal(10,2) | Tax amount |
| `currency` | varchar(10) | Currency |
| `status` | InvoiceStatus | PAID / OPEN / VOID / UNCOLLECTIBLE |
| `pdfUrl` | varchar(1024) | Invoice PDF location |
| `paidAt` | timestamp | Payment timestamp |
| `createdAt`, `updatedAt` | timestamp | Record timestamps |

### 7.5 PaymentTransaction

| Field | Schema type | Meaning |
|---|---|---|
| `id` | varchar | Transaction identifier |
| `idempotencyKey` | varchar(255) | Idempotency key |
| `gatewayTransactionId` | varchar(255) | External gateway transaction ID |
| `gateway` | PaymentGateway | STRIPE / RAZORPAY / MANUAL_BANK_TRANSFER |
| `subscriptionId` | varchar | Subscription reference |
| `collegeId` | varchar | College reference |
| `invoiceId` | varchar | Invoice reference |
| `pricingQuoteId` | varchar | Quote reference |
| `amount` | decimal(10,2) | Transaction amount |
| `currency` | varchar(10) | Currency |
| `status` | PaymentStatus | SUCCESS / FAILED / PENDING / REFUNDED |
| `payload` | json | Gateway/transaction payload; storage rules TBD |
| `createdAt` | timestamp | Creation timestamp |

### 7.6 Related seat fields

| Entity | Field | Schema meaning |
|---|---|---|
| College | `totalSeats` | College seat capacity |
| College | `usedSeats` | College seat usage |
| Department | `allocatedSeats` | Department seat allocation |
| Department | `usedSeats` | Department seat usage |
| Subscription | `purchasedSeats` | Subscription purchased seats |
| Plan | `maxStudents` | Plan student limit |

The relationship among these counters is not fully defined in the source.

---

## 8. Validation and consistency requirements

The following are derived safeguards; detailed business arithmetic remains subject to approval.

| Rule ID | Validation | Status |
|---|---|---|
| SUB-V01 | Subscription must reference a valid College and Plan. | Derived from schema relationships |
| SUB-V02 | Quote currency, invoice currency, and payment currency must be consistent unless an approved conversion process exists. | Derived |
| SUB-V03 | Payment transaction must be traceable to its intended purchase, subscription, invoice, or quote according to approved transaction type rules. | Derived |
| SUB-V04 | Finance Team queries must be restricted to its assigned College. | Source-confirmed RBAC |
| SUB-V05 | Seat usage must not silently exceed capacity; define enforcement and exception handling. | Derived; exact policy TBD |
| SUB-V06 | College/Department seat counters must be reconciled to the approved seat accounting rules. | Derived |
| SUB-V07 | Quote expiration must be checked before accepting quote-based purchase, if quote expiry is enforced. | Schema supports validUntil; behavior TBD |
| SUB-V08 | Payment transaction idempotency must prevent duplicate processing of the same logical payment where applicable. | Derived from schema idempotencyKey; implementation details TBD |
| SUB-V09 | Invoice and Subscription status must be updated consistently after confirmed payment/renewal. | Derived; exact state rules TBD |
| SUB-V10 | A Plan with `isActive=false` must have defined availability rules for new purchases and existing subscriptions. | TBD |

---

## 9. Source gaps, conflicts, and risks

### 9.1 Student subscription/billing access discrepancy

RBAC includes Student “View own” for subscription/billing in a matrix, but Page-to-DB Mapping does not list a Student billing/subscription page.

**Decision required:** should Student see any billing information, or is this RBAC row intended only for college/Finance Team subscription records?

### 9.2 Plan pricing model is not specified

Plan includes `price`, `currency`, `durationMonths`, and `maxStudents`, but the documents do not state whether price is per student, per college, per duration, or a tiered/custom formula.

**Decision required:** approve the billing unit and pricing formula.

### 9.3 Default vs Custom plan semantics

Business Flow shows DEFAULT and CUSTOM, but schema has one Plan model and no plan-type enum.

**Decision required:** define custom plan representation and approval.

### 9.4 PricingQuote status is a string

The schema shows `PricingQuote.status varchar(30)`, with no enum values.

**Decision required:** define allowed quote states and transitions.

### 9.5 Subscription period and renewal date semantics

The schema contains currentPeriodStart, currentPeriodEnd, renewalDate, and gracePeriodEndsAt, but no rules for date equality, time zones, or date extension.

**Decision required:** canonical period/renewal calculations.

### 9.6 Seat counters may overlap

Subscription `purchasedSeats`, College `totalSeats`, College `usedSeats`, Department `allocatedSeats`, Department `usedSeats`, and Plan `maxStudents` all represent related capacity concepts.

**Risk:** independent updates can cause inconsistent seat availability.

**Decision required:** define a canonical capacity source, counter ownership, reconciliation, and seat-counting events.

### 9.7 Payment gateway configuration

Schema lists Stripe, Razorpay, and Manual Bank Transfer. This does not prove all are integrated or enabled.

**Decision required:** which gateway(s) are enabled for the target deployment, and whether manual bank transfer requires Finance Team verification.

### 9.8 Invoice due date and overdue behavior

Invoice has no `dueDate` field in the supplied schema, and InvoiceStatus has no explicit OVERDUE value.

**Decision required:** whether overdue invoices are represented by OPEN plus dates/metadata, another status, or a schema extension.

### 9.9 Refund, partial payment, and payment retry rules

Payment status includes REFUNDED, but the source does not define refund workflows, partial payments, retries, or transaction reversal semantics.

**Decision required:** define financial operations and their authorization/audit requirements.

### 9.10 Subscription status vs College status

College has EntityStatus; Subscription has SubscriptionStatus. The source does not say whether these states are coupled.

**Decision required:** define whether College status overrides subscription status, whether subscription suspension changes College status, and how independent statuses interact.

---

## 10. Decisions required before finalization

| ID | Decision | Why it matters |
|---|---|---|
| SUB-D01 | Define plan pricing unit: per college, per seat, per month/year, tiered, or custom. | Pricing calculation |
| SUB-D02 | Define DEFAULT vs CUSTOM plan semantics and who may create/approve custom plans. | Plan administration |
| SUB-D03 | Define Plan features and whether features are enforced as entitlement flags. | Feature access |
| SUB-D04 | Define whether multiple subscriptions per College are allowed and what counts as the active subscription. | Data model and billing |
| SUB-D05 | Define initial subscription activation sequence: quote, invoice, payment, activation. | Onboarding flow |
| SUB-D06 | Define whether PricingQuote is required for every purchase or only custom/onboarding purchases. | Quote workflow |
| SUB-D07 | Define PricingQuote status values, approval, expiry, revision, and acceptance. | Quote lifecycle |
| SUB-D08 | Define tax and discount formulas, rounding, and currency policy. | Financial accuracy |
| SUB-D09 | Define renewal mode (manual/automatic) and reminder schedule. | Renewal behavior |
| SUB-D10 | Define renewal date calculation and timezone convention. | Period consistency |
| SUB-D11 | Define grace-period duration and allowed features during grace. | Access policy |
| SUB-D12 | Define suspension/cancellation policy, timing, reversibility, and refund handling. | Subscription lifecycle |
| SUB-D13 | Define which roles may create/update/suspend/cancel subscriptions and manage invoices/payments. | RBAC |
| SUB-D14 | Define which payment gateway(s) are active and how manual bank transfers are verified. | Payment integration |
| SUB-D15 | Define payment webhook/callback confirmation and transaction reconciliation. | Payment correctness |
| SUB-D16 | Define partial payments, retries, refunds, and duplicate transaction behavior. | Financial operations |
| SUB-D17 | Define invoice numbering, due date, line items, PDF generation, and void/unc collectible process. | Invoice lifecycle |
| SUB-D18 | Define whether Student billing access exists or remove/clarify the RBAC “View own” entry. | UI/RBAC consistency |
| SUB-D19 | Define seat counting event: user creation, activation, enrollment, or another trigger. | Seat usage |
| SUB-D20 | Define canonical relationship between Plan.maxStudents, Subscription.purchasedSeats, College.totalSeats, and College.usedSeats. | Capacity model |
| SUB-D21 | Define department seat allocation, over-allocation prevention, reallocation, and usedSeats updates. | Department capacity |
| SUB-D22 | Define Buy More Seats quantity semantics, proration, quote/invoice/payment flow, and effective date. | Seat purchase |
| SUB-D23 | Define whether seat reduction is allowed and what happens if current usage exceeds the reduced capacity. | Capacity enforcement |
| SUB-D24 | Define access behavior when subscription expires, is in grace period, is suspended, or is cancelled. | Platform access |
| SUB-D25 | Define interaction between College status and Subscription status. | Tenant lifecycle |
| SUB-D26 | Define whether existing subscribers are affected when a Plan is edited or deactivated. | Plan versioning |
| SUB-D27 | Define audit events and notifications for billing/renewal/payment operations in Modules 13 and 15. | Traceability |
| SUB-D28 | Define how quote, invoice, and transaction records are retained and corrected. | Financial history |

---

## 11. Acceptance checklist

Module 3 is implementation-ready when:

- [ ] Plan pricing units and DEFAULT/CUSTOM behavior are approved.
- [ ] Plan feature entitlements and maxStudents behavior are defined.
- [ ] Subscription creation and activation sequence is finalized.
- [ ] Subscription status transitions and access behavior are defined.
- [ ] Renewal and grace-period rules are finalized.
- [ ] Invoice generation, numbering, due date, PDF, and status rules are defined.
- [ ] Payment gateway, confirmation, idempotency, retries, and refunds are defined.
- [ ] Buy More Seats workflow and proration/effective-date rules are approved.
- [ ] Seat counting and College/Department counter reconciliation are defined.
- [ ] Finance Team and SUPER_ADMIN permissions are explicit and tenant-scoped.
- [ ] Student billing access discrepancy is resolved.
- [ ] Subscription/College status interaction is defined.
- [ ] Billing audit and notification requirements are coordinated with Modules 13 and 15.

---

## 12. Dependencies and interfaces

- **Module 1 — Authentication & User Lifecycle:** Finance Team identity, role, College assignment, and user/account eligibility.
- **Module 2 — College / Department / Section Management:** College/Department seat fields, organizational hierarchy, and tenant status.
- **Module 4 — Course & Enrollment:** seat usage triggers and enrollment eligibility.
- **Module 13 — Notifications:** renewal reminders, invoice/payment notifications, and payment failures.
- **Module 15 — Audit Logging:** subscription, seat, quote, invoice, and payment audit events.
- **Module 17 — Functional State Machines:** Plan/Quote/Subscription/Invoice/Payment state transitions.
- **Module 18 — Edge Cases & Failure Handling:** payment gateway outage, duplicate callbacks, failed renewal, over-capacity, and reconciliation.

---

## 13. Source references

- DEZOLVER Drive: **Business Flow v2.pdf**
- DEZOLVER Drive: **RBAC (1).pdf**
- DEZOLVER Drive: **scheema.pdf**
- DEZOLVER Drive: **Page - DB Mapping v2.pdf**

Drive links:
- [Business Flow v2.pdf](https://drive.google.com/file/d/1rE9q1h0u9FTVLJmlP2Wl7WWfKFqsETgL/view)
- [RBAC (1).pdf](https://drive.google.com/file/d/1Z_q_3N4cd2cKljAmUZnpHIWyGXPFBg3W/view)
- [scheema.pdf](https://drive.google.com/file/d/10_LGfTsQ2eGmbJM5tQE9Pkmlwhft1lAI/view)
- [Page - DB Mapping v2.pdf](https://drive.google.com/file/d/1UdzowkDPP85PNI0-GQ2k4cEmwIIAZYcw/view)

**Review gate:** Module 3 only. Do not begin Module 4 until the user reviews this document and explicitly approves proceeding.


---

# Functional Specification Standard v1.0 — compliance supplement

The existing source-based content above remains the source summary and high-level requirement baseline. The following function inventory and function-level contracts expand it to the authoring standard. This is not an approval of the open financial/product decisions.

## 14. Function inventory

| ID | Function | Actor | Type | Status |
|---|---|---|---|---|
| SUB-F001 | View/manage Plan catalogue | SUPER_ADMIN | CRUD/read | Decision required |
| SUB-F002 | Configure DEFAULT/CUSTOM plan | SUPER_ADMIN | Configuration | Decision required |
| SUB-F003 | View College subscription | SUPER_ADMIN/Finance Team/HOD as permitted | Read | Draft |
| SUB-F004 | Create subscription | SUPER_ADMIN/Finance Team permission TBD | Workflow/create | Decision required |
| SUB-F005 | Update subscription | SUPER_ADMIN/Finance Team scoped | Update | Decision required |
| SUB-F006 | Change subscription status | SUPER_ADMIN/Finance Team permission TBD | Lifecycle | Decision required |
| SUB-F007 | View active period and renewal dates | SUPER_ADMIN/Finance Team/HOD view | Read | Draft |
| SUB-F008 | Renew subscription | Finance Team/SUPER_ADMIN | Workflow | Decision required |
| SUB-F009 | Apply grace period | System/authorized admin | Lifecycle | Decision required |
| SUB-F010 | Suspend/cancel subscription | SUPER_ADMIN/Finance Team permission TBD | Lifecycle | Decision required |
| SUB-F011 | Create/view/update PricingQuote | SUPER_ADMIN/Finance Team permission TBD | Quote workflow | Decision required |
| SUB-F012 | Calculate quote amounts | System | Calculation | Decision required |
| SUB-F013 | Buy More Seats | Finance Team | Purchase workflow | Decision required |
| SUB-F014 | View and reconcile College/Department seats | Finance Team/SUPER_ADMIN | Read/reconciliation | Decision required |
| SUB-F015 | Allocate seats to Department | Finance Team/HOD permission TBD | Allocation | Decision required |
| SUB-F016 | Determine seat usage | System/Module 1 or 4 trigger | Calculation | Decision required |
| SUB-F017 | Generate/manage invoice | SUPER_ADMIN/Finance Team | Billing workflow | Decision required |
| SUB-F018 | View/download invoice | SUPER_ADMIN/Finance Team/HOD/student access TBD | Read | Decision required |
| SUB-F019 | Initiate/record payment | Finance Team/SUPER_ADMIN/gateway | Payment workflow | Decision required |
| SUB-F020 | Process payment result and reconciliation | Gateway/Finance Team/system | System workflow | Decision required |
| SUB-F021 | Link payment, invoice, quote and subscription | System | Data integrity | Derived |
| SUB-F022 | Enforce billing/seat scope and access entitlements | System | Security/authorization | Derived |
| SUB-F023 | Handle subscription access gating | System | Authorization/lifecycle | Decision required |
| SUB-F024 | Audit billing and subscription changes | System | Audit integration | Module 15 dependency |

## 15. Function-level specification supplement

The detail below follows the Standard’s required questions: actor, permission/scope, preconditions, entry point, UI behavior, inputs, flow, rules, system/data behavior, success, alternate/failure paths, audit/privacy, acceptance, QA, and unresolved decisions. UI details not in the source are marked TBD.

### SUB-F001 — View and manage Plan catalogue

**Requirement status:** Plan schema and SUPER_ADMIN Plan Management page are source-confirmed.  
**Specification status:** Decision required. **Actor:** SUPER_ADMIN. **Supporting actor:** System.

- **Purpose/outcome:** Create, inspect, and maintain plan records represented by Plan.
- **Permission/scope:** SUPER_ADMIN global plan administration. Finance Team can view plan context in Business Flow; its plan write permission is not shown. HOD/Manager/Student plan administration is not shown.
- **Preconditions:** Authenticated SUPER_ADMIN; plan field/constraint policy approved.
- **Entry/UI:** Plan Management page mapped to Plan/Subscription. Search, filters, create/edit form, confirmation and page layout are TBD.
- **Inputs:** name varchar(255), description text, price decimal(10,2), currency varchar(10), durationMonths int, features JSON, maxStudents int, isActive boolean. Requiredness, ranges, price units and JSON structure TBD.
- **Main flow:** open Plan Management → server verifies global role → list permitted plans → select create/edit → validate approved field constraints → persist → return updated plan.
- **Rules:** plan entity is source-confirmed; Plan price and maxStudents do not define pricing or seat semantics alone. Do not change historical subscription terms without approved policy.
- **System/data:** Plan record affected; subscription records may reference planId. Versioning/snapshot behavior TBD.
- **Success:** Plan visible with saved values.
- **Alternate/failure:** unauthorized write denied; invalid input rejected; duplicate names/IDs behavior TBD; concurrent edits/versioning TBD.
- **Audit/privacy:** plan changes should be coordinated with Module 15; exact event list TBD.
- **Acceptance:** only authorized role can modify; all accepted fields follow approved constraints; no unapproved retroactive subscription change.
- **QA:** SUB-QA-001–006.
- **Decision refs:** SUB-D01/D03/D26.

### SUB-F002 — Configure DEFAULT/CUSTOM plan selection

**Requirement status:** DEFAULT and CUSTOM labels are source-confirmed in Business Flow v2. Semantics are TBD.

- **Actor:** SUPER_ADMIN configures; Finance Team views/selects within its assigned College if allowed.
- **Entry:** Plan selection in subscription/Finance workflow; page layout TBD.
- **Inputs:** plan reference, plan category/type (no corresponding schema enum shown), custom pricing/features/seat values (contract TBD).
- **Flow:** display approved available choices → actor selects → system validates role and eligibility → persist selected plan/quote association according to approved model.
- **Rule:** do not assume DEFAULT/CUSTOM is a Plan enum, custom quote, per-college override, or separate subscription type.
- **Data:** Plan/PricingQuote/Subscription relationships; exact custom representation requires decision.
- **Failure:** inactive plan selection, expired quote, or unauthorized custom selection behavior TBD.
- **Acceptance:** system only offers/selects categories after semantics are approved; CUSTOM cannot silently bypass pricing/capacity policy.
- **QA:** SUB-QA-007–010.
- **Decision refs:** SUB-D02/D03/D06.

### SUB-F003 — View College subscription

**Requirement status:** Subscription schema, Page mapping, RBAC source-confirmed.  
**Actors:** SUPER_ADMIN, Finance Team; HOD read permission is shown in RBAC.

- **Scope:** SUPER_ADMIN global; Finance Team assigned College; HOD assigned College+Department and view permission (exact fields TBD).
- **Preconditions:** authenticated actor, existing permitted College/subscription.
- **Entry/UI:** SUPER_ADMIN Subscription Management; Finance Team Billing & Subscription; HOD view access route TBD.
- **Inputs:** College/subscription selector and query filters (UI/API names TBD).
- **Flow:** request view → resolve actor scope server-side → fetch subscription joined to permitted College/Plan → render approved fields.
- **Data:** read Subscription, College, Plan; no mutation.
- **Rules:** do not leak other tenant subscription/payment records. HOD read is limited to its approved view fields.
- **Failure:** missing subscription, no access, multiple subscription records, stale data behavior TBD.
- **Acceptance:** Finance Team cannot query another College; list/detail/direct-ID paths all scope-filtered.
- **QA:** SUB-QA-011–015.
- **Decision refs:** SUB-D04/D13/D18.

### SUB-F004 — Create subscription

**Requirement status:** Subscription entity/College/Plan references source-confirmed; exact activation workflow TBD.

- **Actors:** SUPER_ADMIN; Finance Team creation permission follows Subscription & Billing View/Create/Update RBAC, but specific lifecycle action policy TBD.
- **Preconditions:** College and Plan exist; role and scope valid; subscription cardinality/initialization rules approved.
- **Entry:** Subscription Management or onboarding flow; exact trigger TBD.
- **Inputs:** collegeId, planId, purchasedSeats, status, currentPeriodStart, currentPeriodEnd, renewalDate, gracePeriodEndsAt. Requiredness/defaults/initial status TBD.
- **Flow:** select College and Plan → input approved quantity/period fields → validate College/Plan/actor scope → create record → return subscription reference.
- **Rules:** Subscription references College and Plan; multiple active subscriptions per College not resolved. Initial status cannot be assumed ACTIVE before approved payment/activation workflow.
- **Data:** create Subscription; College counters update only according to Module 3 seat accounting decision.
- **Failure:** missing parent, inactive Plan, duplicate active subscription, payment not completed, or retry behavior TBD.
- **Acceptance:** no invalid College/Plan reference; creation follows approved activation ordering; Finance Team scoped to its College.
- **QA:** SUB-QA-016–020.
- **Decision refs:** SUB-D04/D05/D13/D20.

### SUB-F005 — Update subscription

- **Requirement status:** RBAC includes SUPER_ADMIN Full CRUD and Finance Team View/Create/Update.
- **Actor:** SUPER_ADMIN or Finance Team within assigned College.
- **Editable fields:** purchasedSeats, planId, period dates, status and renewal fields are all sensitive; field-specific allowlists TBD.
- **Flow:** open permitted Subscription → edit allowed fields → server verifies role, scope and lifecycle constraints → save → show result.
- **Rules:** changing plan/seats/dates may affect billing/access; do not allow direct edits to payment-derived fields unless approved.
- **Failure:** stale record, invalid date order, unauthorized field mutation, foreign College ID → reject without mutation.
- **Acceptance:** Finance Team cannot change another College; updates conform to field allowlist and approved state transitions.
- **QA:** SUB-QA-021–025.
- **Decision refs:** SUB-D10/D13/D20/D22.

### SUB-F006 — Change subscription status

- **Source:** enum ACTIVE, GRACE_PERIOD, SUSPENDED, CANCELLED. Transition rules TBD.
- **Actors:** system/admin; exact role permissions TBD.
- **Entry:** Subscription Management action or system event.
- **Input:** target status, reason/comment if approved, effective time (TBD).
- **Flow:** actor/system requests transition → current status and authorization checked → Module 17 transition matrix applied → update status and approved dates → audit event.
- **Rules:** enum membership alone does not authorize transition or determine feature gating. No automatic College status coupling.
- **Failure:** invalid transition, wrong scope, missing required reason, stale status → reject/no change.
- **Acceptance:** only approved transition pairs accepted; downstream access follows Module 17/1/18 rules.
- **QA:** SUB-QA-026–030.
- **Decision refs:** SUB-D11/D12/D24/D25.

### SUB-F007 — View active period and renewal dates

- **Source:** currentPeriodStart, currentPeriodEnd, renewalDate and gracePeriodEndsAt schema fields.
- **Actors:** SUPER_ADMIN/Finance Team; HOD view as permitted.
- **UI:** show approved date values; format/timezone and labels TBD.
- **Flow:** retrieve subscription → check scope → display stored dates and period status.
- **Rules:** no assumption that renewalDate equals currentPeriodEnd; date inclusivity/timezone TBD.
- **Failure:** absent date, invalid stored date, timezone discrepancy → behavior TBD.
- **Acceptance:** displayed values map to stored fields and are tenant-scoped.
- **QA:** SUB-QA-031–033.
- **Decision refs:** SUB-D10/D11.

### SUB-F008 — Renew subscription

- **Source:** Renewal appears in Business Flow; exact workflow TBD.
- **Actors:** Finance Team/SUPER_ADMIN, with Finance Team limited to assigned College.
- **Preconditions:** subscription renewal eligibility, plan and price policy, payment workflow approved.
- **Entry:** Renewal action or scheduled process; trigger TBD.
- **Inputs:** subscription, renewal term/plan/seats, quote/invoice/payment references as approved.
- **Flow:** show renewal options → validate subscription and scope → calculate quote using approved pricing rules → create billing records if required → payment confirmation → update period/status only when approved success condition occurs.
- **Rules:** do not extend period from payment time vs existing end date without approved rule; do not presume auto-renewal.
- **Failure:** payment failure, quote expiry, renewal race, subscription already renewed, gateway unavailable → decision-gated.
- **Acceptance:** dates/status change only after approved renewal conditions; duplicate renewal does not create duplicate periods/charges.
- **QA:** SUB-QA-034–039.
- **Decision refs:** SUB-D05/D09/D10/D15/D24.

### SUB-F009 — Apply grace period

- **Source:** GRACE_PERIOD and gracePeriodEndsAt are schema-confirmed; policy TBD.
- **Actor:** system or authorized admin; trigger/role TBD.
- **Input:** subscription, grace end date/duration (policy TBD).
- **Flow:** approved event triggers grace → validate transition → store status/gracePeriodEndsAt → access policy applied via Module 17/1.
- **Rules:** duration, trigger, user access, extension, and end behavior are not inferred from status name.
- **Failure:** invalid grace end, repeated grace extension, grace expiry processing → TBD.
- **Acceptance:** grace state and timestamp follow approved policy; downstream entitlement is not assumed.
- **QA:** SUB-QA-040–043.
- **Decision refs:** SUB-D11/D24.

### SUB-F010 — Suspend or cancel subscription

- **Source:** SUSPENDED/CANCELLED status values.
- **Actor:** SUPER_ADMIN/Finance Team permissions TBD.
- **Inputs:** action, effective timing, reason, refund/cancellation context if approved.
- **Flow:** request → authorize and scope-check → validate transition → confirm if policy requires → set state/effective dates → audit.
- **Rules:** immediate vs end-of-period cancellation, refunds, reversal, and service access are TBD.
- **Failure:** already suspended/cancelled, unauthorized College, dependent financial action failure → behavior TBD.
- **Acceptance:** state change is permitted only by approved transition/permission; refund and College status do not change automatically.
- **QA:** SUB-QA-044–047.
- **Decision refs:** SUB-D12/D13/D16/D25.

### SUB-F011 — Create/view/update PricingQuote

- **Source:** PricingQuote entity fields and Plan/CollegeOnboarding relationship.
- **Actors:** SUPER_ADMIN/Finance Team (specific create/update permission TBD).
- **Inputs:** collegeOnboardingId, planId, studentCount, baseAmount, discountAmount, taxAmount, totalAmount, currency, status, validUntil, createdById.
- **Flow:** select onboarding/Plan → enter approved quote parameters → server checks scope and allowed plan → quote calculated per approved formula → save quote → display reference/expiry.
- **Rules:** status values, quote approval, expiry, revisions, and requirement for quote are TBD. `createdById` should represent actual creator under approved identity model.
- **Failure:** invalid Plan/onboarding, expired quote, duplicate/revision race, invalid currency → reject/decision-gated.
- **Acceptance:** quote links to valid Plan and onboarding record; actor scope enforced; amounts are not invented or recalculated under unapproved formula.
- **QA:** SUB-QA-048–053.
- **Decision refs:** SUB-D06/D07/D08.

### SUB-F012 — Calculate quote amounts

- **Source:** fields baseAmount, discountAmount, taxAmount, totalAmount exist; formula not source-defined.
- **Actor:** system, based on approved pricing policy.
- **Inputs:** price basis, studentCount, discount, tax, currency, plan, custom terms (rules TBD).
- **Flow:** load approved pricing policy → validate values → compute amounts/rounding → persist quote calculation breakdown → return totals.
- **Rule:** do not assume arithmetic order or exact formula from names. Formula must be approved (SUB-D08).
- **Failure:** unsupported currency, invalid quantity, missing tax rule, rounding boundary → no quote finalized until policy resolved.
- **Acceptance:** same inputs/policy produce consistent values; line/breakdown reconciliation follows approved formula.
- **QA:** SUB-QA-054–057.
- **Decision refs:** SUB-D01/D08.

### SUB-F013 — Buy More Seats

- **Source:** Finance Team Buy More Seats page; Plan/Subscription/College mapping.
- **Actor:** Finance Team for assigned College; SUPER_ADMIN oversight permission TBD.
- **Inputs:** selected Subscription, additional-seat quantity vs target capacity (TBD), desired plan/custom package (TBD).
- **Flow:** open Buy More Seats → load current plan/seat data → choose quantity under approved semantics → quote/proration policy applied → invoice/payment workflow if required → capacity update on approved effective event.
- **Rules:** purchasedSeats vs totalSeats semantics; effective date, proration, and subscription term extension are TBD.
- **Failure:** no active subscription, seat max reached, payment fails, duplicate click, concurrent purchase → decision-gated.
- **Acceptance:** capacity is increased only on approved confirmation; Finance Team cannot purchase/update another College's seats.
- **QA:** SUB-QA-058–064.
- **Decision refs:** SUB-D20/D22/D23.

### SUB-F014 — View and reconcile College/Department seats

- **Source:** College totalSeats/usedSeats; Subscription purchasedSeats; Department allocatedSeats/usedSeats.
- **Actors:** SUPER_ADMIN/Finance Team; HOD department seat view/update TBD.
- **Flow:** load related records → apply College/Department scope → display values and reconciliation status if approved.
- **Rules:** canonical capacity and counter relationship is unresolved. UI must not label counters as available seats using unapproved formula.
- **Failure:** counter mismatch, negative/overflow value, multiple subscriptions → show/reconcile only under approved policy.
- **Acceptance:** values shown trace to source fields; Finance Team sees only assigned College.
- **QA:** SUB-QA-065–068.
- **Decision refs:** SUB-D19/D20/D21.

### SUB-F015 — Allocate seats to Department

- **Source:** Department allocatedSeats/usedSeats fields. Workflow/permission TBD.
- **Actors:** Finance Team/HOD/SUPER_ADMIN permission must be approved.
- **Inputs:** Department ID, allocation quantity, source College capacity.
- **Flow:** select Department → submit allocation → server checks College scope and allocation constraints → update according to approved policy → return allocation.
- **Rules:** sum of allocations vs College capacity, reallocation, over-use, and student assignment impact TBD.
- **Failure:** allocation exceeds capacity, foreign Department, allocation below current use → policy-dependent.
- **Acceptance:** allocation cannot exceed approved constraints; unauthorized actor denied; no seat count change outside canonical event.
- **QA:** SUB-QA-069–072.
- **Decision refs:** SUB-D21/D23.

### SUB-F016 — Determine seat usage

- **Source:** College/Department usedSeats fields; trigger not specified.
- **Actor:** system triggered by Module 1 user lifecycle or Module 4 enrollment.
- **Input/event:** user create/activate/deactivate, enrollment, assignment, or other trigger—decision required.
- **Flow:** receive approved counting event → identify College/Department → apply idempotent count transition → update counters or derive usage according to approved model.
- **Rules:** one student counted once; handling transfers, deactivation, reactivation, deletion, multiple enrollment, and concurrent events TBD.
- **Failure:** duplicate event, missing organization, counter drift → reconciliation policy TBD.
- **Acceptance:** usage changes only on approved event; retries do not double-count; cross-College transfer follows approved rule.
- **QA:** SUB-QA-073–078.
- **Decision refs:** SUB-D19/D20/D21.

### SUB-F017 — Generate and manage Invoice

- **Source:** Invoice entity, SUPER_ADMIN Invoice Management, Finance Team Invoice History.
- **Actors:** SUPER_ADMIN; Finance Team view/history and invoice creation/update permissions TBD.
- **Inputs:** subscriptionId, collegeId, amount, taxAmount, currency, status, invoiceNumber, pdfUrl, paidAt.
- **Flow:** approved billing event → invoice generation policy → validate subscription/College → assign invoice number and totals → persist Invoice → expose permitted invoice.
- **Rules:** no dueDate/line item schema fields shown; invoice status values PAID/OPEN/VOID/UNCOLLECTIBLE. Overdue and correction policy TBD.
- **Failure:** duplicate invoice trigger, invalid subscription, invoice number conflict, PDF generation failure → policy TBD.
- **Acceptance:** invoice links to valid Subscription/College; no unauthorized invoice access; duplicate trigger follows idempotency rule.
- **QA:** SUB-QA-079–084.
- **Decision refs:** SUB-D05/D17/D28.

### SUB-F018 — View/download Invoice

- **Source:** SUPER_ADMIN Invoice Management; Finance Team Invoice History.
- **Actors:** SUPER_ADMIN and assigned College Finance Team. HOD/Student invoice access is not explicitly mapped.
- **Entry:** invoice history/details; download link/PDF generation behavior TBD.
- **Flow:** request invoice → server checks College scope → retrieve Invoice and approved associated payment/subscription fields → display/download permitted file.
- **Rules:** PDF URL validity, storage access, and data retention TBD. Student “own billing” RBAC mismatch remains open.
- **Failure:** missing invoice, expired link, unauthorized direct URL, PDF unavailable → no cross-tenant disclosure.
- **Acceptance:** invoice history and PDF both enforce same scope.
- **QA:** SUB-QA-085–088.
- **Decision refs:** SUB-D13/D17/D18/D28.

### SUB-F019 — Initiate or record PaymentTransaction

- **Source:** PaymentTransaction schema and Stripe/Razorpay/Manual Bank Transfer enums.
- **Actors:** Finance Team/SUPER_ADMIN and external payment provider; gateway availability is not confirmed.
- **Inputs:** idempotencyKey, gatewayTransactionId, gateway, subscriptionId, collegeId, invoiceId, pricingQuoteId, amount, currency, status, payload.
- **Flow:** initiate approved gateway/manual flow → create pending transaction if policy supports it → validate target invoice/subscription/College and amount → store allowed transaction metadata → await verified outcome.
- **Rules:** no assumption that all enum gateways are implemented; payload must not store unnecessary sensitive payment data; payment status cannot be set from untrusted client input.
- **Failure:** gateway timeout, duplicate request, invalid callback, mismatched amount/currency, failed bank transfer verification → reject/mark pending per approved process.
- **Acceptance:** payment records are scoped and traceable; idempotency prevents duplicate logical transaction; secret/payment data not exposed.
- **QA:** SUB-QA-089–095.
- **Decision refs:** SUB-D14/D15/D16.

### SUB-F020 — Process payment result and reconciliation

- **Source:** payment statuses SUCCESS/FAILED/PENDING/REFUNDED; webhook/confirmation workflow not defined.
- **Actor:** trusted gateway callback, reconciliation service, or authorized Finance Team (mechanism TBD).
- **Flow:** receive status event → verify source/signature or approved manual evidence → match transaction and amount/currency → update status once → reconcile Invoice/Subscription according to approved business rules → audit.
- **Rules:** duplicate/out-of-order callbacks and refund semantics TBD. Do not activate Subscription based on unverified browser return.
- **Failure:** invalid signature, unknown transaction, callback replay, mismatched amount, delayed success after failure → policy decision required.
- **Acceptance:** only trusted confirmation changes payment status; duplicates are idempotent; invoice/subscription side effects follow approved mapping.
- **QA:** SUB-QA-096–101.
- **Decision refs:** SUB-D15/D16.

### SUB-F021 — Link payment, invoice, quote, and subscription

- **Requirement status:** schema relationships source-confirmed; mandatory references by transaction type TBD.
- **Actor:** system.
- **Flow:** create/update billing record → validate related IDs exist and belong to same College/transaction context → save relationship → expose consistent history.
- **Rules:** Invoice references Subscription/College; PaymentTransaction may reference Subscription/College/Invoice/PricingQuote; whether fields are nullable/mandatory is not stated.
- **Acceptance:** no cross-College billing relationship; related record history is consistent.
- **QA:** SUB-QA-102–104.
- **Decision refs:** SUB-D17/D28.

### SUB-F022 — Enforce billing/seat scope

- **Requirement status:** RBAC source-confirmed.
- **Actor:** backend authorization layer.
- **Rules:** SUPER_ADMIN global; Finance Team assigned College; HOD view scope as allowed; Manager no billing permissions shown; Student “own billing” permission is disputed by page mapping.
- **Flow:** authenticate → role/action check → resolve College from trusted context/record relationship → filter/reject → read/write.
- **Failure:** foreign College IDs, direct object ID, altered query, PDF URL tampering → deny without data leak.
- **Acceptance:** every list/detail/mutation/export/download is scope-filtered; frontend alone is insufficient.
- **QA:** SUB-QA-105–109.
- **Decision refs:** SUB-D13/D18.

### SUB-F023 — Apply subscription access gating

- **Requirement status:** not explicitly defined in source; decision required.
- **Actor:** system, relying on approved entitlement policy.
- **Inputs:** subscription status, period dates, College status, seat usage, feature entitlement.
- **Flow:** protected product feature requests entitlement check → resolve subscription/College → apply approved status/seat policy → allow/deny with approved response.
- **Rules:** ACTIVE/GRACE_PERIOD/SUSPENDED/CANCELLED meanings and College-status precedence must be defined. Do not infer access from labels.
- **Acceptance:** gating matches approved policy and is consistent across modules; no access policy is implemented while open.
- **QA:** SUB-QA-110–114.
- **Decision refs:** SUB-D11/D12/D24/D25.

### SUB-F024 — Audit billing/subscription changes

- **Requirement status:** RBAC source identifies sensitive subscription/payment/invoice changes; detailed event list belongs to Module 15.
- **Actor:** system.
- **Flow:** approved financial action occurs → emit audit event with actor, target, scope, action and approved metadata → audit store handles retention/visibility.
- **Privacy:** avoid secrets, full payment credentials, or unnecessary gateway payload in audit.
- **Acceptance:** critical subscription/payment/invoice/seat changes are auditable according to Module 15; audit failure behavior is coordinated with Module 18.
- **QA:** SUB-QA-115–117.
- **Decision refs:** SUB-D27/D28.

## 16. Additional business rules and validation matrix

| Rule | Validation | Status |
|---|---|---|
| SUB-R01 | Subscription must reference existing College and Plan. | Derived from schema |
| SUB-R02 | Finance Team must be scoped to its assigned College for every billing operation. | Source-confirmed RBAC |
| SUB-R03 | Quote/invoice/payment currencies and amounts must reconcile under approved financial policy. | Derived; formula TBD |
| SUB-R04 | Payment idempotencyKey must be used according to approved duplicate transaction policy. | Schema-supported; policy TBD |
| SUB-R05 | Seat capacity and usage must not be calculated from overlapping fields until canonical model is approved. | TBD |
| SUB-R06 | Plan deactivation must not silently rewrite existing subscription history. | Derived safeguard |
| SUB-R07 | Subscription status is not interchangeable with College EntityStatus. | Source gap / derived |
| SUB-R08 | Untrusted client-side payment result must not be treated as confirmed payment. | Security recommendation; requires approval |
| SUB-R09 | Seat usage trigger must be idempotent and consistent across Module 1/4. | Derived |
| SUB-R10 | All invoice/payment/quote relationships must remain within the same College billing context. | Derived |

| Validation ID | Condition | Expected behavior | Status |
|---|---|---|---|
| SUB-V01 | Subscription references missing College/Plan | Reject before write | Derived |
| SUB-V02 | Finance Team requests foreign College billing record | Deny | RBAC |
| SUB-V03 | Quote expired | Reject acceptance/checkout if expiry policy approved | TBD |
| SUB-V04 | Payment amount/currency mismatches quote/invoice | Reject or hold for reconciliation per policy | TBD |
| SUB-V05 | Duplicate payment callback/idempotency key | No duplicate financial effect | Derived; policy TBD |
| SUB-V06 | Seat allocation exceeds allowed capacity | Reject or follow approved override | TBD |
| SUB-V07 | Renewal dates are inconsistent | Reject/hold according to date rules | TBD |
| SUB-V08 | Plan inactive | New purchase eligibility follows approved plan policy | TBD |
| SUB-V09 | Invoice status transition invalid | Reject; preserve state | Module 17/TBD |
| SUB-V10 | College status and Subscription status disagree | Apply approved precedence; do not infer | TBD |

## 17. Edge cases and failure handling

- Two Finance Team users attempt simultaneous seat purchases for the same College.
- Payment provider callback arrives twice or out of order.
- Payment succeeds but invoice/subscription update fails.
- Payment is pending beyond the expected period.
- Manual bank transfer evidence is submitted twice or is insufficient.
- Quote expires while checkout is in progress.
- Plan price/features change after quote creation.
- Subscription period dates cross timezone/daylight/date boundaries.
- Subscription is cancelled while renewal is being processed.
- Seat count changes while a purchase/renewal is pending.
- Department allocation is reduced below current usage.
- College/Department seat counters diverge from Subscription purchasedSeats.
- Student/Finance Team access changes while billing page is open.
- Invoice PDF URL is guessed/shared across College tenants.
- Refund event arrives after subscription renewal/activation.
- College is suspended while Subscription remains ACTIVE.
- Gateway outage, reconciliation timeout, or partial processing.

Each detailed policy is TBD unless explicitly approved in the decision log or Modules 17/18.

## 18. Cross-module dependencies

- Module 1: Finance Team identity, College assignment, role/scope change and user lifecycle events.
- Module 2: College/Department hierarchy, status, seat field ownership.
- Module 4: enrollment/student seat-count trigger and eligibility.
- Module 13: renewal/invoice/payment failure notifications.
- Module 15: billing and payment audit event catalog.
- Module 17: Plan/Quote/Subscription/Invoice/Payment state transitions.
- Module 18: payment retries, partial failure, reconciliation and race conditions.

## 19. Decision log supplement

The initial SUB-D01–SUB-D28 log remains in Section 10. Additional decisions:

| Decision ID | Question | Impact | Owner | Status |
|---|---|---|---|---|
| SUB-D29 | Define per-action permission for plan create/update/deactivate. | Plan security | Product/Security | Open |
| SUB-D30 | Define subscription create/update/status action permissions per role. | Billing security | Product/Security | Open |
| SUB-D31 | Define field allowlist for Finance Team subscription updates. | Tenant billing | Product/Security | Open |
| SUB-D32 | Define quote creator, approver, and revision permissions. | Quote lifecycle | Product/Finance | Open |
| SUB-D33 | Define invoice create/void/correct/download permissions. | Financial integrity | Product/Finance | Open |
| SUB-D34 | Define payment initiation, manual bank confirmation, and refund permissions. | Payment control | Finance/Security | Open |
| SUB-D35 | Define mandatory nullable/required references for PaymentTransaction per transaction type. | Schema/data integrity | Architecture/Finance | Open |
| SUB-D36 | Define invoice/payment event idempotency keys and uniqueness constraints. | Duplicate financial processing | Architecture | Open |
| SUB-D37 | Define invoice PDF storage, signed link, access expiration and retention. | Privacy/security | Architecture/Security | Open |
| SUB-D38 | Define reconciliation and recovery when payment succeeds but related record update fails. | Financial consistency | Architecture/Finance | Open |
| SUB-D39 | Define feature/seat access response and UI behavior for each subscription state. | Product availability | Product/Security | Open |
| SUB-D40 | Define subscription/seat change concurrency and effective-time policy. | Billing correctness | Architecture/Finance | Open |

## 20. Acceptance criteria and review gate

Module 3 is ready for product review when:
- Every function in SUB-F001–SUB-F024 has source/evidence status and a clear actor/scope.
- Plan, subscription, quote, invoice, payment and seat functions have inputs, flow, validation, data/state, success/failure and QA references.
- Finance Team tenant isolation is explicit for reads, writes, downloads, payment, quote and seat operations.
- No price formula, renewal rule, gateway integration, refund policy, seat-count trigger, or access gating is invented.
- SUB-D01–SUB-D40 are reviewed/assigned; product/finance/security decisions remain open.
- Acceptance criteria are testable and QA tests include expected results.
- Dependencies on Modules 1, 2, 4, 13, 15, 17 and 18 are explicit.

**Review gate:** Module 3 only. Commit status means document versioning, not approval of billing/financial behavior. Do not treat any recommendation or open decision as implementation-ready.
