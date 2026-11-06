# Module 3 — Subscription & Seat Management
## Detailed QA Test Cases

**Document ID:** DEZ-SUB-QA-003  
**Version:** 0.1 — Detailed QA draft  
**Status:** Draft; decisions must be resolved before tests are treated as implementation acceptance  
**Specification:** [Module 3 README](./README.md)

| Test ID | Scenario | Preconditions / data | Steps | Expected result | Trace |
|---|---|---|---|---|---|
| SUB-QA-001 | SUPER_ADMIN views plans | Plan records exist | Open Plan Management | Plans visible globally | SUB-F001 |
| SUB-QA-002 | SUPER_ADMIN creates Plan | Plan field policy approved | Submit valid Plan | Plan saved under approved constraints | SUB-F001, D01/D03 |
| SUB-QA-003 | Finance Team attempts Plan edit | Finance role | Edit Plan | Denied unless permission explicitly approved | SUB-F001, D29 |
| SUB-QA-004 | Invalid Plan price/duration | Validation rules approved | Submit invalid values | Validation error; no invalid Plan saved | SUB-F001 |
| SUB-QA-005 | Deactivate Plan with existing subscriptions | Existing subscriber | Deactivate Plan | Existing subscription impact follows approved policy; no silent mutation | SUB-F001, D26 |
| SUB-QA-006 | Concurrent Plan edit | Two admin sessions | Save conflicting changes | Conflict/last-write behavior follows approved policy | SUB-F001 |
| SUB-QA-007 | Select DEFAULT plan | DEFAULT semantics approved | Finance selects plan | Selection follows approved plan eligibility | SUB-F002, D02 |
| SUB-QA-008 | Select CUSTOM plan | Custom plan policy approved | Configure/select custom | Custom selection follows approved quote/plan model | SUB-F002, D02/D06 |
| SUB-QA-009 | Unauthorized custom-plan selection | Role not permitted | Attempt custom plan | Denied | SUB-F002 |
| SUB-QA-010 | Plan type/record mismatch | Plan/custom semantics approved | Submit inconsistent category | Reject or normalize per approved model | SUB-F002 |
| SUB-QA-011 | SUPER_ADMIN views subscription | College subscription exists | Open subscription page | Global permitted details returned | SUB-F003 |
| SUB-QA-012 | Finance Team views assigned subscription | Finance College A | Open billing page | Only College A subscription shown | SUB-F003, RBAC |
| SUB-QA-013 | Finance Team direct-ID foreign subscription | Finance A; subscription College B | Request B by ID | Denied/not-found; no B data disclosed | SUB-F003, SUB-F022 |
| SUB-QA-014 | HOD views subscription fields | HOD assigned department | Open allowed view | Only approved HOD fields/scope returned | SUB-F003, D13 |
| SUB-QA-015 | Multiple subscriptions per College | Multiple subscription records | Open College subscription view | Display/selection follows approved cardinality/active rules | SUB-F003, D04 |
| SUB-QA-016 | Create subscription for valid College/Plan | Existing College/Plan; permissions approved | Create subscription | Record references valid College and Plan | SUB-F004 |
| SUB-QA-017 | Create subscription with missing College | Invalid collegeId | Submit | Reject; no record created | SUB-F004 |
| SUB-QA-018 | Create subscription with missing Plan | Invalid planId | Submit | Reject; no record created | SUB-F004 |
| SUB-QA-019 | Finance Team creates subscription outside College | Finance A; target College B | Submit create | Denied | SUB-F004, RBAC |
| SUB-QA-020 | Initial subscription activation order | Activation/payment sequence approved | Execute onboarding subscription flow | State/record order matches approved sequence | SUB-F004, D05 |
| SUB-QA-021 | Finance Team updates permitted subscription field | Finance assigned College; field allowlist approved | Update allowed field | Update succeeds in assigned scope | SUB-F005, D31 |
| SUB-QA-022 | Finance Team updates foreign subscription | Finance A; College B subscription | Update by ID | Denied; no mutation | SUB-F005 |
| SUB-QA-023 | Change protected payment-derived field | Attempt direct status/payment edit | Submit | Denied unless specifically approved | SUB-F005 |
| SUB-QA-024 | Invalid subscription date order | Period rules approved | Set end before start | Reject | SUB-F005, D10 |
| SUB-QA-025 | Concurrent subscription update | Two sessions | Update overlapping fields | Conflict behavior follows approved concurrency policy | SUB-F005, D40 |
| SUB-QA-026 | Valid subscription status transition | Transition matrix approved | Request valid status change | State updated and side effects match approved policy | SUB-F006, Module 17 |
| SUB-QA-027 | Invalid status transition | Matrix approved | Request forbidden transition | Reject; unchanged | SUB-F006 |
| SUB-QA-028 | Finance Team unauthorized status change | Role action permission not granted | Change status | Denied | SUB-F006, D30 |
| SUB-QA-029 | Status transition reason requirement | Reason policy approved | Submit without required reason | Reject if required | SUB-F006 |
| SUB-QA-030 | Subscription state/College state conflict | College suspended; Subscription active | Evaluate state | Precedence follows approved rule; no inferred coupling | SUB-F006, D25 |
| SUB-QA-031 | View subscription dates | Dates populated | Open details | Stored period and renewal dates shown | SUB-F007 |
| SUB-QA-032 | Date/timezone rendering | Display convention approved | View dates across timezone | Render consistently per approved convention | SUB-F007, D10 |
| SUB-QA-033 | Missing/invalid renewal date | Date absent or inconsistent | View/renew | Behavior follows approved date policy; no silent date calculation | SUB-F007 |
| SUB-QA-034 | Manual renewal | Manual renewal approved | Finance starts renewal | Flow follows approved quote/invoice/payment steps | SUB-F008, D09 |
| SUB-QA-035 | Auto renewal | Auto renewal approved | Trigger scheduled renewal | Only approved automatic workflow runs | SUB-F008, D09 |
| SUB-QA-036 | Renewal payment succeeds | Renewal billing approved | Confirm payment | Period/status updated at approved point | SUB-F008 |
| SUB-QA-037 | Renewal payment fails | Payment failure | Run renewal | No unauthorized period extension; failure handling per policy | SUB-F008 |
| SUB-QA-038 | Renewal request repeated | Same subscription/period | Repeat renewal action | No duplicate charge/period | SUB-F008, D36 |
| SUB-QA-039 | Renewal races with cancellation | Concurrent actions | Start renewal and cancel | Conflict resolution follows approved state model | SUB-F008, D40 |
| SUB-QA-040 | Enter grace period | Grace trigger approved | Trigger grace transition | Status and gracePeriodEndsAt set per policy | SUB-F009, D11 |
| SUB-QA-041 | Grace-period access | Access policy approved | Access product features in grace | Features match approved policy | SUB-F009, D24 |
| SUB-QA-042 | Grace period expires | Grace end reached | Process expiry | State/access transition follows approved rule | SUB-F009, D11/D24 |
| SUB-QA-043 | Repeated grace extension | Grace state active | Extend grace twice | Extension policy enforced; no unauthorized extension | SUB-F009 |
| SUB-QA-044 | Suspend subscription | Permission/transition approved | Suspend | State and access follow policy | SUB-F010 |
| SUB-QA-045 | Cancel immediately vs end period | Cancellation policy approved | Cancel subscription | Effective date/refund/access match approved option | SUB-F010, D12 |
| SUB-QA-046 | Unauthorized cancellation | Actor lacks permission | Cancel | Denied | SUB-F010, D13 |
| SUB-QA-047 | Resume/reverse suspension | Reversal policy approved | Resume | State follows approved transition; no unintended billing | SUB-F010 |
| SUB-QA-048 | Create quote for valid onboarding/Plan | Quote permissions approved | Create quote | Quote references valid onboarding/Plan | SUB-F011 |
| SUB-QA-049 | Quote for foreign College scope | Finance assigned College A; quote linked to B | Create/view | Denied | SUB-F011 |
| SUB-QA-050 | Quote expiry | validUntil passed | Accept quote | Expired quote rejected if expiry policy approved | SUB-F011, D07 |
| SUB-QA-051 | Quote revision | Existing quote; revision policy approved | Modify quote | New/revised version follows policy; prior record traceable | SUB-F011 |
| SUB-QA-052 | Quote invalid currency | Unsupported currency | Submit | Reject according to currency policy | SUB-F011/D08 |
| SUB-QA-053 | Quote status transition | Quote status model approved | Request invalid status | Reject; no invalid state | SUB-F011, D07 |
| SUB-QA-054 | Calculate price under approved formula | Formula/rounding approved | Calculate quote | Total matches approved formula | SUB-F012, D08 |
| SUB-QA-055 | Discount boundary | Discount policy approved | Apply max/over-max discount | Boundary enforced | SUB-F012 |
| SUB-QA-056 | Tax calculation order | Tax policy approved | Quote with discount/tax | Amount matches approved order and jurisdiction | SUB-F012 |
| SUB-QA-057 | Currency rounding | Currency precision approved | Calculate fractional amount | Rounding follows approved currency rules | SUB-F012 |
| SUB-QA-058 | Buy More Seats additional quantity | Purchase semantics approved | Enter quantity and continue | Quantity treated as add-on or target per decision | SUB-F013, D22 |
| SUB-QA-059 | Buy More Seats with inactive subscription | Subscription inactive | Purchase | Eligibility follows approved policy | SUB-F013 |
| SUB-QA-060 | Seat purchase payment success | Billing workflow approved | Complete payment | Seat capacity increases at approved event | SUB-F013 |
| SUB-QA-061 | Seat purchase payment failure | Payment fails | Attempt purchase | No unapproved capacity increase | SUB-F013 |
| SUB-QA-062 | Seat purchase duplicate request | Same idempotency context | Submit twice | No duplicate purchase/charge | SUB-F013, D36 |
| SUB-QA-063 | Seat purchase foreign College | Finance A; target College B | Purchase seats | Denied | SUB-F013 |
| SUB-QA-064 | Seat purchase concurrency | Two Finance actions simultaneously | Buy seats | No oversell; conflict/reconciliation policy applied | SUB-F013, D40 |
| SUB-QA-065 | View College/Subscription seat values | Seat data exists | Open billing seat panel | Values displayed from correct fields; no unsupported available-seat calculation | SUB-F014 |
| SUB-QA-066 | Seat counter reconciliation | Counters intentionally inconsistent | Reconcile | Mismatch detected/handled per approved policy | SUB-F014, D20 |
| SUB-QA-067 | Finance Team seat view scope | College A/B | Query seats | Only assigned College data | SUB-F014 |
| SUB-QA-068 | Multiple subscription seat totals | Multiple subscriptions allowed/disallowed decision | View aggregate | Result follows approved canonical capacity model | SUB-F014, D04/D20 |
| SUB-QA-069 | Allocate Department seats | Allocation permission/policy approved | Set allocation | Allocation saved under policy | SUB-F015 |
| SUB-QA-070 | Allocate above College capacity | College capacity fixed | Request allocation > capacity | Reject or override only per approved policy | SUB-F015, D21 |
| SUB-QA-071 | Reduce allocation below current usage | Department has more users than new allocation | Reduce seats | Policy enforced; no silent user removal | SUB-F015, D21 |
| SUB-QA-072 | Finance/HOD allocation permission | Permission unresolved | Attempt allocation | Outcome matches approved matrix; no assumed permission | SUB-F015 |
| SUB-QA-073 | Seat usage event increments | Seat trigger approved | Create/activate/enroll event | Counter/derived usage updates once | SUB-F016, D19 |
| SUB-QA-074 | Duplicate seat usage event | Same event replayed | Process twice | No double count | SUB-F016 |
| SUB-QA-075 | User/Student deactivation seat release | Release policy approved | Deactivate user | Usage changes per approved release rule | SUB-F016, D19 |
| SUB-QA-076 | Student transfers Department/College | Existing usage; move event | Transfer | Counts update consistently per approved policy | SUB-F016, D21 |
| SUB-QA-077 | Concurrent seat usage | Multiple enroll/create events | Process in parallel | Capacity integrity follows approved concurrency policy | SUB-F016 |
| SUB-QA-078 | Reconciliation detects seat drift | Counter mismatch | Run reconciliation | Drift detected; correction follows approved policy | SUB-F016 |
| SUB-QA-079 | Generate invoice for approved billing event | Invoice workflow approved | Trigger invoice | Invoice created with valid references | SUB-F017 |
| SUB-QA-080 | Invoice duplicate generation | Same event retried | Trigger twice | No duplicate invoice unless approved | SUB-F017 |
| SUB-QA-081 | Invoice number uniqueness | Numbering policy approved | Generate concurrent invoices | Unique numbering and retry policy enforced | SUB-F017, D17 |
| SUB-QA-082 | Invoice PDF generation failure | Invoice saved; PDF service fails | Generate PDF | No false PDF success; retry/error policy followed | SUB-F017 |
| SUB-QA-083 | Invoice void/correction | Void policy approved | Void/correct invoice | Status and correction trail follow approved policy | SUB-F017 |
| SUB-QA-084 | Invoice status transition | Invoice state model approved | Attempt invalid transition | Reject; status unchanged | SUB-F017, Module 17 |
| SUB-QA-085 | Finance Team views invoice history | Finance College A | Open Invoice History | Only College A invoices shown | SUB-F018 |
| SUB-QA-086 | Foreign invoice direct access | Finance A; invoice College B | Request invoice | Denied | SUB-F018 |
| SUB-QA-087 | Invoice PDF link access | PDF URL known from another College | Open URL | Auth/scope validation denies foreign access | SUB-F018 |
| SUB-QA-088 | Student/HOD invoice view discrepancy | Role access policy approved | Request invoice | Result follows approved policy; no assumption from RBAC alone | SUB-F018, D18 |
| SUB-QA-089 | Initiate payment via approved gateway | Gateway configured | Start payment | Transaction created/started according to gateway contract | SUB-F019 |
| SUB-QA-090 | Gateway enum not actually enabled | Gateway enum exists but config disabled | Select gateway | Unavailable gateway not offered/accepted | SUB-F019, D14 |
| SUB-QA-091 | Manual bank transfer | Manual transfer process approved | Submit transfer details/evidence | Pending/verification status follows policy | SUB-F019 |
| SUB-QA-092 | Payment amount mismatch | Invoice amount differs from submitted transaction | Submit/process | Reject or hold for reconciliation | SUB-F019 |
| SUB-QA-093 | Sensitive payload storage | Gateway payload contains sensitive data | Process payment | Sensitive data excluded/redacted per approved storage policy | SUB-F019 |
| SUB-QA-094 | Duplicate idempotency key | Same key reused | Initiate payment twice | One logical transaction/effect | SUB-F019, D36 |
| SUB-QA-095 | Gateway timeout | Provider timeout | Initiate payment | Transaction remains in approved pending/failed state; no false success | SUB-F019 |
| SUB-QA-096 | Verified payment success | Valid gateway confirmation | Process success | Transaction status and downstream records updated per policy | SUB-F020 |
| SUB-QA-097 | Unverified client-side success | Browser return says success; no trusted confirmation | Process callback | Payment not marked SUCCESS based only on untrusted return | SUB-F020 |
| SUB-QA-098 | Duplicate payment callback | Same callback replayed | Process twice | Idempotent; no duplicate invoice/subscription/seat effect | SUB-F020 |
| SUB-QA-099 | Invalid callback signature | Gateway signature invalid | Process callback | Reject/ignore; no financial state change | SUB-F020 |
| SUB-QA-100 | Out-of-order payment callback | FAILED then later SUCCESS or vice versa | Process events | State follows approved reconciliation rules | SUB-F020 |
| SUB-QA-101 | Payment success but downstream update fails | Gateway success; invoice/subscription write fails | Process result | Recovery/reconciliation creates no duplicate charge; state remains traceable | SUB-F020, D38 |
| SUB-QA-102 | Payment references valid Invoice/Subscription | Related records in same College | Create transaction | Links are valid and traceable | SUB-F021 |
| SUB-QA-103 | Cross-College billing reference | Payment College A; Invoice B | Create transaction | Reject cross-College link | SUB-F021 |
| SUB-QA-104 | Partial/missing related IDs | Transaction type/reference policy approved | Create transaction | Required references enforced per approved type | SUB-F021, D35 |
| SUB-QA-105 | SUPER_ADMIN global billing access | Multiple Colleges | Query billing records | Global access permitted | SUB-F022 |
| SUB-QA-106 | Finance Team query scope | Finance assigned College A | List subscriptions/invoices/payments | Only College A records | SUB-F022 |
| SUB-QA-107 | Finance Team request foreign payment ID | Finance A; payment College B | Direct ID request | Denied; no payment details exposed | SUB-F022 |
| SUB-QA-108 | Manager billing access | Manager session | Request billing resource | Denied under current matrix | SUB-F022 |
| SUB-QA-109 | Scope tampering in billing search/export | Finance/HOD; altered collegeId/filter | Search/export | Backend applies trusted scope; foreign data excluded | SUB-F022 |
| SUB-QA-110 | ACTIVE subscription entitlement | Access policy approved | Request product capability | Allow/deny matches approved ACTIVE policy | SUB-F023 |
| SUB-QA-111 | GRACE_PERIOD entitlement | Grace policy approved | Request capability | Access matches approved grace policy | SUB-F023 |
| SUB-QA-112 | SUSPENDED subscription entitlement | Suspension policy approved | Request capability | Access matches approved suspension policy | SUB-F023 |
| SUB-QA-113 | CANCELLED subscription entitlement | Cancellation policy approved | Request capability | Access matches approved cancellation policy | SUB-F023 |
| SUB-QA-114 | Seat limit enforcement | Seat limit policy approved | Create/activate/enroll beyond limit | Behavior matches approved seat enforcement rule | SUB-F023 |
| SUB-QA-115 | Audit subscription change | Audit event catalog approved | Change subscription | Actor/target/action/time logged per Module 15 | SUB-F024 |
| SUB-QA-116 | Audit payment/refund action | Payment audit policy approved | Process payment/refund | Audit trail created without sensitive credential data | SUB-F024 |
| SUB-QA-117 | Audit write failure | Audit failure policy approved | Execute sensitive billing action | Operation response follows approved atomicity/failure policy | SUB-F024, Module 18 |

## QA execution note

Cases blocked by open pricing, permission, lifecycle, gateway, seat-counting, or audit decisions must be marked blocked until the relevant decision is approved. The test suite does not select an option for the product owner.
