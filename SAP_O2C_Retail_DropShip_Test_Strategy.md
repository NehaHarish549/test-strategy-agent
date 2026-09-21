# Test Strategy: Order-to-Cash (SD/FI) — Retail Client, New Implementation
### Including Third-Party Drop-Shipment

**Prepared for:** New S/4HANA Implementation Program
**Date generated:** 2026-09-18
**Prepared by:** SAP Test Strategy & Scenario Generator Agent

---

## Scope & Confidence Legend

This document distinguishes two categories of content throughout:

| Marker | Meaning |
|---|---|
| *(standard)* | Based on documented, standard SAP process behavior — generated with confidence. |
| ⚠ **SME REQUIRED** | Depends on client-specific configuration, custom code, or middleware detail this agent cannot verify. Flagged rather than guessed — see Section 6. |

No tolerance value, config setting, or org-structure detail in this document is invented. Where a typical
value is shown for illustration, it is explicitly labeled "illustrative — confirm actual setting."

---

## 1. Scope & Assumptions

| Parameter | Value |
|---|---|
| Business process | Order-to-Cash (O2C) |
| SAP module(s) | SD (Sales & Distribution), FI (Financial Accounting) |
| Engagement type | New implementation |
| Client profile | Retail |
| Stated constraint / focus area | Includes third-party drop-shipment scenarios |
| Additional constraints stated | None supplied |

**Risk emphasis rationale (new implementation):** Because this is a first-time build rather than a
regression or upgrade, this strategy weights heavily toward (a) end-to-end configuration validation across
SD and FI — pricing, account determination, tax, credit management — since none of it has been proven in
this landscape before; (b) master data readiness (customer, material, vendor, condition records) as a
go-live blocker risk; (c) first-time validation of the drop-shipment integration path (SD third-party item
category triggering MM procurement), which is a cross-module process new implementations frequently
under-test; and (d) cutover/data-migration risk for open sales orders and master data cutover.

**Process assumption (standard O2C):** Sales Order → Delivery → Goods Issue (or, for drop-ship, vendor
goods receipt/confirmation) → Billing → FI posting → Payment/Collection. If the client's actual flow
differs materially (e.g. no delivery-based billing, order-related billing only), this assumption should be
confirmed before test execution begins — this document proceeds on the standard delivery-based billing
assumption for the "own-warehouse" leg and order-related billing for the drop-ship leg *(standard)*.

---

## 2. Test Scenario Catalogue

Scenarios are business-level conditions ("what to test"), not scripted steps. Risk ratings reflect business
impact if the scenario fails in production, not effort to test.

### 2.1 Order Creation & Pricing (OC)

| ID | Scenario | Type | Risk | Justification | Traceability |
|---|---|---|---|---|---|
| OC-01 | Standard sales order created with valid customer, material, and pricing conditions determines price, tax, and availability correctly *(standard)* | Happy path | Medium | Core revenue entry point; failure blocks all downstream O2C activity, but is highly visible and caught early | Order Creation |
| OC-02 | Sales order for a customer with incomplete master data (missing tax classification, missing pricing-relevant field) is created | Edge | High | Incomplete master data causes silent pricing/tax errors that surface only at billing — expensive to trace back and can cause revenue leakage or tax misstatement | Order Creation |
| OC-03 | Pricing condition record missing or expired for an order item (no valid price found) | Edge | High | Order proceeds with $0 or incorrect price if not blocked — direct revenue leakage risk | Order Creation / Pricing |
| OC-04 | Multiple overlapping pricing conditions (e.g. customer-specific + material-specific discount) produce a combined price outside expected margin | Edge | Medium | Margin erosion risk; typically caught in UAT by business but should be scenario-tested, not assumed | Order Creation / Pricing |
| OC-05 | Order quantity exceeds available stock and/or credit limit simultaneously | Edge | Medium | Tests interaction of availability check (ATP) and credit management together, a common gap when each is tested in isolation | Order Creation |
| OC-06 | Free-of-charge / sample order (no billing relevance) processed correctly without triggering FI revenue posting | Edge | Low | Common retail scenario; failure causes accounting noise rather than customer-facing failure | Order Creation |

### 2.2 Credit Management (CM)

