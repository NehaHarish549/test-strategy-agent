# Test Strategy: Procure-to-Pay (MM/FI) — Standard Domestic Goods Purchase
### Regression Cycle on Existing System

**Prepared for:** Existing S/4HANA System — Regression Test Cycle
**Date generated:** 2026-09-18
**Prepared by:** SAP Test Strategy & Scenario Generator Agent

---

## Scope & Confidence Legend

| Marker | Meaning |
|---|---|
| *(standard)* | Based on documented, standard SAP process behavior — generated with confidence. |
| ⚠ **SME REQUIRED** | Depends on client-specific configuration, custom code, change scope, or a detail this agent cannot verify. Flagged rather than guessed. |

---

## 1. Scope & Assumptions

| Parameter | Value |
|---|---|
| Business process | Procure-to-Pay (P2P) |
| SAP module(s) | MM (Materials Management), FI (Financial Accounting) |
| Engagement type | Regression cycle on existing system |
| Purchase profile | Standard domestic goods purchase |
| Constraints / focus areas | None stated — explicitly confirmed by client |

**Process assumption (standard P2P):** Purchase Requisition → Purchase Order → Goods Receipt → Invoice
Receipt (three-way match) → Payment. "Standard domestic goods purchase" is read as: stock material,
single company code/plant, domestic vendor, no import/customs duty, no intercompany or consignment
procurement, no service PO. If any of those actually apply, the scenario set below should be extended —
this document does not assume them since they were not stated *(standard)*.

**Risk emphasis rationale (regression cycle):** Per this agent's operating rules, a regression cycle
emphasizes change-impact analysis tied to the actual transport/change scope, confirmation that unaffected
areas still work, and targeted re-test rather than full re-test — not the configuration-build-out or
cutover risk that would apply to a new implementation.

⚠ **Critical scoping gap — flagged, not guessed:** No specific transport, configuration change, or code
change driving this regression cycle was supplied. Real change-impact prioritization (which of the
scenarios below are actually *at risk* from this specific change, versus which can be smoke-tested only)
requires the change/transport log. Without it, this document provides the **full standard P2P regression
baseline catalogue** rather than a change-targeted subset. Treat Section 2 as the candidate pool to
prioritize against the actual change scope once supplied — see Section 6, item 1.

---

## 2. Test Scenario Catalogue

### 2.1 Purchase Requisition & PO Creation (PR)

| ID | Scenario | Type | Risk | Justification | Traceability |
|---|---|---|---|---|---|
| PR-01 | Standard PR converted to PO for a domestic stock material with valid vendor and pricing *(standard)* | Happy path | Low | Core, well-proven path in an existing system; low likelihood of regression unless directly touched | PO Creation |
| PR-02 | PO price differs from an active outline agreement/contract price for the same material/vendor | Edge | Medium | Contract-price enforcement is a control point; a regression here causes silent margin/spend leakage rather than a blocked transaction | PO Creation / Pricing |
| PR-03 | PO created against a vendor currently blocked for purchasing (posting block) is correctly rejected | Edge | High | This is an existing control being **re-verified**, not built — a regression that silently disables it exposes the business to buying from a blocked vendor with no visible error | PO Creation / Vendor Control |
| PR-04 | PO above the existing approval threshold correctly triggers the release/approval workflow before it can be sent to the vendor | Edge | High | Release strategy is a financial control; if a recent change inadvertently bypasses it, unapproved spend can commit the company with no audit trail | PO Creation / Approval |

### 2.2 Goods Receipt (GR)

| ID | Scenario | Type | Risk | Justification | Traceability |
|---|---|---|---|---|---|
| GR-01 | Full goods receipt posted against the PO; stock quantity and GR/IR clearing account update correctly *(standard)* | Happy path | Medium | Core fulfillment posting; regression here affects every subsequent transaction, so worth confirming even though it's low-novelty | Goods Receipt |
| GR-02 | Partial goods receipt posted; PO remains open for the undelivered balance and reflects the correct open quantity | Edge | Medium | Partial receipts are common in domestic goods purchasing (split deliveries); a regression causing the PO to close prematurely or stay open incorrectly affects downstream matching | Goods Receipt |
| GR-03 | Over-delivery within the configured tolerance is accepted; over-delivery beyond tolerance is blocked | Edge | High | Tolerance enforcement is a control against unauthorized over-receipt; confirm the existing configured tolerance (illustrative example: a typical over-delivery tolerance is a small single-digit percentage — confirm actual OMR6/PO tolerance setting, not assumed here) still behaves as configured after the change | Goods Receipt / Tolerance Control |
| GR-04 | Goods receipt reversal (movement type 102) correctly reverses both the stock quantity and the GR/IR account posting | Edge | Medium | Reversal is a correction mechanism; if it fails to fully reverse the FI-relevant posting, it leaves a reconciling difference in GR/IR | Goods Receipt / Reversal |

### 2.3 Invoice Receipt & Three-Way Match (IR)

