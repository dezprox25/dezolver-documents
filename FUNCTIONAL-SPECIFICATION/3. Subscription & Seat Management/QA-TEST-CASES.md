# Module 3 — Subscription & Seat Management
## Detailed QA Test Cases

**Document ID:** DEZ-SUB-QA-003  
**Version:** 0.2 — Adopted SaaS subscription and seat baseline  
**Status:** Aligned to adopted SUB-D01–SUB-D40; engineering prerequisites and execution evidence pending  
**Specification:** [Module 3 README](./README.md)

| Test ID | Scenario | Preconditions / data | Steps | Expected result | Trace |
|---|---|---|---|---|---|
| SUB-QA-001 | SUPER_ADMIN views plans | SUPER_ADMIN session; active/inactive Plan versions exist | Open Plan Management | SUPER_ADMIN sees global catalogue, version history and active/published state. | SUB-F001 |
| SUB-QA-002 | SUPER_ADMIN creates Plan | SUPER_ADMIN; valid price/currency/duration/included seats/features | Create and publish a Plan | Save as a new validated version; publish makes it selectable for new purchases. | SUB-F001, D02 |
| SUB-QA-003 | Finance Team attempts Plan edit | FINANCE_TEAM; Plan exists | Attempt to edit global Plan price/features | Denied; Finance Team cannot modify global Plan catalogue. | SUB-F001, D26 |
| SUB-QA-004 | Invalid Plan price/duration | SUPER_ADMIN; negative price, invalid duration or seat count | Submit Plan changes | Reject invalid values; no Plan version is published. | SUB-F001, D02 |
| SUB-QA-005 | Deactivate Plan with existing subscriptions | Plan has active/historical subscribers | SUPER_ADMIN deactivates Plan | Unavailable for new purchase; existing subscriptions retain accepted Plan snapshot. | SUB-F001, D16 |
| SUB-QA-006 | Concurrent Plan edit | Two SUPER_ADMIN sessions edit same Plan version | Save both edits with same version | One update succeeds; stale update receives conflict. No silent lost update. | SUB-F001, D40 |
| SUB-QA-007 | Select DEFAULT plan | Finance assigned College; published DEFAULT plans exist | Select package | Only active published DEFAULT packages are selectable. | SUB-F002, D03 |
| SUB-QA-008 | Select CUSTOM plan | Finance assigned College needs custom terms | Request custom package | Create College-scoped quote request; SUPER_ADMIN prepares and approves before acceptance. | SUB-F002, D03/D04 |
| SUB-QA-009 | Unauthorized custom-plan selection | MANAGER/STUDENT session | Attempt custom quote/purchase action | Denied; no quote or subscription created. | SUB-F002, D26 |
| SUB-QA-010 | Plan type/record mismatch | SUPER_ADMIN; invalid category/client-supplied price | Submit inconsistent category/price | Reject invalid category; price comes from Plan or approved quote, never untrusted client input. | SUB-F002, D03 |
| SUB-QA-011 | SUPER_ADMIN views subscription | College subscription exists | Open subscription page | Global permitted details returned | SUB-F003 |
| SUB-QA-012 | Finance Team views assigned subscription | Finance assigned College A | Open Billing & Subscription | Only College A current subscription and history returned. | SUB-F003, RBAC |
| SUB-QA-013 | Finance Team direct-ID foreign subscription | Finance A; subscription belongs to B | Request B by ID | Return 404; no foreign billing metadata disclosed. | SUB-F003, D39 |
| SUB-QA-014 | HOD views subscription fields | HOD assigned Department A | Open Department context | Show Department seat allocation/usage summary only; hide prices, quotes, invoices and payments. | SUB-F003, D18 |
| SUB-QA-015 | Multiple subscriptions per College | College has current and historical Subscription records | Open history | Show one current effective subscription and retained history; do not aggregate old seats into current capacity. | SUB-F003, D05 |
| SUB-QA-016 | Create subscription for valid College/Plan | College and active Plan; Finance assigned College | Select package and start purchase | Create PENDING Subscription with accepted Plan/price/seat/feature snapshot. | SUB-F004, D11 |
| SUB-QA-017 | Create subscription with missing College | Invalid collegeId | Submit | Reject; no record created | SUB-F004 |
| SUB-QA-018 | Create subscription with missing Plan | Invalid planId | Submit | Reject; no record created | SUB-F004 |
| SUB-QA-019 | Finance Team creates subscription outside College | Finance A; target College B | Attempt purchase for B | Return 404; no cross-College Subscription created. | SUB-F004, D26 |
| SUB-QA-020 | Initial subscription activation order | Valid College/Plan and payment flow | Create purchase; inspect before/after trusted confirmation | Subscription is PENDING until trusted payment success or SUPER_ADMIN-confirmed bank transfer, then ACTIVE. | SUB-F004, D11/D30 |
| SUB-QA-021 | Finance Team updates permitted subscription field | Finance assigned College; billing contact correction needed | Update permitted contact fields | Allow approved billing-contact update; direct changes to price, seats, period or payment-derived status are denied. | SUB-F005, D26 |
| SUB-QA-022 | Finance Team updates foreign subscription | Finance A; College B subscription | Update by ID | Denied; no mutation | SUB-F005 |
| SUB-QA-023 | Change protected payment-derived field | Finance tries direct status/payment/date mutation | Submit API mutation | Reject; fields change only through verified billing/state-machine action or explicit SUPER_ADMIN migration. | SUB-F005, D26 |
| SUB-QA-024 | Invalid subscription date order | Period rules approved | Set end before start | Reject | SUB-F005, D10 |
| SUB-QA-025 | Concurrent subscription update | Two sessions update same version | Save both | Stale update conflicts; no silent loss of paid terms. | SUB-F005, D40 |
| SUB-QA-026 | Valid subscription status transition | PENDING subscription; trusted payment success | Process confirmation | Transition PENDING to ACTIVE once; set contracted period and audit. | SUB-F006, D11/D30 |
| SUB-QA-027 | Invalid status transition | Matrix approved | Request forbidden transition | Reject; unchanged | SUB-F006 |
| SUB-QA-028 | Finance Team unauthorized status change | FINANCE_TEAM session | Force status value | Deny direct mutation; Finance may request renewal/cancellation flows only. | SUB-F006, D26 |
| SUB-QA-029 | Status transition reason requirement | Reason adopted policy and required configuration are present | Submit without required reason | Reject if required | SUB-F006 |
| SUB-QA-030 | Subscription state/College state conflict | College SUSPENDED; Subscription ACTIVE | Evaluate access | Module 2 College status independently blocks College access; Subscription status is not rewritten. | SUB-F006, D15 |
| SUB-QA-031 | View subscription dates | Dates populated | Open details | Stored period and renewal dates shown | SUB-F007 |
| SUB-QA-032 | Date/timezone rendering | UTC period stored; timezone configured | View period dates | Store UTC; display in configured billing timezone, initial default Asia/Kolkata. | SUB-F007, D10 |
| SUB-QA-033 | Missing/invalid renewal date | Date absent or inconsistent | View/renew | Behavior follows approved date policy; no silent date calculation | SUB-F007 |
| SUB-QA-034 | Manual renewal | Active subscription nearing end | Finance initiates renewal | Create renewal invoice; manual renewal is v1 default. | SUB-F008, D12 |
| SUB-QA-035 | Auto renewal | Auto-renew disabled by default | Run scheduled renewal without consent/mandate | No automatic charge; enable only with SUPER_ADMIN configuration, College consent and supported mandate. | SUB-F008, D12 |
| SUB-QA-036 | Renewal payment succeeds | Renewal invoice OPEN; trusted payment confirmation | Process success | Mark invoice PAID and extend period once from existing period-end boundary. | SUB-F008, D10/D30 |
| SUB-QA-037 | Renewal payment fails | Renewal unpaid at period boundary | Process failure/timeout | Do not extend period; enter configured GRACE_PERIOD and notify Finance Team. | SUB-F008, D13 |
| SUB-QA-038 | Renewal request repeated | Same subscription and renewal period | Submit same renewal twice | Idempotency returns existing renewal attempt; no duplicate charge or extension. | SUB-F008, D34 |
| SUB-QA-039 | Renewal races with cancellation | Concurrent actions | Start renewal and cancel | Conflict resolution follows approved state model | SUB-F008, D40 |
| SUB-QA-040 | Enter grace period | Renewal unpaid; configured grace policy exists | Process period boundary | Set GRACE_PERIOD and gracePeriodEndsAt using configured calendar-day duration. | SUB-F009, D13 |
| SUB-QA-041 | Grace-period access | Subscription in grace | Use existing features and attempt new seat/add-on | Existing contracted features remain; new seats and entitlement expansion are blocked. | SUB-F009, D13/D25 |
| SUB-QA-042 | Grace period expires | Grace end reached; no successful payment | Process expiry | Transition to SUSPENDED; paid features blocked, billing access retained. | SUB-F009, D13/D25 |
| SUB-QA-043 | Repeated grace extension | Subscription already in grace | Attempt automatic extension | No automatic repeated extension; SUPER_ADMIN override requires reason and audit. | SUB-F009, D13/D37 |
| SUB-QA-044 | Suspend subscription | Subscription past grace or admin action | Process suspension | Set SUSPENDED through allowed transition; preserve history and restrict paid features. | SUB-F010, D14/D25 |
| SUB-QA-045 | Cancel immediately vs end period | Finance requests cancellation during active term | Submit cancellation | Default schedules period-end cancellation; immediate cancellation is SUPER_ADMIN-only with reason; refund separate. | SUB-F010, D14 |
| SUB-QA-046 | Unauthorized cancellation | MANAGER/HOD or foreign Finance | Attempt cancellation | Deny or 404 for foreign College; no state change. | SUB-F010, D26/D39 |
| SUB-QA-047 | Resume/reverse suspension | Reversal adopted policy and required configuration are present | Resume | State follows approved transition; no unintended billing | SUB-F010 |
| SUB-QA-048 | Create quote for valid onboarding/Plan | SUPER_ADMIN; onboarding or existing College and Plan | Create custom quote | Link quote to onboarding or existing College; accepted terms become immutable snapshot. | SUB-F011, D04/D06 |
| SUB-QA-049 | Quote for foreign College scope | Finance A requests quote for B | Submit quote request | Return 404; Finance cannot create/view foreign quote. | SUB-F011, D26/D39 |
| SUB-QA-050 | Quote expiry | validUntil passed | Attempt acceptance | Reject as EXPIRED; no billing/subscription side effect. | SUB-F011, D06 |
| SUB-QA-051 | Quote revision | Existing quote needs revision | SUPER_ADMIN revises | Create a new quote version; preserve prior sent/accepted terms. | SUB-F011, D04/D06 |
| SUB-QA-052 | Quote invalid currency | Unsupported currency | Submit | Reject according to currency policy | SUB-F011/D08 |
| SUB-QA-053 | Quote status transition | Quote status model approved | Request invalid status | Reject; no invalid state | SUB-F011, D07 |
| SUB-QA-054 | Calculate price under approved formula | Package plus add-ons, discount and tax configured | Calculate quote | Total = subtotal - discount + configured tax on taxable discounted amount. | SUB-F012, D08 |
| SUB-QA-055 | Discount boundary | SUPER_ADMIN applies discount | Try negative or > subtotal discount | Reject; valid discount is bounded by subtotal and stored as monetary snapshot. | SUB-F012, D08 |
| SUB-QA-056 | Tax calculation order | Tax rules and jurisdiction configured | Calculate with discount | Apply configured tax after discount; persist tax breakdown; no hardcoded legacy 18%. | SUB-F012, D08 |
| SUB-QA-057 | Currency rounding | Currency precision approved | Calculate fractional amount | Rounding follows approved currency rules | SUB-F012 |
| SUB-QA-058 | Buy More Seats additional quantity | Active subscription; Finance scoped | Enter additional quantity | Quantity is additive to current purchasedSeats; rate/pack from active Plan version. | SUB-F013, D20/D22 |
| SUB-QA-059 | Buy More Seats with inactive subscription | Subscription inactive | Purchase | Eligibility follows approved policy | SUB-F013 |
| SUB-QA-060 | Seat purchase payment success | Seat add-on invoice OPEN | Verify payment | Increase purchasedSeats and College.totalSeats only after verified payment; period end unchanged. | SUB-F013, D22/D30 |
| SUB-QA-061 | Seat purchase payment failure | Add-on payment fails/pending | Process result | Do not increase capacity; safe retry allowed. | SUB-F013, D22/D31 |
| SUB-QA-062 | Seat purchase duplicate request | Same idempotency context | Submit twice | One logical purchase/charge/capacity change only. | SUB-F013, D34 |
| SUB-QA-063 | Seat purchase foreign College | Finance A; target College B | Purchase seats | Denied | SUB-F013 |
| SUB-QA-064 | Seat purchase concurrency | Two Finance users purchase concurrently | Submit at capacity boundary | Serialize capacity updates; no oversell; stale request conflicts. | SUB-F013, D40 |
| SUB-QA-065 | View College/Subscription seat values | Plan and subscription seats exist | Open seat panel | Show included seats, purchased capacity, College used/total and Department allocations/usage/unallocated seats. | SUB-F014, D20/D21 |
| SUB-QA-066 | Seat counter reconciliation | Ledger and counters intentionally differ | Run reconciliation | Detect mismatch and reconcile from auditable seat ledger. | SUB-F014, D20/D34 |
| SUB-QA-067 | Finance Team seat view scope | College A/B | Query seats | Only assigned College data | SUB-F014 |
| SUB-QA-068 | Multiple subscription seat totals | Multiple subscriptions allowed/disallowed decision | View aggregate | Result follows approved canonical capacity model | SUB-F014, D04/D20 |
| SUB-QA-069 | Allocate Department seats | Finance assigned College; free capacity exists | Set Department allocation | Save within College capacity; remainder stays unallocated at College level. | SUB-F015, D21 |
| SUB-QA-070 | Allocate above College capacity | Requested allocation exceeds College total | Submit allocation | Reject; Department allocation sum cannot exceed College capacity. | SUB-F015, D21 |
| SUB-QA-071 | Reduce allocation below current usage | Department usage exceeds proposed allocation | Reduce allocation | Reject; no student is silently removed or moved. | SUB-F015, D21/D23 |
| SUB-QA-072 | Finance/HOD allocation permission | Finance and HOD assigned Department | Finance updates; HOD attempts update | Finance may allocate within College; HOD has read-only Department seat summary. | SUB-F015, D21/D26 |
| SUB-QA-073 | Seat usage event increments | Student membership pending then active | Activate membership | Consume one College seat and one Department used seat when membership becomes ACTIVE. | SUB-F016, D19/D20 |
| SUB-QA-074 | Duplicate seat usage event | Same activation event replayed | Process twice | One active User+College seat assignment only; no double count. | SUB-F016, D19/D34 |
| SUB-QA-075 | User/Student deactivation seat release | Active Student has seat assignment | Deactivate/archive membership | Release seat exactly once; preserve ledger history. | SUB-F016, D19/D38 |
| SUB-QA-076 | Student transfers Department/College | Active seat assignment exists | Transfer within and across Colleges | Same-College transfer moves Department usage only; cross-College transfer consumes destination and releases source transactionally. | SUB-F016, D19 |
| SUB-QA-077 | Concurrent seat usage | Multiple activations at capacity boundary | Process in parallel | Transaction/locking prevents usedSeats exceeding capacity or Department allocation. | SUB-F016, D24/D40 |
| SUB-QA-078 | Reconciliation detects seat drift | Counter mismatch | Run reconciliation | Drift detected; correction follows approved policy | SUB-F016 |
| SUB-QA-079 | Generate invoice for approved billing event | Purchase/renewal/custom/add-on event | Trigger invoice | Create one immutable OPEN invoice with line/tax snapshot and due date. | SUB-F017, D17/D35 |
| SUB-QA-080 | Invoice duplicate generation | Same billing event retried | Trigger twice | Return original invoice; no duplicate number or obligation. | SUB-F017, D34/D35 |
| SUB-QA-081 | Invoice number uniqueness | Concurrent invoice generation | Generate invoices | Atomic configured fiscal-year series produces unique non-reused numbers. | SUB-F017, D35 |
| SUB-QA-082 | Invoice PDF generation failure | Invoice exists; PDF worker fails | Generate/retry PDF | Invoice remains valid; PDF generation is retryable; no false download success. | SUB-F017, D36 |
| SUB-QA-083 | Invoice void/correction | Invoice error identified | SUPER_ADMIN voids and replaces | Original retained as VOID; replacement links to it with audit history. | SUB-F017, D27/D38 |
| SUB-QA-084 | Invoice status transition | Invoice state model approved | Attempt invalid transition | Reject; status unchanged | SUB-F017, Module 17 |
| SUB-QA-085 | Finance Team views invoice history | Finance A | Open Invoice History | Only College A invoices returned. | SUB-F018, D18 |
| SUB-QA-086 | Foreign invoice direct access | Finance A; invoice B | Request invoice ID | Return 404; no invoice data disclosed. | SUB-F018, D18/D39 |
| SUB-QA-087 | Invoice PDF link access | Foreign or expired PDF link | Open link | Require authorized College scope and short-lived signature; deny foreign/expired access. | SUB-F018, D36 |
| SUB-QA-088 | Student/HOD invoice view discrepancy | Student/HOD session | Request invoice | Deny invoice access; HOD sees Department seat summary only; Student has no Module 3 billing page in v1. | SUB-F018, D18 |
| SUB-QA-089 | Initiate payment via approved gateway | Finance A; enabled gateway | Start payment | Create PENDING transaction using server-validated invoice amount/currency. | SUB-F019, D28/D29 |
| SUB-QA-090 | Gateway enum not enabled | Gateway listed but disabled/unconfigured | Select gateway | Reject; only SUPER_ADMIN-enabled/configured methods selectable. | SUB-F019, D29 |
| SUB-QA-091 | Manual bank transfer | Manual transfer enabled | Finance submits reference/evidence | Create PENDING transaction; require SUPER_ADMIN verification before paid state. | SUB-F019, D28/D30 |
| SUB-QA-092 | Payment amount mismatch | Transaction amount/currency differs | Process transaction | Reject/quarantine for reconciliation; do not mark invoice paid or activate subscription. | SUB-F019, D33/D34 |
| SUB-QA-093 | Sensitive payload storage | Provider payload includes secret/card data | Process and inspect records | Store allowlisted metadata only; never store card credentials, secrets or unnecessary sensitive data. | SUB-F019, D30 |
| SUB-QA-094 | Duplicate idempotency key | Same payment request retried | Initiate twice | Return one logical payment attempt; no duplicate charge. | SUB-F019, D34 |
| SUB-QA-095 | Gateway timeout | Provider timeout | Initiate and reconcile | Keep PENDING/unknown until verified; do not infer success/failure from timeout alone. | SUB-F019, D31/D34 |
| SUB-QA-096 | Verified payment success | Valid signed confirmation and matching amount | Process event | Mark transaction SUCCESS and invoice PAID; activate/extend subscription or seats exactly once. | SUB-F020, D30/D34 |
| SUB-QA-097 | Unverified client-side success | Browser says success; no provider confirmation | Process browser return | Do not mark payment SUCCESS from browser return alone. | SUB-F020, D30 |
| SUB-QA-098 | Duplicate payment callback | Same verified webhook replay | Process twice | Unique event/idempotency yields one financial and entitlement effect. | SUB-F020, D30/D34 |
| SUB-QA-099 | Invalid callback signature | Webhook secret missing or signature invalid | Submit webhook | Fail closed; no financial state change; record security event. | SUB-F020, D30 |
| SUB-QA-100 | Out-of-order payment callback | Delayed success/failure events | Process and reconcile | Do not downgrade a confirmed success due to older failure; reconcile provider state and audit. | SUB-F020, D34 |
| SUB-QA-101 | Payment success but downstream update fails | Provider success; invoice/subscription write failure | Process and retry reconciliation | Keep provider success traceable; retry downstream safely without duplicate charge/activation. | SUB-F020, D34 |
| SUB-QA-102 | Payment references valid Invoice/Subscription | Same College/currency references | Create transaction | All required references exist and belong to same College/currency. | SUB-F021, D33 |
| SUB-QA-103 | Cross-College billing reference | Payment A; invoice B | Create transaction | Reject before write; no cross-College link. | SUB-F021, D33 |
| SUB-QA-104 | Partial/missing related IDs | Initial, renewal and add-on examples | Create each type | Enforce type-specific references; PENDING initial subscription may be linked before settlement. | SUB-F021, D33 |
| SUB-QA-105 | SUPER_ADMIN global billing access | Multiple Colleges | Query billing records | Global access permitted | SUB-F022 |
| SUB-QA-106 | Finance Team query scope | Finance assigned College A | List billing and seat data | Only College A records returned. | SUB-F022, D26 |
| SUB-QA-107 | Finance Team request foreign payment ID | Finance A; payment B | Request by ID | Return 404; no payment/gateway data disclosed. | SUB-F022, D39 |
| SUB-QA-108 | Manager billing access | MANAGER session | Request billing resource | Deny; Manager has no Module 3 billing permission. | SUB-F022, D18 |
| SUB-QA-109 | Scope tampering in billing search/export | Finance/HOD alters College ID/filter | Search/export | Backend derives scope from membership/resource; client filter cannot expand access. | SUB-F022, D26/D39 |
| SUB-QA-110 | ACTIVE subscription entitlement | College ACTIVE; Subscription ACTIVE/in-period | Request contracted feature | Allow Plan/quote entitlements within feature and seat limits. | SUB-F023, D25 |
| SUB-QA-111 | GRACE_PERIOD entitlement | Subscription in grace | Use existing feature; attempt new seat/expansion | Existing entitlements remain; deny new seat activation/add-ons/feature expansion. | SUB-F023, D13/D25 |
| SUB-QA-112 | SUSPENDED subscription entitlement | Subscription SUSPENDED | Request academic feature and billing portal | Block paid academic features; retain limited Finance billing settlement/support. | SUB-F023, D25 |
| SUB-QA-113 | CANCELLED subscription entitlement | Subscription CANCELLED at effective time | Request feature/history | Block paid academic features; retain authorized financial history. | SUB-F023, D14/D25 |
| SUB-QA-114 | Seat limit enforcement | College/Department at capacity | Activate another Student; attempt override | Block normal activation; only reasoned audited SUPER_ADMIN override; existing students remain active. | SUB-F023, D19/D24 |
| SUB-QA-115 | Audit subscription change | Plan/subscription term or status changes | Perform action | Record actor, College, target, changed fields, effective time and reason per Module 15. | SUB-F024, D37 |
| SUB-QA-116 | Audit payment/refund action | Payment/refund operation | Process action | Audit initiation/confirmation/refund amount/reason without credential secrets. | SUB-F024, D32/D37 |
| SUB-QA-117 | Audit write failure | Audit service unavailable; critical financial event | Execute action | Preserve financial operation and ensure durable audit retry/outbox recovery per Module 18; surface operational failure. | SUB-F024, D37 |

