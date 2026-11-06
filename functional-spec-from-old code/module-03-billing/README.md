# Module 3 — Subscription & Seat Management

## 1. Module Executive Summary

Module 3 governs commercial billing, multi-tier subscription lifecycles, payment processing, invoice generation, payout settlements, and rate-limit tier enforcement across the Dezolver platform. It interfaces directly with the Razorpay payment gateway (supporting both live and stub development modes) and manages automated financial recurring workflows via scheduled cron tasks.

A foundational forensic discovery in the legacy codebase is that **Seat Management, Per-User Licensing, and Seat Quota Enforcement DO NOT EXIST** in the implementation. Subscriptions operate at the organization/tenant level as flat tier entitlements without seat limits or per-user license checks. Academic user quotas (`maxStudents`, `maxFaculty`) exist solely as unvalidated JSON metadata on `Plan.features` and mock UI interfaces.

---

## 2. Directory Layout & Document Navigation

```
docs/functional-spec/module-03-billing/
├── README.md               # Executive summary, forensic findings, and navigational guide (this file)
└── SPECIFICATION.md        # Complete 17-Section (A–Q) legacy functional reconstruction
```

- **Detailed Functional Specification:** [`SPECIFICATION.md`](./SPECIFICATION.md)
- **Authoring Standard:** [`FUNCTIONAL-SPECIFICATION-STANDARD.md`](../../FUNCTIONAL-SPECIFICATION-STANDARD.md)
- **Master Index & Registry:** [`INDEX.md`](../INDEX.md)

---

## 3. Key Forensic Discoveries & Architecture Summary

| Aspect | Legacy Source Code Implementation Reality |
|---|---|
| **Database Schema** | `billing` schema (`prisma/billing/schema.prisma`) with 7 models: `Plan`, `Subscription`, `Invoice`, `Payment`, `InvoiceSequence`, `Payout`, `WebhookEvent`. |
| **Seat Management** | **NON-EXISTENT.** No `Seat`, `License`, or `SeatAllocation` models exist. Student/faculty user creation is unbounded. |
| **Payment Gateway** | Razorpay SDK integration (`RazorpayClient`) supporting Order checkout, Subscription checkout, payment capture, and automated refunds, with fallback stub mode. |
| **Webhook Processing** | `POST /webhooks/razorpay` verifies HMAC SHA-256 signatures and processes `payment.captured`, `subscription.charged`, and `payment.failed`. |
| **Subscription Lifecycle** | 7-State machine: `pending` $\to$ `trial` $\to$ `active` $\rightleftarrows$ `past_due` $\to$ `suspended` $\to$ `expired` / `cancelled`. |
| **Automated Billing Cron** | `SubscriptionBillingCycleService` (01:00 AM daily renewal), `DunningService` (09:00 AM daily overdue suspension/expiry), `RazorpayReconcilerService` (01:30 AM daily discrepancy check), `CollegePayoutService` (06:00 AM 7th monthly 80/20 revenue share). |
| **Sequential Invoicing** | Atomic PostgreSQL sequence allocation (`billing.invoice_sequences`) generating GST-compliant fiscal-year invoices (`INV-YYYY-YYYY-NNNNN`). |
| **Frontend Implementation** | Complete 8-page/component billing suite in React + Vite (`/billing`, `/billing/plans`, `/billing/subscription/:id`, `/billing/invoices`, `/billing/payments`, `/admin/v1/billing`). |

---

## 4. Primary Source Code References