| ID | Scenario | Type | Risk | Justification | Traceability |
|---|---|---|---|---|---|
| IR-01 | Vendor invoice matches PO price and GR quantity exactly; posts without a payment block *(standard)* | Happy path | Medium | Core three-way-match path; regression affects every clean invoice, high volume impact if broken | Invoice Receipt |
| IR-02 | Invoice quantity or price differs from the goods receipt beyond tolerance — invoice is blocked for payment (GR/IR mismatch) | Edge | High | This is the central three-way-match control. If a regression weakens this block, mismatched invoices can be paid without review — direct financial exposure | Invoice Receipt / Three-Way Match |
| IR-03 | Duplicate invoice (same vendor, invoice number/reference, amount) is detected and blocked from double posting | Edge | High | Duplicate payment prevention is a standard SAP control (duplicate invoice check); a regression disabling it causes real cash leakage, not just a data-quality issue | Invoice Receipt / Duplicate Check |
| IR-04 | Invoice received and entered before the corresponding goods receipt is posted (invoice-first scenario) is held/parked per configured three-way-match rules rather than posted outright | Edge | Medium | Common timing variance in real procurement; confirm existing invoice-before-GR handling wasn't altered by the underlying change | Invoice Receipt |
| IR-05 | Invoice tax code differs from the PO's tax code; the system flags or corrects the mismatch per existing configuration rather than posting silently | Edge | Medium | Tax-code mismatches are a compliance-adjacent control; regression risk is moderate since domestic purchases typically have stable, well-tested tax logic | Invoice Receipt / Tax |

### 2.4 FI / AP Integration & Payment (FI)