| ID | Scenario | Type | Risk | Justification | Traceability |
|---|---|---|---|---|---|
| CM-01 | Order for a customer within approved credit limit processes without a credit hold *(standard)* | Happy path | Low | Expected behavior; low business impact if occasionally over-permissive in test | Credit Check |
| CM-02 | Order exceeding customer credit limit is correctly blocked and routed to the credit release workflow | Edge | High | If credit blocks fail to trigger, retailer ships against unsecured receivables — direct financial exposure | Credit Check |
| CM-03 | Released/blocked order interaction with subsequent delivery — a delivery should not be creatable/PGI-able against a still-blocked order | Edge | High | A gap here lets goods leave the warehouse against a blocked order, defeating the entire credit control purpose | Credit Check → Delivery |

### 2.3 Delivery & Fulfillment — Own Warehouse Leg (DF)

| ID | Scenario | Type | Risk | Justification | Traceability |
|---|---|---|---|---|---|
| DF-01 | Delivery created from sales order, full quantity picked and goods-issued on time *(standard)* | Happy path | Medium | Core fulfillment path; failure is highly visible and typically caught immediately | Delivery / Goods Issue |
| DF-02 | Partial delivery / short-shipment — warehouse can only fulfill part of the ordered quantity | Edge | High | Retail-specific pain point: triggers partial billing, backorder processing, and customer communication — a common real-world failure point that's frequently under-tested | Delivery |
| DF-03 | Backorder processing — remaining quantity from a short-shipment is correctly tracked and re-triggers delivery once stock is available | Edge | Medium | If backorder tracking fails, the unfulfilled balance is silently lost rather than escalated | Delivery |
| DF-04 | Batch/serial-managed material delivery selects correct batch/serial and reflects it on the delivery document | Edge | Medium | Retail with batch-managed goods (e.g. perishables, electronics with serials) — traceability and recall-readiness risk if batch determination is wrong | Delivery |
| DF-05 | Return delivery for a previously shipped order, reversing stock and triggering a credit memo | Edge | Medium | Retail has high return volume; failure causes inventory/financial discrepancy, but is usually caught in reconciliation | Delivery / Returns |

### 2.4 Third-Party Drop-Shipment (DS) — Client-Stated Focus Area

Drop-ship uses SD third-party item category (standard config: **TAS**), which on order save automatically
triggers a purchase requisition/purchase order to the vendor in MM. The vendor ships directly to the
end customer; the retailer never holds physical stock for this item. This cross-module handoff is the
scenario group most likely to be under-tested in a standard O2C test plan, per the client's stated focus.

| ID | Scenario | Type | Risk | Justification | Traceability |
|---|---|---|---|---|---|
| DS-01 | Sales order line with third-party item category correctly auto-generates a purchase requisition/PO to the assigned vendor *(standard)* | Happy path | High | This is the entire mechanism drop-ship depends on; if the PO doesn't generate, the customer order silently stalls with no physical fulfillment path | Order Creation → Procurement Trigger |
| DS-02 | Vendor confirms shipment and a **statistical goods receipt** is posted in MM (no physical warehouse stock impact) *(standard)* | Happy path | Medium | Confirms the "GR without inventory movement" behavior specific to third-party flow behaves as designed, not as a standard stock GR | Vendor Fulfillment |
| DS-03 | Customer billing is attempted **before** the vendor's shipment/GR confirmation is received | Edge | High | Retail-specific revenue recognition risk: billing ahead of confirmed vendor fulfillment can overstate revenue or bill a customer for goods not yet shipped | Billing timing dependency |
| DS-04 | Vendor invoice (MIRO) is received and posted **before** the customer billing document is created | Edge | Medium | Tests costing sequence — cost of goods may need to be known before revenue/margin is finalized in COGS-sensitive retail reporting | Vendor Invoice → Customer Billing sequence |
| DS-05 | Vendor invoice price differs from the originally expected PO price ("price variance" on the third-party PO) | Edge | High | Directly affects margin reporting on a sale the retailer never had visibility/control of physical goods for — a known drop-ship margin-leakage pattern | Vendor Invoice |
| DS-06 | A single customer sales order line with drop-ship quantity is fulfilled by **two separate partial vendor shipments** | Edge | High | Tests whether partial third-party fulfillment correctly triggers partial customer billing rather than forcing an all-or-nothing billing block | Vendor Fulfillment → Billing |
| DS-07 | Customer initiates a return on a drop-shipped item that the retailer never physically received into its own warehouse | Edge | High | Retail-specific edge case: standard return-delivery flow assumes the retailer's own warehouse receives the return; drop-ship returns typically route back to the vendor directly or require a distinct process — high risk of process gap if not explicitly designed and tested | Returns (drop-ship variant) |
| DS-08 | Item category / vendor assignment is misconfigured for a material flagged as drop-ship, and no PO is generated at all | Failure | High | Silent-failure scenario — the sales order looks normal to the customer-facing team while no procurement action occurs behind it; high risk of going unnoticed until a customer complains about non-delivery | Order Creation → Procurement Trigger |
| DS-09 | Vendor cancels or cannot fulfill the drop-ship order after the PO was created and the customer has already been promised a delivery date | Edge | Medium | Customer-experience and re-sourcing process risk; financially recoverable but reputationally costly if not tested | Vendor Fulfillment |