### Backend
- **Prisma Schema:** [`prisma/billing/schema.prisma`](file:///E:/Dezolver%20project/dezlovers-main/Final%20product/dezlovers-main(cuurent%20update)/prisma/billing/schema.prisma)
- **Billing Module:** [`src/billing/billing.module.ts`](file:///E:/Dezolver%20project/dezlovers-main/Final%20product/dezlovers-main(cuurent%20update)/src/billing/billing.module.ts)
- **Controllers:**
  - [`src/billing/controllers/plans.controller.ts`](file:///E:/Dezolver%20project/dezlovers-main/Final%20product/dezlovers-main(cuurent%20update)/src/billing/controllers/plans.controller.ts)
  - [`src/billing/controllers/subscriptions.controller.ts`](file:///E:/Dezolver%20project/dezlovers-main/Final%20product/dezlovers-main(cuurent%20update)/src/billing/controllers/subscriptions.controller.ts)
  - [`src/billing/controllers/invoices.controller.ts`](file:///E:/Dezolver%20project/dezlovers-main/Final%20product/dezlovers-main(cuurent%20update)/src/billing/controllers/invoices.controller.ts)
  - [`src/billing/controllers/payments.controller.ts`](file:///E:/Dezolver%20project/dezlovers-main/Final%20product/dezlovers-main(cuurent%20update)/src/billing/controllers/payments.controller.ts)
  - [`src/billing/controllers/event-orders.controller.ts`](file:///E:/Dezolver%20project/dezlovers-main/Final%20product/dezlovers-main(cuurent%20update)/src/billing/controllers/event-orders.controller.ts)
  - [`src/billing/controllers/admin-billing.controller.ts`](file:///E:/Dezolver%20project/dezlovers-main/Final%20product/dezlovers-main(cuurent%20update)/src/billing/controllers/admin-billing.controller.ts)
  - [`src/billing/controllers/webhooks.controller.ts`](file:///E:/Dezolver%20project/dezlovers-main/Final%20product/dezlovers-main(cuurent%20update)/src/billing/controllers/webhooks.controller.ts)
- **Services:**
  - [`src/billing/services/subscription.service.ts`](file:///E:/Dezolver%20project/dezlovers-main/Final%20product/dezlovers-main(cuurent%20update)/src/billing/services/subscription.service.ts)
  - [`src/billing/services/webhook.service.ts`](file:///E:/Dezolver%20project/dezlovers-main/Final%20product/dezlovers-main(cuurent%20update)/src/billing/services/webhook.service.ts)
  - [`src/billing/services/dunning.service.ts`](file:///E:/Dezolver%20project/dezlovers-main/Final%20product/dezlovers-main(cuurent%20update)/src/billing/services/dunning.service.ts)
  - [`src/billing/services/subscription-billing-cycle.service.ts`](file:///E:/Dezolver%20project/dezlovers-main/Final%20product/dezlovers-main(cuurent%20update)/src/billing/services/subscription-billing-cycle.service.ts)
  - [`src/billing/services/invoice-generator.service.ts`](file:///E:/Dezolver%20project/dezlovers-main/Final%20product/dezlovers-main(cuurent%20update)/src/billing/services/invoice-generator.service.ts)
  - [`src/billing/services/razorpay-reconciler.service.ts`](file:///E:/Dezolver%20project/dezlovers-main/Final%20product/dezlovers-main(cuurent%20update)/src/billing/services/razorpay-reconciler.service.ts)
  - [`src/billing/services/college-payout.service.ts`](file:///E:/Dezolver%20project/dezlovers-main/Final%20product/dezlovers-main(cuurent%20update)/src/billing/services/college-payout.service.ts)
- **Rate Limit Guard:** [`src/shared/rate-limit/per-tenant-rate-limit.guard.ts`](file:///E:/Dezolver%20project/dezlovers-main/Final%20product/dezlovers-main(cuurent%20update)/src/shared/rate-limit/per-tenant-rate-limit.guard.ts)

### Frontend
- **Billing API Services:** [`frontend/src/services/api/billing.service.ts`](file:///E:/Dezolver%20project/dezlovers-main/Final%20product/frontend/src/services/api/billing.service.ts), [`invoice.service.ts`](file:///E:/Dezolver%20project/dezlovers-main/Final%20product/frontend/src/services/api/invoice.service.ts), [`payment.service.ts`](file:///E:/Dezolver%20project/dezlovers-main/Final%20product/frontend/src/services/api/payment.service.ts)
- **Pages:** [`frontend/src/pages/billing/`](file:///E:/Dezolver%20project/dezlovers-main/Final%20product/frontend/src/pages/billing/)
- **Components:** [`frontend/src/components/billing/`](file:///E:/Dezolver%20project/dezlovers-main/Final%20product/frontend/src/components/billing/)
- **Checkout Modal:** [`frontend/src/components/billing/RazorpayCheckout.tsx`](file:///E:/Dezolver%20project/dezlovers-main/Final%20product/frontend/src/components/billing/RazorpayCheckout.tsx)

