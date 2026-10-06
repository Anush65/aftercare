# Aftercare Domain Model

## 1. Purpose

Aftercare helps a reseller handle problems discovered when a physical shipment arrives at the receiving dock.

The receiving user may have their hands occupied while unpacking products, so the primary interaction is voice-first. The user describes what arrived, Aftercare reconciles the physical receipt against the expected purchase order, determines the resulting discrepancy and financial impact, and can prepare a recovery action such as a supplier claim.

The system is designed around a strict separation of responsibilities:

- The **agent** interprets the user's request, selects the appropriate tool, and communicates the result.
- The **MCP server** exposes controlled operations to the agent.
- **Deterministic business logic** performs reconciliation, calculations, validation, and scoring.
- **SQLite** stores the underlying business state and audit information.
- **Approval logic** prevents consequential actions from being executed without explicit confirmation.

The language model must not be the source of truth for business numbers.

---

## 2. Core Domain Concepts

### 2.1 Supplier

A supplier is a business or party from whom the reseller purchases products.

Example:

> Supplier: Ravi Electronics

A supplier is associated with purchase orders and receiving history.

The supplier record should contain only information required by the current demo and business logic. Supplier reliability must **not** be stored as a manually assigned or hardcoded score.

Supplier performance is derived from historical receipt data.

---

### 2.2 Product

A product represents an item that the reseller purchases, receives, sells, or returns.

Example:

> Product: iPhone 17  
> Unit price: ₹52,000

A product can appear on multiple purchase-order lines and receipt lines.

For the initial demo, product information should remain intentionally small. We do not need a full product-information-management system.

---

### 2.3 Purchase Order

A purchase order (PO) represents an order placed by the reseller with a supplier.

Example:

> PO 1001  
> Supplier: Ravi Electronics  
> Status: Open

A purchase order contains one or more purchase-order lines.

A PO establishes what the reseller expected to receive. It is therefore the primary reference against which a physical receipt is reconciled.

A purchase order may have multiple receipts because a supplier may deliver an order in multiple shipments.

---

### 2.4 Purchase Order Line

A purchase-order line represents one product and its expected quantity within a purchase order.

Example:

| Product   | Ordered Quantity | Unit Price |
| --------- | ---------------: | ---------: |
| iPhone 17 |               20 |    ₹52,000 |

A PO line connects a purchase order to a product and provides the expected quantity and financial basis for reconciliation.

Keeping PO and PO line separate allows one purchase order to contain multiple products.

---

### 2.5 Receipt

A receipt represents a physical receiving event at the reseller's receiving dock.

A receipt is associated with a purchase order and records what was observed during a particular receiving event.

Example:

> Receipt R-001  
> PO: 1001  
> Supplier: Ravi Electronics

A PO may have multiple receipts over time.

A receipt should be treated as an event in the receiving history rather than as a replacement for the original purchase order.

---

### 2.6 Receipt Line

A receipt line records the receiving result for a particular product during a receipt.

For the current demo, the receiving quantities are explicitly distinguished as:

- **Sellable quantity** — units received in acceptable condition.
- **Damaged quantity** — units physically received but not sellable because of observed damage.
- **Missing quantity** — units expected according to the PO but not present in the shipment.

Example:

> PO quantity: 20  
> Sellable: 17  
> Damaged: 1  
> Missing: 2

This means:

```text
17 sellable
+ 1 damaged
+ 2 missing
= 20 expected

This distinction is important because a damaged unit is physically present but has a different business consequence from a completely missing unit.
3. Domain Relationships
The initial domain relationship is:
Supplier
   │
   └──< Purchase Order
            │
            └──< Purchase Order Line >── Product
            │
            └──< Receipt
                    │
                    └──< Receipt Line >── Product

In relational terms:
Supplier 1 ── * PurchaseOrder

PurchaseOrder 1 ── * PurchaseOrderLine

Product 1 ── * PurchaseOrderLine

PurchaseOrder 1 ── * Receipt

Receipt 1 ── * ReceiptLine

Product 1 ── * ReceiptLine

The same product may therefore appear on multiple orders and receipts.
4. The Receiving Workflow
The central operation in the first Aftercare scene is record_receipt.
Conceptually:
User describes shipment
        ↓
Agent identifies the relevant PO
        ↓
Agent collects missing receiving information
        ↓
MCP record_receipt tool
        ↓
Validate input
        ↓
Load PO and expected quantities
        ↓
Reconcile physical receipt against expectation
        ↓
Calculate discrepancy and financial impact
        ↓
Persist receipt
        ↓
Return deterministic result
        ↓
Agent explains result to user

The agent should not independently calculate the reconciliation.
For example, if the user says:
"I've received Ravi's shipment. Two are missing and one screen is damaged."

the system should resolve this into structured receiving information and pass it to deterministic business logic.
The business logic then determines the verified result.
5. record_receipt Conceptual Input
The exact MCP schema will be defined during implementation, but conceptually the operation needs enough information to identify the receiving event and its observed quantities.
At minimum, this includes:
- Purchase order identifier
- Product or PO line identifier
- Sellable quantity
- Damaged quantity
- Receiving event information
The system should not require the user to manually provide information that can be safely obtained from the PO.
For example, the user should not need to say:
"The unit price is ₹52,000."

The system should retrieve the authoritative unit price from the purchase-order data.
6. Reconciliation Rules
The core reconciliation invariant for a complete receipt line is:
sellable_quantity
+ damaged_quantity
+ missing_quantity
= ordered_quantity

All quantities must be non-negative integers.
Therefore:
sellable_quantity >= 0
damaged_quantity >= 0
missing_quantity >= 0

The system must reject inconsistent input.
For example:
Ordered: 20
Sellable: 18
Damaged: 3
Missing: 2

is invalid because:
18 + 3 + 2 = 23

which exceeds the ordered quantity.
The server must reject such input rather than allowing the language model or user interface to silently normalize it.
7. Partial Deliveries
A purchase order may be fulfilled through multiple receiving events.
For example:
PO quantity: 20

Receipt 1:
10 sellable

Receipt 2:
8 sellable
1 damaged

The system must distinguish between:
- quantity received in the current receipt,
- cumulative quantity received across receipts,
- quantity still outstanding on the purchase order.
This means reconciliation should consider previous receipts, not only the current receipt.
The precise treatment of missing quantity across multiple partial receipts must be finalized during implementation so that a temporarily outstanding quantity is not incorrectly treated as a permanent supplier failure.
8. Damaged vs Missing
A damaged unit and a missing unit represent different physical situations.
Damaged
The product was physically received but is not considered sellable.
Example:
Ordered: 20
Sellable: 17
Damaged: 1
Missing: 2

The damaged unit may create a supplier recovery or warranty action.
Missing
The product expected according to the purchase order was not physically received.
A missing unit contributes to the outstanding discrepancy and may also create a supplier claim.
The system should preserve these categories separately because they can result in different downstream actions.
9. Financial Impact
Financial values must be calculated by deterministic business logic using authoritative product or purchase-order data.
The language model must never invent or independently calculate financial values for the user.
For example, if:
Unit value = ₹52,000
Missing = 2
Damaged = 1

the system can calculate the relevant claim or risk value according to the project's explicitly defined business rule.
The term "value at risk" must have one precise definition in the implementation.
It must not be used ambiguously to mean:
- total PO value,
- missing inventory value,
- damaged inventory value,
- potential lost sales,
- or supplier claim value.
The final implementation should document exactly which value is being reported.
10. Supplier Reliability
Supplier reliability is derived from historical receiving data.
It must not be hardcoded as a static value such as:
Ravi = 92%

Instead, the supplier scorecard should be computed from recorded receipt history.
Potential inputs include:
- late deliveries,
- missing quantities,
- damaged quantities,
- total ordered quantities,
- historical receiving events.
The exact scoring formula will be defined and tested separately before implementation.
The score returned to the agent must originate from deterministic server-side calculations.
11. Stored vs Calculated Data
The system should distinguish between facts that happened and values that can be derived from those facts.
Stored facts
Examples:
- supplier identity
- product identity
- PO identity
- ordered quantity
- unit price
- receipt event
- sellable quantity
- damaged quantity
- timestamps
- claim/action status
- audit events
Calculated values
Examples:
- missing quantity
- outstanding quantity
- discrepancy value
- supplier reliability score
- forecasted days of stock
- financial impact
Calculated values should generally be derived from stored facts rather than manually entered or hardcoded.
This reduces the risk of stale or contradictory data.
12. Important Business Invariants
The following rules should be enforced by deterministic server-side logic.
Quantity validity
All quantities must be non-negative integers.

Reconciliation consistency
For a completed receiving determination:
sellable + damaged + missing = expected quantity

No over-receiving without explicit domain support
The system should not silently accept:
sellable + damaged > ordered

unless the domain model explicitly supports over-delivery.
For the initial demo, over-delivery should be treated as an exceptional condition requiring explicit handling rather than silently accepted.
Authoritative pricing
Financial calculations must use stored product/PO pricing rather than a price supplied by the language model.
Supplier score integrity
Supplier reliability must be derived from receipt history.
Auditability
A consequential action must have an associated audit record.
Approval
Drafting an action must not execute the action.
Only the approval mechanism is allowed to execute consequential actions.
13. Example: Ravi Shipment
Consider the primary demo scenario.
Expected
Supplier: Ravi
PO: 1001
Product: iPhone 17
Ordered: 20

Observed
Sellable: 17
Damaged: 1
Missing: 2

Reconciliation
17 + 1 + 2 = 20

Therefore the receiving data is internally consistent.
The system can then determine the applicable financial impact using the authoritative unit value.
The resulting information can be used to draft a supplier claim.
Importantly:
record_receipt

records and reconciles the receiving event.
It does not automatically send a supplier claim.
A later tool/action flow is responsible for drafting and, after explicit approval, executing the consequential action.
14. Separation Between Drafting and Execution
After reconciliation, Aftercare may identify an appropriate recovery action.
For example:
Receipt discrepancy
        ↓
Draft supplier claim
        ↓
User reviews
        ↓
User explicitly approves
        ↓
Confirm action
        ↓
Execute simulated supplier communication

The initial receipt operation must therefore remain separate from the execution of consequential actions.
This separation supports:
- user control,
- auditability,
- idempotency,
- safer agent behavior.
15. Edge Cases to Test
The following cases should eventually become automated tests.
Case A — Normal complete receipt
Ordered: 20
Sellable: 20
Damaged: 0
Missing: 0

Expected: valid.
Case B — Missing and damaged units
Ordered: 20
Sellable: 17
Damaged: 1
Missing: 2

Expected: valid discrepancy.
Case C — Quantity exceeds expected quantity
Ordered: 20
Sellable: 18
Damaged: 3
Missing: 0

Expected: rejected or explicitly handled as an over-delivery condition.
Case D — Negative quantity
Ordered: 20
Sellable: -1
Damaged: 0
Missing: 21

Expected: rejected.
Case E — Partial delivery
Ordered: 20

Receipt 1:
10 sellable

Receipt 2:
10 sellable

Expected: cumulative receiving history should reconcile to 20.
Case F — Repeated receipt submission
Submitting the same receiving event twice should not silently create duplicate business effects.
The exact idempotency mechanism will be designed as part of the approval/action architecture.
16. Relationship to the Agent
The agent is responsible for interpreting natural language and selecting tools.
For example:
User:
"I've received Ravi's shipment. Two are missing and one is damaged."

                ↓

Agent

                ↓

record_receipt(...)

                ↓

Deterministic business logic

                ↓

Verified result

                ↓

Agent explains result

The agent should not be trusted to:
- calculate financial values,
- calculate supplier scores,
- infer unsupported database facts,
- bypass validation,
- execute consequential actions without confirmation.
This is a core design principle of Aftercare.
17. Initial Scope Boundary
The domain model intentionally does not attempt to model every part of a reseller's business.
The first implementation focuses on the receiving and recovery workflow.
Out of scope for the initial domain model:
- full accounting
- warehouse management
- general-purpose inventory management
- payment processing
- real supplier API integrations
- real supplier email delivery
- real Alexa+ device integration
- continuous autonomous monitoring
These may be simulated or represented elsewhere in the demo where appropriate, but they should not expand the core receiving domain unnecessarily.
18. Open Questions
The following questions should be resolved before or during implementation rather than silently assumed:
1. What exact definition should be used for claim value?
2. What exact definition should be used for value at risk?
3. How should missing quantities be represented when a PO has multiple partial receipts?
4. Should missing_quantity be stored directly or derived from expected and received quantities?
5. What uniquely identifies a receiving event for idempotency?
6. How should over-delivery be handled?
7. What exact formula should determine the supplier reliability score?
8. Should a receipt be editable after it has been recorded, or should corrections create a separate event?
9. What exact receiving states are needed, if any?
10. Which information should be required from the user versus retrieved from the purchase order?
11. What exact business event causes a supplier claim to become eligible for drafting?
12. Which values must be persisted for the audit log to reproduce why a decision was made?
These questions are intentionally recorded because unresolved assumptions are more dangerous than explicitly acknowledged uncertainty.
19. Design Principle
The central domain principle for Aftercare is:
The receiving event is the source of truth for what physically happened; the purchase order is the source of truth for what was expected; deterministic business logic reconciles the two.

The agent provides the conversational interface between the user and these systems, but it is not the authority on the underlying business facts.
```