### 2.5 Billing & Revenue Recognition (BL)

| ID | Scenario | Type | Risk | Justification | Traceability |
|---|---|---|---|---|---|
| BL-01 | Standard billing document created from delivery, correctly reflects price, tax, and triggers FI posting *(standard)* | Happy path | Medium | Core billing path; visible failure, typically caught quickly | Billing |
| BL-02 | Partial billing against a partially delivered order (ties to DF-02) correctly bills only the delivered quantity | Edge | High | If billing doesn't correctly split against partial delivery, retailer either under-bills (revenue leakage) or over-bills (customer dispute/credit note rework) | Billing |
| BL-03 | Billing block due to incomplete data (e.g. missing cost, missing tax classification) is correctly triggered rather than allowing an incomplete document through | Edge | High | An unblocked incomplete billing document can post inaccurate revenue/tax directly to FI, requiring downstream correction | Billing |
| BL-04 | Credit memo / debit memo request correctly reverses or adjusts a prior billing document and flows to FI | Edge | Medium | Common in retail returns/pricing disputes; failure causes reconciliation effort but is generally caught in period-close review | Billing / Adjustments |
| BL-05 | Intercompany billing scenario, if retailer's drop-ship vendor relationship is intercompany rather than third-party external vendor | Edge | ⚠ **SME REQUIRED** | Whether this scenario applies at all depends on the client's actual vendor/company-code structure, which has not been supplied — flagging rather than assuming intercompany applies | Billing (conditional) |

### 2.6 FI Integration & Financial Postings (FI)

| ID | Scenario | Type | Risk | Justification | Traceability |
|---|---|---|---|---|---|
| FI-01 | Billing document correctly determines the FI account via account determination (condition technique, e.g. VKOA) *(standard)* | Happy path | High | If account determination fails or misroutes, revenue posts to the wrong GL account — a financial statement accuracy issue, not just a process hiccup | Billing → FI Posting |
| FI-02 | Tax determination on a customer order correctly applies the right tax code/jurisdiction for retail sales (including any tax-exempt customer scenarios, if applicable to this client) | Edge | High | Tax misclassification is a compliance risk, not just an internal error — potential regulatory exposure | Order/Billing → Tax |
| FI-03 | Foreign currency sales order/billing (if retailer sells cross-border) correctly applies exchange rate at the appropriate rate date | Edge | ⚠ **SME REQUIRED** | Applicability depends on whether this retail client actually transacts in multiple currencies — not stated in scope; flagging rather than assuming single- or multi-currency | Billing → FI Posting |
| FI-04 | Vendor invoice for the drop-ship leg posts correctly to accounts payable, independent of and reconcilable against the customer billing posting in accounts receivable | Edge | High | The AP/AR postings for the same underlying transaction happen on two separate documents (vendor invoice vs. customer billing) — reconciliation gap here directly affects financial statement accuracy for drop-ship revenue | Vendor Invoice / Billing → FI |
| FI-05 | Payment received from customer correctly clears the open AR item, including partial payment handling | Edge | Medium | Standard collections process; failure is a reconciliation/DSO issue rather than a customer-facing failure | Payment/Collection |

### 2.7 Integration Points & Interfaces (INT)

| ID | Scenario | Type | Risk | Justification | Traceability |
|---|---|---|---|---|---|
| INT-01 | Order intake channel (e.g. EDI, e-commerce storefront, or manual entry) fails to transmit a valid order into SD | Failure | High | If this client's order intake is channel-integrated (common in retail), an interface failure means orders never reach SAP at all — a silent revenue-loss risk. **The specific integration technology/middleware in use has not been stated** | Order Creation (upstream) |
| INT-02 | Outbound customer-facing document (e.g. order confirmation, ASN, invoice) fails to transmit via the output channel in use | Failure | Medium | Customer-experience risk (no confirmation/invoice received) rather than a data-integrity risk, since the underlying SAP document itself is usually still correct | Output Determination |
| INT-03 | Downstream integration point (e.g. warehouse management system, e-commerce order status feed, tax engine, payment gateway) fails or times out mid-transaction | Failure | ⚠ **SME REQUIRED** | This client's actual downstream integration landscape has not been specified. The *category* of risk (any downstream integration failure needs a defined fallback/retry/reconciliation behavior) is valid test-design judgment; the *specific* systems and failure modes cannot be named without that detail | Various (downstream) |

