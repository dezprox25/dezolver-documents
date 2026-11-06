# DEZOLVER — Functional Specification Standard

**Document type:** Functional Specification Authoring Standard  
**Version:** 1.0  
**Status:** Working standard for Modules 1–18  
**Applies to:** All module-level functional specifications in `FUNCTIONAL-SPECIFICATION/`

---

## 1. Purpose

This standard defines the minimum level of detail required for every DEZOLVER functional specification.

A module document must explain the behavior of its functions in enough detail that:

- a product owner can review the intended behavior;
- a designer can understand the required user experience;
- a developer can implement the approved behavior without guessing about business rules;
- a QA engineer can derive test cases and expected results;
- unresolved product choices remain visible and are not silently decided by the author.

A document that only describes a module's purpose, lists screens, or names features is **not** a complete functional specification.

## 2. What a functional specification must answer

For every user-facing or system function, the specification must make clear:

1. Who can perform the function?
2. Where does the function begin?
3. What must be true before it can begin?
4. What does the user see and do?
5. What information is entered, selected, or supplied?
6. What validation and business rules apply?
7. What happens step by step when it succeeds?
8. What data or system state changes?
9. What does the user or calling system receive?
10. What happens when it fails or an alternate condition occurs?
11. What permissions and organizational scope are enforced?
12. How can the behavior be verified through acceptance criteria and QA tests?

If a source does not answer a question, mark it as **TBD / Decision Required**. Do not invent an answer.

## 3. Evidence and requirement labels

Use these labels consistently throughout the documents.

| Label | Meaning | Authoring rule |
|---|---|---|
| **Source-confirmed** | Explicitly stated in a supplied source document or approved requirement. | Cite or identify the source. |
| **Legacy-verified** | Verified directly in the legacy implementation, repository, or forensic report. | Identify the artifact/code/report and describe the observed behavior. |
| **Target requirement** | Intended behavior approved for the DEZOLVER target product. | Record the approval or decision reference where available. |
| **Derived** | A functional conclusion logically drawn from multiple confirmed facts. | Explain the basis; do not present it as a directly stated source fact. |
| **Recommendation** | Suggested product behavior that has not been approved. | Keep separate from approved requirements. |
| **TBD / Decision Required** | Information is missing, conflicting, or requires product/security/technical approval. | Record it in the decision log; do not treat it as implemented or approved. |

### 3.1 Legacy behavior is not automatically target behavior

A legacy implementation finding describes what the old system did. It does not automatically define what the new product should do.

When legacy behavior and target requirements differ:

- document the legacy behavior as evidence;
- document the target requirement separately;
- identify the difference explicitly;
- record a decision if the intended target behavior is not approved.

### 3.2 Recommendations are not requirements

Recommendations must be visibly marked and must not be written as mandatory behavior until approved.

## 4. Required module document structure

Each module specification should use the following structure. Sections that are genuinely not applicable may be marked **Not applicable**, with a short reason.

1. **Document metadata and review status**
2. **Purpose and module outcome**
3. **Scope**
   - In scope
   - Out of scope
4. **Source documents and evidence**
5. **Terminology and definitions**
6. **Actors, roles, permissions, and scope**
7. **Module overview and navigation / entry points**
8. **Functional inventory**
9. **Detailed function specifications**
10. **Business rules and validations**
11. **Data and state reference**
12. **Notifications, audit, and integrations** (where applicable)
13. **Cross-module dependencies**
14. **Edge cases and failure handling**
15. **Open decisions / decision log**
16. **Acceptance criteria and QA coverage**
17. **Review and approval checklist**
18. **Source references**

The module may add sections where needed, but must not omit the detailed function specifications.

## 5. Functional inventory

Before describing each function in detail, provide an inventory of the functions in the module.

Recommended columns:

| Function ID | Function name | Primary actor | Entry point | Function type | Specification status |
|---|---|---|---|---|---|
| MOD-F001 | Example function | Example role | Example page/action | Create / Read / Update / Delete / workflow / system | Draft / Decision required / Approved |

Function IDs must be unique within the module and stable across revisions. Do not renumber a function merely because its wording changes.

Use a module prefix, for example:

- AUTH-F001 for Authentication & User Lifecycle
- ORG-F001 for College / Department / Section Management
- SUB-F001 for Subscription & Seat Management
- COURSE-F001 for Course & Enrollment
- PROB-F001 for Problem Management
- SUBMIT-F001 for Code Submission & Judge0
- TEST-F001 for Test Case Evaluation
- LAB-F001 for Labs & Lab Scoring
- ASMT-F001 for Assessments

## 6. Required function-level specification template

Use this template for **every function**. If a field cannot be determined from the available evidence, state what is unknown and link it to a decision ID.

### [FUNCTION-ID] — [Function name]

**Requirement status:** Source-confirmed / Legacy-verified / Target requirement / Derived / Recommendation / TBD  
**Specification status:** Draft / Decision required / Ready for review / Approved  
**Priority:** Must have / Should have / Could have / TBD (only if priority has been agreed)  
**Primary actor(s):**  
**Supporting/system actor(s):**