## Additional Super Admin configurability and contract tests

| Test ID | Scenario | Preconditions / data | Steps | Expected result | Trace |
|---|---|---|---|---|---|
| SUB-QA-118 | Plan edit applies to future purchases | Plan has active subscribers on version 1 | SUPER_ADMIN publishes version 2 | New purchases use version 2; existing subscriptions retain version 1 until renewal or explicit migration. | SUB-F001, D02/D16 |
| SUB-QA-119 | Explicitly migrate existing subscription | Active subscription on old Plan version | SUPER_ADMIN previews/confirms migration with effective date/reason | Only selected subscription changes; prior terms remain traceable and audited. | SUB-F005, D16/D40 |
| SUB-QA-120 | Configure add-on seat price/proration | SUPER_ADMIN edits add-on settings | Change unit/pack price and proration | New purchase uses new setting; accepted quotes/purchases remain unchanged. | SUB-F001/F013, D02/D22 |
| SUB-QA-121 | Custom quote for existing College | Finance assigned College requests custom terms | SUPER_ADMIN creates/sends quote | Quote links directly to existing College, not only onboarding; Finance sees only own quote. | SUB-F011, D04/D06 |
| SUB-QA-122 | Custom quote cannot bypass entitlement rules | SUPER_ADMIN enters unsupported feature/seat values | Attempt approval | Reject unsupported feature keys/limits; only supported entitlements can be quoted. | SUB-F011/F012, D07 |
| SUB-QA-123 | Tax settings are editable and snapshotted | SUPER_ADMIN changes tax rule; old quote exists | Create new quote and inspect old quote | New quote uses new configured rule; prior quote/invoice retains original tax snapshot. | SUB-F012/F017, D08 |
| SUB-QA-124 | Add-on seats share subscription end date | Active subscription mid-period | Purchase additional seats | Add-on capacity becomes effective after payment and ends with underlying subscription; no separate renewal date. | SUB-F013, D22 |
| SUB-QA-125 | Staff account does not consume Student seat | Staff membership activated | Activate staff role | College/Department student seat usage does not increase. | SUB-F016, D19 |
| SUB-QA-126 | Student consumes one seat per College | User has multiple academic links in one College | Activate additional links | Only one active User+College seat assignment is counted. | SUB-F016, D19 |
| SUB-QA-127 | Same-College Department transfer | Student has active seat in Department A | Transfer to Department B | Department A usage decreases and B increases; College usedSeats unchanged. | SUB-F016, D19/D20 |
| SUB-QA-128 | Manual transfer remains pending | Manual transfer enabled | Finance submits reference/evidence | Transaction remains PENDING; invoice/subscription not activated until SUPER_ADMIN verifies. | SUB-F019/F020, D28/D30 |
| SUB-QA-129 | SUPER_ADMIN verifies bank transfer | Pending transfer with evidence | SUPER_ADMIN confirms amount/reference | Payment becomes SUCCESS once; related invoice/subscription effect is idempotent and audited. | SUB-F020, D28/D30 |
| SUB-QA-130 | Partial refund is recorded independently | Successful payment; SUPER_ADMIN authorized | Initiate partial refund with reason | Separate refund record created; totals reconcile; subscription is not automatically reactivated/cancelled. | SUB-F020/F024, D32 |
| SUB-QA-131 | Public storage URL cannot bypass invoice scope | Invoice PDF exists; foreign College user has raw URL | Attempt direct access | Storage remains private; require authorized short-lived signed URL and same-College check. | SUB-F018, D36 |
| SUB-QA-132 | Disabled Plan remains visible in history | Historical subscription uses deactivated Plan | Open catalogue and subscription history | Disabled Plan cannot be newly purchased; historical contract shows exact accepted version/terms. | SUB-F001/F003, D16/D38 |

## QA execution note

Cases blocked by open pricing, permission, lifecycle, gateway, seat-counting, or audit decisions must be marked blocked until the relevant decision is approved. The test suite does not select an option for the product owner.


SUB-D01–SUB-D40 are adopted; tests are no longer blocked waiting for product pricing decisions. Block a test only when its required schema migration, canonical RBAC change, gateway configuration or dependent-module contract is not implemented. QA execution and sign-off have not yet occurred.