### 2.8 Master Data & Cutover Readiness (MD) — New-Implementation Specific

| ID | Scenario | Type | Risk | Justification | Traceability |
|---|---|---|---|---|---|
| MD-01 | Customer master data migrated/created is complete for all SD/FI-relevant fields (pricing, tax, credit, output) before cutover | Edge | High | This is a new implementation — incomplete master data is a go-live blocker, not a runtime edge case; failure surfaces as OC-02/OC-03/FI-02-type errors at scale on day one | Cutover Readiness |
| MD-02 | Material master data for drop-ship-eligible items correctly carries the third-party item category and vendor assignment | Edge | High | Direct precondition for DS-01; a data setup gap here reproduces DS-08's failure mode across many materials at once, not just one order | Cutover Readiness |
| MD-03 | Open sales orders/deliveries migrated from the legacy system (if applicable) retain correct status and process correctly through to completion in the new system | Edge | ⚠ **SME REQUIRED** | Whether there is a legacy-to-S/4HANA data migration of open transactional documents (versus a clean cutover with no open-order carryover) has not been stated by the client | Cutover / Data Migration |

---

## 3. Entry / Exit Criteria

Criteria are scoped to a new-implementation program: SIT, UAT, and a Cutover Rehearsal phase (not a
pure regression cycle's criteria, per Section 1's risk-emphasis rationale).

### 3.1 System Integration Testing (SIT)

**Entry criteria:**
- SD and FI configuration for the O2C flow (pricing, account determination, tax, credit management,
  third-party item category/procurement determination) is complete and transported to the SIT environment.
- Master data listed in MD-01/MD-02 is loaded for at least the representative subset of test materials/
  customers/vendors needed to execute Section 2's scenario catalogue.
- Drop-ship vendor(s) are set up in MM with valid purchasing info records for the materials in scope.
- Unit-level configuration checks (e.g. a single order can be created, priced, and saved) have passed —
  SIT is not the phase to discover basic configuration is missing.

**Exit criteria:**
- All High-risk scenarios in Section 2 have executed with a Pass result, or have an open defect with a
  documented workaround and business sign-off to proceed.
- All Medium-risk scenarios have executed at least once, with no open Severity-1/2 defects.
- The drop-ship cross-module handoff (DS-01 through DS-09) has been executed end-to-end at least once
  with real vendor/PO data flowing through to a customer billing document.
- Defect log reviewed and triaged jointly by SD, FI, and MM functional leads (cross-module sign-off,
  given the drop-ship dependency spans SD and MM).

### 3.2 User Acceptance Testing (UAT)

**Entry criteria:**
- SIT exit criteria met.
- Business users assigned to execute UAT have been walked through the drop-ship process flow
  specifically (not assumed to be familiar with it, since it is a new process for this client).
- Test data available in UAT reflects realistic retail volumes/patterns, not single-record happy-path data
  only.

**Exit criteria:**
- Business sign-off obtained specifically on: credit management behavior (CM-01–CM-03), partial
  delivery/billing (DF-02, BL-02), and the drop-ship customer experience end-to-end (DS-01–DS-09) —
  these are the scenario groups most likely to surface a business-process gap rather than a pure
  configuration defect.
- No open Severity-1 defects; open Severity-2/3 defects have documented business-accepted workarounds.
- Reconciliation check performed: a sample of drop-ship transactions traced from sales order → vendor
  PO/invoice → customer billing → FI posting, confirming AP and AR sides reconcile (ties to FI-04).

### 3.3 Cutover Rehearsal (New-Implementation Specific)

**Entry criteria:**
- UAT exit criteria met.
- Final master data load approach confirmed (see MD-03 flag — if open-order migration applies, cutover
  rehearsal must include it).

**Exit criteria:**
- Cutover rehearsal completed within the planned downtime window (or the actual window is renegotiated
  before the real cutover, not discovered live).
- Post-migration validation confirms a sample of migrated/created master data (MD-01, MD-02) is usable
  for order entry without error.