#### 6.1 Purpose and outcome
Describe the business purpose and the observable outcome. Avoid vague statements such as “manage users” without describing the result.

#### 6.2 Permissions and data scope
Specify:

- which roles may use the function;
- whether access is read, create, update, delete, publish, assign, export, or another action;
- college / department / section / group / user scope;
- ownership or membership requirements;
- server-side enforcement expectations;
- any exceptions or delegated access.

If the permission matrix is not known, reference the relevant decision. Do not assume that page visibility grants permission.

#### 6.3 Preconditions
List the conditions that must be true before the function can start.

Examples may include authenticated user, active account, correct role and scope, existing record, required subscription status, or prerequisite workflow state. Include only conditions supported by requirements or approved decisions.

#### 6.4 Entry point and trigger
Specify where the function is initiated:

- page or navigation route;
- button, menu, link, row action, or automatic trigger;
- direct URL/API/system event, where relevant;
- source page or workflow from which the actor arrives.

#### 6.5 User interface behavior
Describe the functional UI behavior in sequence:

- page/dialog/drawer/modal that opens;
- information shown to the actor;
- available actions;
- fields, selectors, and controls;
- loading, empty, disabled, and confirmation states;
- navigation after success or cancellation.

Do not invent exact visual styling. UI labels and messages should be specified only when known or approved; otherwise mark them TBD.

#### 6.6 Inputs and field definitions
Use a table for all inputs.

| Field / input | Source | Data type | Required? | Format / allowed values | Default | Validation | Notes |
|---|---|---|---|---|---|---|---|

Include relevant constraints such as length, range, uniqueness, allowed enum values, relationships, and whether the field is editable. Distinguish user-entered fields from values supplied by the system.

#### 6.7 Main flow — step by step
Write the successful path as numbered steps from the actor's first action to the final observable outcome.

Each step should identify, as applicable:

- actor action;
- UI response;
- system/backend action;
- data read or changed;
- next state or screen.

Avoid compressed statements such as “the system validates and saves.” Explain what is validated and what is saved, or reference the specific validation and data rules.

#### 6.8 Business rules
List all rules that affect the outcome. Give each rule a stable identifier where practical.

Examples of rule categories:

- eligibility;
- uniqueness;
- ownership;
- limits and quotas;
- ordering;
- calculations;
- timing and deadlines;
- lifecycle/state restrictions;
- scope inheritance;
- score or status aggregation.

Do not include unapproved recommendations as confirmed rules.

#### 6.9 Validation rules and user feedback
Describe validation at the relevant level:

- client-side validation, if required;
- server-side validation;
- validation timing (on input, on blur, on submit, or after processing);
- invalid condition;
- expected behavior when invalid;
- exact message if approved, otherwise message intent and **TBD** label.

| Validation ID | Condition | Validation point | Expected system behavior | User feedback |
|---|---|---|---|---|

Client-side validation must not replace server-side validation for security or business rules.

#### 6.10 System / backend behavior
Describe functional system behavior without prescribing unnecessary implementation details:

- checks performed;
- records queried or created/updated/deleted;
- calculations or processing;
- state changes;
- interactions with other modules or external services;
- asynchronous behavior and completion status;
- idempotency or duplicate-request behavior when relevant.

Do not guess APIs, database schema, queues, or implementation architecture. If an API contract is needed, reference or create a separate technical specification.

#### 6.11 Data changes and state transitions
Specify:

- entities/records affected;
- fields created, updated, or deleted;
- actor and timestamps recorded;
- before/after state;
- related records or side effects;
- whether the operation is reversible.

If state changes are involved, reference the canonical state machine or define the transition and identify it for inclusion in Module 17.

#### 6.12 Success result
State exactly what happens on success:

- confirmation or result shown;
- destination screen or next workflow step;
- resulting status/data;
- records or outputs available to the user;
- notifications/audit events triggered, if confirmed.

#### 6.13 Alternate flows and failure handling
Document important non-happy paths, including:

- validation failure;
- permission denied;
- record not found;
- stale or conflicting data;
- duplicate action/request;
- expired session;
- external service failure or timeout;
- partial completion;
- cancellation;
- retry behavior;
- concurrent updates.

For each case, state the condition, system response, user-visible feedback, and whether data changes or remains unchanged. If behavior is not decided, mark it TBD and reference a decision.

#### 6.14 Edge cases
Include relevant boundary conditions and unusual but valid cases. Do not duplicate generic failure handling unless the case has module-specific impact.

#### 6.15 Audit, notification, and privacy impact
Where applicable, specify:

- audit event/action;
- actor, target, and scope;
- data that must or must not be logged;
- notification trigger and recipient;
- sensitive information handling;
- retention or visibility dependency.

If these are centrally defined in Modules 13 or 15, reference the specific requirement instead of duplicating it.

#### 6.16 Acceptance criteria
Write measurable, observable criteria. Prefer Given / When / Then or numbered conditions.