| ID | Scenario | Type | Risk | Justification | Traceability |
|---|---|---|---|---|---|
| FI-01 | Posted invoice determines the correct GL account via existing automatic account determination, unchanged from the pre-change baseline *(standard)* | Happy path | High | Misrouted postings are a financial statement accuracy issue; even though this is "standard," it must be explicitly re-verified after any change touching MM-FI integration | Invoice → FI Posting |
| FI-02 | Payment run correctly selects due, unblocked open items and clears the vendor liability | Edge | Medium | Core AP process; well-proven in an existing system, but worth confirming the payment proposal logic wasn't affected by the change | Payment |
| FI-03 | Invoice blocked per IR-02 or IR-03 is correctly excluded from the payment run until the block is released | Edge | High | This is the enforcement point that makes the three-way-match block meaningful — if payment runs can select blocked invoices, the earlier control is cosmetic only | Payment / Block Enforcement |
| FI-04 | Withholding tax (if applicable to this client's domestic vendor population) calculates and posts correctly on the vendor invoice | Edge | ⚠ **SME REQUIRED** | Whether withholding tax applies to this client's domestic vendor base at all was not stated — flagging rather than assuming it applies or doesn't | Invoice → FI Posting (conditional) |

### 2.5 Regression-Specific: Change-Impact & Unaffected-Area Confirmation (RG)

| ID | Scenario | Type | Risk | Justification | Traceability |
|---|---|---|---|---|---|
| RG-01 | Process areas **not** touched by the underlying transport/change still complete the full PR→PO→GR→IR→Payment flow without new defects | Regression smoke | High | This is the fundamental regression-cycle question: did the change break something unrelated? Without the actual change scope, this must be tested broadly rather than narrowly targeted — see Section 6 | End-to-end (cross-area) |
| RG-02 | Existing custom Z-programs, user-exits, or BAdIs in the P2P flow (if any) continue to execute without error after the change | Edge | ⚠ **SME REQUIRED** | Whether custom code exists in this flow, and which of it is anywhere near the actual change, was not stated. The *risk category* (custom code regression) is valid test-design judgment; the *specific* code paths cannot be named without that visibility | Various (custom logic) |
| RG-03 | Existing interfaces/integration points in the P2P flow (e.g. a vendor EDI invoice feed, if one exists) continue to function post-transport | Edge | ⚠ **SME REQUIRED** | Whether this client has any interface in this flow at all was not stated as part of scope, and the constraint field was explicitly confirmed as "none" — this scenario is included as a standard regression-checklist item, not because an interface is known to exist | Various (integration) |

---

## 3. Entry / Exit Criteria

Criteria are scoped to a regression cycle on an existing, live system — not a new-implementation build-out.
No cutover/data-migration phase is included, since this system is already live.

### 3.1 Regression Test Cycle

**Entry criteria:**
- The transport(s)/configuration or code change(s) driving this regression cycle have been moved into the
  test environment, and the change log/transport list is available to the test team. ⚠ *If this is not
  yet available, entry should be held — testing "blind" against an unknown change scope undermines the
  entire purpose of a regression cycle (see Section 1's scoping gap).*
- Existing master data (vendor, material, contract/outline agreement records) in the test environment
  reflects current production-equivalent state, not stale or reset data.
- The baseline regression scenario set (Section 2, prioritized against the actual change scope once known)
  has been agreed with the business/functional leads.

**Exit criteria:**
- All High-risk scenarios in Section 2 (PR-03, PR-04, GR-03, IR-02, IR-03, FI-01, FI-03, RG-01) have
  executed with a Pass result, or have an open defect with documented business sign-off to proceed.
- All scenarios in the areas actually touched by the change (once the transport scope is known) have
  passed with no open Severity-1/2 defects.
- Smoke-level regression (RG-01) has been executed across the full P2P flow at least once, confirming no
  unrelated area was broken by the change.
- Defect log reviewed and triaged jointly by MM and FI functional leads, given the cross-module nature of
  GR/IR matching and payment blocking.

### 3.2 Post-Deployment Validation (Existing Live System)

**Entry criteria:**
- Regression Test Cycle exit criteria met and the change has been approved for production deployment.

**Exit criteria:**
- A defined smoke-test subset (recommend: PR-01, GR-01, IR-01, FI-01, FI-02 at minimum — the core happy-path
  chain) has been executed successfully in production immediately post-deployment.
- No new incident raised against the P2P process within the agreed post-deployment monitoring window.
- Go/no-go rollback decision point documented, with a named owner authorized to trigger rollback if the
  smoke test fails.

---

## 4. Risk Summary

| Process Area | High | Medium | Low | Total |
|---|---|---|---|---|
| Purchase Requisition & PO Creation (PR) | 2 | 1 | 1 | 4 |
| Goods Receipt (GR) | 1 | 2 | 0 | 3 |
| Invoice Receipt & Three-Way Match (IR) | 2 | 2 | 0 | 4 |
| FI / AP Integration & Payment (FI) | 2 | 1 | 0 | 3 (+1 flagged) |
| Regression-Specific (RG) | 1 | 0 | 0 | 1 (+2 flagged) |

**Observation:** Risk concentrates in the control-enforcement scenarios — vendor block (PR-03), approval
workflow (PR-04), tolerance/three-way-match blocking (GR-03, IR-02, IR-03), account determination (FI-01),
and payment-block enforcement (FI-03) — rather than in the happy-path scenarios themselves. This is
expected for a *regression* cycle: the happy path is already proven in production; what a regression cycle
must re-verify is that existing controls weren't silently weakened by the change. This reinforces why
knowing the actual change scope (Section 1) matters — these are exactly the areas most likely to be
affected if the underlying change touches pricing, tolerances, release strategy, or account determination
configuration.

---

## 5. Traceability Note

Each scenario ID is tagged with a **Traceability** reference to the P2P process step it validates
(PO Creation, Goods Receipt, Invoice Receipt, Payment, etc.), structured to seed a Requirements
Traceability Matrix: business process step → test scenario ID → (once written) detailed test case ID →
execution result. This document defines the scenario layer only; it does not contain step-level scripts.

---

## 6. Out-of-Scope / Requires SME Input

1. **Actual transport/change scope** — the single most important missing input for a regression cycle.
   This document could not target specific at-risk areas without it and instead provides the full standard
   P2P regression baseline. Recommend obtaining the change/transport log and re-prioritizing Section 2
   against it before finalizing execution scope.
2. **Withholding tax applicability (FI-04)** — not stated whether this client's domestic vendor population
   is subject to withholding tax.
3. **Custom code in the P2P flow (RG-02)** — existence and location of any Z-programs, user-exits, or
   BAdIs touching PR/PO/GR/IR/Payment was not stated. Recommend a technical change-impact review by the
   ABAP/functional leads to identify custom logic near the actual change.
4. **Interfaces in the P2P flow (RG-03)** — whether any exist (e.g. vendor EDI invoice feed, automated
   3-way-match tooling) was not stated; included above as a standard regression checklist item only.
5. **Exact tolerance and control configuration values** — where this document references standard SAP
   controls (e.g. GR quantity tolerance, invoice price tolerance, release strategy thresholds), it does not
   state the client's actual configured values, since none were supplied. Confirm actual settings (e.g.
   tolerance keys in OMR6, release strategy in PO release procedure config) with the client's MM/FI
   configuration team before finalizing detailed test cases.
6. **Scope exclusions assumed from "standard domestic goods purchase"** — this document excludes service
   POs, consignment, intercompany procurement, and import/cross-border duty scenarios because the client
   described the purchase profile as standard domestic goods. If any of those actually occur in this
   client's environment, they are not covered here and should be scoped as a separate exercise.

---

## Assumptions & Disclaimers

- This document is a **test scenario strategy**, not a scripted test case set.
- Standard transaction/configuration references (e.g. MIGO, MIRO, ME21N, OMR6) reflect documented standard
  SAP behavior for traceability/precision, not a claim about this client's specific configuration.
- Given the regression-cycle scoping gap in Section 1, this catalogue should be treated as a starting
  candidate pool and re-prioritized once the actual change/transport scope is available.