- Go/no-go decision documented with named business and IT sign-off.

---

## 4. Risk Summary

| Process Area | High | Medium | Low | Total |
|---|---|---|---|---|
| Order Creation & Pricing (OC) | 2 | 3 | 1 | 6 |
| Credit Management (CM) | 2 | 0 | 1 | 3 |
| Delivery & Fulfillment (DF) | 1 | 3 | 0 | 4 (excl. DS) |
| Third-Party Drop-Shipment (DS) | 5 | 3 | 0 | 8 |
| Billing & Revenue Recognition (BL) | 2 | 2 | 0 | 4 (+1 flagged) |
| FI Integration (FI) | 3 | 1 | 0 | 4 (+1 flagged) |
| Integration/Interfaces (INT) | 1 | 1 | 0 | 1 (+1 flagged) |
| Master Data & Cutover (MD) | 2 | 0 | 0 | 2 (+1 flagged) |

**Observation:** Risk concentrates heavily in the Third-Party Drop-Shipment group (5 of 9 scenarios rated
High) — consistent with the client's own stated focus area, and consistent with drop-ship being a
cross-module (SD+MM+FI) process being validated for the first time in a new implementation. This group
warrants disproportionate SIT/UAT time and explicit business sign-off, not pro-rata effort based on
scenario count alone.

---

## 5. Traceability Note

Each scenario ID in Section 2 is tagged with a **Traceability** reference to the O2C process step it
validates (Order Creation, Credit Check, Delivery, Vendor Fulfillment, Billing, FI Posting, Cutover
Readiness, etc.). This mapping is structured so it can seed a Requirements Traceability Matrix (RTM)
directly: each row becomes an RTM entry linking a business process step → test scenario ID → (once
written) detailed test case ID → execution result. This document does not itself contain step-level test
cases; it defines the scenario layer the RTM's test-case layer would trace up to.

---

## 6. Out-of-Scope / Requires SME Input

The following were identified during scenario generation but explicitly **not** answered with an assumed
or invented detail, per this agent's honesty rule. Each requires functional consultant or client input
before the affected scenario can be finalized:

1. **BL-05 / Intercompany billing** — whether the drop-ship vendor relationship is external third-party or
   intercompany was not stated. This changes whether intercompany billing scenarios apply at all.
2. **FI-03 / Multi-currency** — whether this retail client transacts in multiple currencies/cross-border
   was not stated.
3. **INT-01/INT-02/INT-03 / Integration & middleware specifics** — the actual order-intake channel,
   output/interface technology (EDI, IDoc, API, middleware such as SAP PI/CPI), and downstream systems
   (WMS, e-commerce platform, tax engine, payment gateway) were not specified. This document identifies
   *that* integration risk exists and *where* in the process it sits, but cannot specify exact interface
   behavior, IDoc segment/message type details, or middleware-specific failure modes without that
   information.
4. **MD-03 / Legacy data migration scope** — whether open sales orders/deliveries are being migrated from
   a legacy system versus a clean cutover with no open-order carryover was not stated.
5. **Custom code / user-exits / BAdIs** — this document assumes standard SAP process behavior throughout.
   Any client-specific custom logic (pricing user-exits, credit-check BAdIs, custom drop-ship automation)
   would introduce additional risk and test scenarios that cannot be predicted without visibility into
   that custom code. Recommend a technical design review pass by the implementation's ABAP/functional
   leads specifically to identify custom logic touching the scenarios above, and extend this catalogue
   accordingly.
6. **Exact tolerance/configuration values** — where this document references standard SAP mechanisms
   (e.g. credit limit checks, pricing procedures), it does not state specific tolerance percentages, credit
   limit amounts, or pricing procedure step numbers, since none were supplied by the client. Confirm actual
   configured values (e.g. via OVA8 for credit management, the actual pricing procedure in V/08) with the
   client's SD/FI configuration team before finalizing detailed test cases.

---

## Assumptions & Disclaimers

- This document is a **test scenario strategy**, not a scripted test case set — see Rule 6 of this
  project's operating guide.
- Standard transaction codes and SAP terminology referenced (e.g. MIGO, MIRO, VKOA, OVA8, V/08) reflect
  documented standard SAP behavior and are used for traceability/precision, not as a claim about this
  client's specific configuration.
- This strategy should be reviewed by the program's SD, MM, and FI functional leads before being finalized
  as the governing test-strategy document, particularly to close the six items in Section 6.
