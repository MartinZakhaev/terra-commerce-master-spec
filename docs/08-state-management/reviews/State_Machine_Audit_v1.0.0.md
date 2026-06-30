# State Machine Specification Audit

**Document ID:** REVIEW-TC-006  
**Version:** 1.0.0  
**Status:** Completed  
**Audit Date:** 2026-06-30  
**Reviewed Document:** `State_Machine_Terra_Commerce_v1.0.0_Draft.md`  
**Upstream:** PRD v1.0.0 Final, FSD v1.0.1 Draft, System Architecture v1.0.0 Final

## 1. Executive Summary

The draft provides broad and coherent lifecycle coverage for tenants, orders, payments, inventory reservations, fulfillment, picking, packing, shipments, returns, refunds, and background jobs.

No critical contradiction was found. Four medium-priority issues require correction before finalization.

**Draft verdict:** Pass with required corrections.

## 2. Findings

### SM-M01 — Order Failure Transition Is Too Broad

The draft allows any nonterminal order state to transition directly to Failed. This could incorrectly bypass cancellation, refund, shipment, or reconciliation obligations.

**Required correction:** Restrict Failed to explicitly defined unrecoverable pre-fulfillment or integrity conditions. Post-payment and post-shipment failures must enter exception, cancellation, return, or refund paths rather than silently terminate.

### SM-M02 — Cancellation During Picking and Packing Is Incomplete

The FSD allows stoppable fulfillment and warehouse exception resolution to cancel an order. The draft does not define controlled cancellation from Picking or Packing.

**Required correction:** Add transitions from Picking and Packing to Cancellation Requested only when tasks can be stopped safely, with inventory and package compensation guards.

### SM-M03 — Concurrency Control Is Not Mandatory

The draft defines `state_conflict` but does not require an optimistic version, compare-and-set, or equivalent concurrency guard.

**Required correction:** Require every mutable state aggregate to use a version or equivalent atomic transition guard so two concurrent commands cannot both succeed from the same prior state.

### SM-M04 — Shipment Out-of-Order Handling Needs Monotonic Rules

The draft mentions monotonic-state rules without defining them. Provider events can arrive late or duplicated.

**Required correction:** Define that backward transitions are rejected unless explicitly allowed, duplicate states are idempotent, terminal delivery states are protected, and provider timestamps alone cannot override a later locally accepted state without reconciliation.

## 3. Confirmed Coverage

| Lifecycle | Result |
|---|---|
| Tenant | Pass |
| Order | Pass with corrections |
| Payment | Pass |
| Inventory reservation | Pass |
| Fulfillment | Pass |
| Picking | Pass |
| Packing | Pass |
| Shipment and delivery | Pass with correction |
| Return | Pass |
| Refund | Pass |
| Background jobs | Pass |
| Tenant isolation | Pass |
| Idempotency | Pass |
| Durable events | Pass |
| Auditability | Pass |

## 4. Finalization Gate

The document may become Final after SM-M01 through SM-M04 are incorporated.

No new ADR is required because these changes clarify transition safety and concurrency rather than alter architecture decisions.

## 5. Audit Result

- Critical findings: **0**
- High findings: **0**
- Medium findings: **4**
- Material contradictions: **0**

**Required action:** publish a corrected final state-machine specification.