Acceptance criteria must cover:

- normal success;
- permission/scope behavior;
- key validation;
- important alternate/failure flow;
- resulting data/state;
- any critical boundary condition.

Do not use subjective criteria such as “works correctly” or “is user-friendly” without defining what can be observed.

#### 6.17 QA test cases
Provide concrete test cases or reference a dedicated test-case document.

| Test ID | Scenario | Preconditions / test data | Steps | Expected result | Requirement / decision trace |
|---|---|---|---|---|---|

At minimum, cover the happy path, relevant permissions, required validation, and important failure/edge cases. A QA test must have an expected result; a test title alone is insufficient.

#### 6.18 Unresolved items
List any unresolved questions that prevent the function from being finalized. Link to the module decision log using a stable decision ID.

---

## 7. Roles, permissions, and scope matrix

Each module must include a permission matrix for relevant actions. Use explicit values such as **Allow**, **Deny**, **Conditional**, or **TBD**.

| Role | Action | Scope | Conditions | Status / source |
|---|---|---|---|---|

Do not infer permission from role names, page mappings, or frontend navigation. If access depends on membership, ownership, organization, lifecycle state, or subscription, state the condition.

## 8. Data and state reference

For each entity or data object used by the module, specify:

- field name and business meaning;
- type/format where supported by sources;
- required/optional status;
- allowed values;
- constraints and relationships;
- source of truth;
- lifecycle/status values;
- who can view/change the field;
- unresolved schema conflicts.

Do not treat conceptual schema as proof of legacy implementation. Do not invent schema details absent from the source.

For lifecycle behavior, include a state-transition table where applicable:

| Current state | Trigger/action | Preconditions | Next state | Side effects | Actor | Status |
|---|---|---|---|---|---|---|

## 9. Cross-module consistency

Every module must identify dependencies and avoid defining conflicting rules.

Examples:

- authentication and authorization rely on Module 1;
- organization hierarchy and scope rely on Module 2;
- subscriptions and seat eligibility rely on Module 3;
- common notifications rely on Module 13;
- audit event definitions rely on Module 15;
- canonical state machines rely on Module 17;
- shared failure behavior relies on Module 18.

When a shared rule is owned by another module, reference it and do not redefine it inconsistently.

## 10. Decision log

Every unresolved product, security, or behavior decision must be recorded.

| Decision ID | Question / conflict | Options (if known) | Recommendation (optional) | Impact if unresolved | Decision owner | Status | Approved decision |
|---|---|---|---|---|---|---|---|

Decision status should be one of:

- Open
- Recommendation pending review
- Approved
- Rejected
- Deferred
- Superseded

Do not record a recommendation as approved. Keep the approved decision and the historical recommendation distinguishable.

## 11. Document review status

Use one of the following document statuses:

- **Source-based draft:** organized from available sources; functional behavior is not yet fully specified.
- **Detailed draft:** function-level behavior is described, but decisions or review remain open.
- **Ready for product review:** function-level specifications and traceability are complete; unresolved decisions are clearly presented.
- **Approved functional specification:** product owner has explicitly approved the target behavior. Any deferred decisions remain clearly identified and cannot be implemented as settled behavior.
- **Superseded:** replaced by a newer approved version.

“Committed to GitHub” or “approved for repository inclusion” means the document is stored/versioned. It does **not** mean the functional behavior is approved.

## 12. Readiness checklist

A module is ready for product review when:

- [ ] Every in-scope function appears in the functional inventory.
- [ ] Every function has a complete function-level specification or clearly identified decision gaps.
- [ ] Actors, permissions, and organizational scope are explicit.
- [ ] Entry points and user interaction sequences are described.
- [ ] Inputs, fields, constraints, and validations are documented.
- [ ] Main flow and system/data behavior are described step by step.
- [ ] Success, alternate, failure, and important edge cases are covered.
- [ ] State changes, audit, and notifications are specified or referenced.
- [ ] Acceptance criteria are observable and testable.
- [ ] QA cases include expected results.
- [ ] Source facts, legacy findings, target requirements, derived behavior, and recommendations are clearly distinguished.
- [ ] Conflicts and unresolved decisions are tracked.
- [ ] Cross-module dependencies and terminology are consistent.
- [ ] No unapproved recommendation is presented as a confirmed requirement.

## 13. Review and approval workflow

1. Collect and identify relevant source documents and legacy evidence.
2. Inventory module functions and user workflows.
3. Draft the module using this standard.
4. Identify missing facts, conflicts, and decisions.
5. Perform a consistency and QA-readiness review.
6. Present the module for user/product-owner review.
7. Incorporate feedback and record decisions.
8. Obtain explicit approval for the target behavior.
9. Update document and index status accurately.
10. Proceed to the next module only when the agreed review gate is satisfied.

**Working sequence:** Upgrade Modules 1–9 using this standard before starting Module 10 (Groups). Module 1 should be used as the pilot to validate the template and level of detail.
