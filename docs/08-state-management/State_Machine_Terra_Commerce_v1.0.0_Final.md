# State Machine Specification — Terra Commerce

**Document ID:** SM-TC-001  
**Version:** 1.0.0  
**Status:** Final  
**Owner:** Product Owner and Technical Lead  
**Last Updated:** 2026-06-30  
**Upstream:** PRD v1.0.0 Final, FSD v1.0.1 Draft, System Architecture v1.0.0 Final  
**Audit:** `reviews/State_Machine_Audit_v1.0.0.md`

## 1. Purpose

This document defines authoritative states, allowed transitions, guards, side effects, concurrency rules, idempotency, and terminal conditions for Terra Commerce MVP.

Covered lifecycles:

- Tenant.
- Order.
- Payment.
- Inventory reservation.
- Fulfillment.
- Picking.
- Packing.
- Shipment and delivery.
- Return.
- Refund.
- Background jobs.

## 2. Global Transition Rules

1. Every tenant-owned transition requires validated tenant context.
2. Every transition validates actor permission, resource ownership, and business guards.
3. Invalid transitions return `invalid_state_transition`.
4. Every mutable aggregate uses an atomic version, compare-and-set, row lock, or equivalent concurrency guard.
5. Concurrent requests based on stale state return `state_conflict`.
6. Critical transitions are audited and durably published.
7. Duplicate transition requests are idempotent where callbacks, retries, or resubmission are possible.
8. Transition history is append-only.
9. Compensations are explicit transitions or actions; state is never silently rewound.
10. Terminal states cannot transition unless this document explicitly permits a related but separate lifecycle.

## 3. Tenant Lifecycle

### States

- Draft
- Trial
- Active
- Past Due
- Suspended
- Deactivated
- Archived

### Allowed Transitions

| From | To | Guard | Side Effects |
|---|---|---|---|
| Draft | Trial | owner and required base configuration exist | activate trial dates |
| Draft | Active | required configuration and plan exist | enable normal operations |
| Trial | Active | activation approved | enable active behavior |
| Trial | Past Due | trial expired and account unresolved | apply configured restrictions |
| Active | Past Due | billing or account rule triggered | warn or restrict according to plan policy |
| Active | Suspended | authorized action or severe violation | revoke sessions and terminate SSE |
| Past Due | Active | account resolved | restore operations |
| Past Due | Suspended | grace period expired or manual action | revoke sessions and terminate SSE |
| Suspended | Active | authorized reactivation | restore access |
| Suspended | Deactivated | authorized shutdown | block tenant operations |
| Active | Deactivated | authorized shutdown | revoke sessions |
| Past Due | Deactivated | authorized shutdown | revoke sessions |
| Deactivated | Active | authorized restoration and valid configuration | restore tenant |
| Deactivated | Archived | archival prerequisites satisfied | historical read-only state |

### Rules

- Archived is terminal in MVP.
- Suspension and deactivation block normal commerce mutations.
- Audited support-mode access is governed separately.

## 4. Order Lifecycle

### States

- Draft
- Pending Payment
- Payment Pending Verification
- Paid
- Processing
- Ready for Fulfillment
- Picking
- Packing
- Ready for Shipment
- Shipped
- Delivered
- Completed
- Cancellation Requested
- Cancelled
- Return Requested
- Returned
- Refund Pending
- Refunded
- Failed

### Primary Flow

```text
Draft
 -> Pending Payment
 -> Paid
 -> Processing
 -> Ready for Fulfillment
 -> Picking
 -> Packing
 -> Ready for Shipment
 -> Shipped
 -> Delivered
 -> Completed
```

### Allowed Transitions

| From | To | Guard |
|---|---|---|
| Draft | Pending Payment | order validation and reservation succeed |
| Pending Payment | Payment Pending Verification | manual payment flow selected or evidence submitted |
| Pending Payment | Paid | provider confirms payment |
| Payment Pending Verification | Paid | authorized verification succeeds |
| Pending Payment | Failed | unrecoverable pre-payment creation or validation failure |
| Payment Pending Verification | Failed | unrecoverable verification integrity failure requiring closure |
| Paid | Processing | order accepted for processing |
| Processing | Ready for Fulfillment | order remains fulfillable and reservation valid |
| Ready for Fulfillment | Picking | picking starts |
| Picking | Packing | picking completes with no unresolved exception |
| Packing | Ready for Shipment | packing completes and package data is valid |
| Ready for Shipment | Shipped | valid shipment exists and dispatch is accepted |
| Shipped | Delivered | shipment reaches canonical Delivered |
| Delivered | Completed | completion policy satisfied |
| Pending Payment | Cancelled | cancellation permitted before captured payment |
| Payment Pending Verification | Cancelled | authorized cancellation succeeds |
| Paid | Cancellation Requested | request submitted before irreversible fulfillment |
| Processing | Cancellation Requested | request submitted and work can still stop |
| Ready for Fulfillment | Cancellation Requested | picking has not started or can be stopped safely |
| Picking | Cancellation Requested | picking can be stopped and compensation plan exists |
| Packing | Cancellation Requested | packing can be stopped and package/shipment not finalized |
| Cancellation Requested | Cancelled | tasks stopped and inventory/payment compensations complete |
| Cancellation Requested | Processing | request rejected or withdrawn before fulfillment resumes |
| Cancellation Requested | Ready for Fulfillment | request rejected and order remains ready without active work |
| Delivered | Return Requested | return policy permits |
| Completed | Return Requested | return window remains open |
| Return Requested | Returned | accepted items are received |
| Returned | Refund Pending | approved refund is required |
| Cancelled | Refund Pending | captured amount requires refund |
| Refund Pending | Refunded | refund completes |

### Failure Rules

- Failed is restricted to unrecoverable pre-fulfillment integrity or payment-establishment failures.
- A paid, picking, packing, shipped, delivered, return, or refund state must not jump directly to Failed.
- Post-payment problems use cancellation, exception, return, refund, or manual reconciliation flows.

### Terminal Conditions

- Completed is terminal except a separately valid Return lifecycle may begin within policy.
- Cancelled is terminal when no refund is required.
- Refunded is terminal.
- Failed is terminal unless an approved recovery process creates a new order or explicitly reopens through a future versioned rule.

### Rules

- Order state is distinct from payment, fulfillment, and shipment state.
- No direct Paid-to-Shipped transition is allowed.
- Customer-visible status may be a projection of several internal lifecycles.

## 5. Payment Lifecycle

### States

- Not Started
- Pending
- Pending Verification
- Authorized
- Captured
- Failed
- Cancelled
- Refund Pending
- Partially Refunded
- Refunded

### Transitions

| From | To | Guard |
|---|---|---|
| Not Started | Pending | payment session created |
| Pending | Authorized | provider authorization succeeds |
| Pending | Captured | immediate capture succeeds |
| Pending | Pending Verification | manual payment flow selected |
| Pending Verification | Captured | authorized confirmation succeeds |
| Pending | Failed | provider reports failure |
| Pending | Cancelled | session expires or is cancelled |
| Authorized | Captured | capture succeeds |
| Authorized | Cancelled | authorization is voided |
| Captured | Refund Pending | approved refund begins |
| Refund Pending | Partially Refunded | partial refund succeeds |
| Refund Pending | Refunded | full refund succeeds |
| Partially Refunded | Refund Pending | additional refund begins |
| Partially Refunded | Refunded | remaining refundable amount completes |

### Rules

- Captured amount cannot exceed payable amount.
- Total refunded amount cannot exceed captured amount.
- Provider callbacks and manual confirmations are idempotent.
- Failed refund attempts remain in Refund Pending with failure metadata until retry or rejection through refund lifecycle.

## 6. Inventory Reservation Lifecycle

### States

- Pending
- Active
- Released
- Consumed
- Expired
- Failed

### Transitions

| From | To | Guard |
|---|---|---|
| Pending | Active | atomic reservation succeeds |
| Pending | Failed | insufficient stock or transaction failure |
| Active | Released | cancellation or explicit release succeeds |
| Active | Consumed | fulfillment deduction succeeds |
| Active | Expired | expiration policy applies and order is not protected |

### Rules

- Only Active reservations reduce available stock.
- Release, consume, and expire are idempotent.
- Released, Consumed, Expired, and Failed are terminal for that reservation record.
- Reactivation requires a new reservation record.

## 7. Fulfillment Lifecycle

### States

- Not Ready
- Ready
- In Progress
- Exception
- Completed
- Cancelled

### Transitions

| From | To | Guard |
|---|---|---|
| Not Ready | Ready | payment and order guards pass |
| Ready | In Progress | picking starts |
| In Progress | Exception | warehouse exception occurs |
| Exception | In Progress | authorized resolution permits continuation |
| Exception | Cancelled | resolution requires cancellation |
| In Progress | Completed | picking and packing complete |
| Ready | Cancelled | order cancels before work starts |
| In Progress | Cancelled | stoppable work and compensation complete |

## 8. Picking Lifecycle

### States

- Not Created
- Ready
- In Progress
- Paused
- Exception
- Completed
- Cancelled

### Transitions

| From | To | Guard |
|---|---|---|
| Not Created | Ready | order reaches Ready for Fulfillment |
| Ready | In Progress | authorized picker starts |
| In Progress | Paused | task pause requested |
| Paused | In Progress | authorized resume |
| In Progress | Exception | missing, damaged, wrong, or partial stock reported |
| Exception | In Progress | resolution permits continuation |
| In Progress | Completed | all required lines confirmed |
| Ready | Cancelled | order cancels before start |
| Paused | Cancelled | authorized cancellation and compensation succeed |
| Exception | Cancelled | exception resolution requires cancellation |
| In Progress | Cancelled | safe stop and compensation succeed |

### Rules

- Completed tasks are immutable except through an explicit correction workflow.
- Completion emits `picking.completed`.

## 9. Packing Lifecycle

### States

- Not Created
- Ready
- In Progress
- Exception
- Completed
- Cancelled

### Transitions

| From | To | Guard |
|---|---|---|
| Not Created | Ready | picking completes |
| Ready | In Progress | authorized packer starts |
| In Progress | Exception | mismatch or package problem occurs |
| Exception | In Progress | issue resolves |
| In Progress | Completed | items verified and package data valid |
| Ready | Cancelled | order cancels before packing |
| In Progress | Cancelled | safe stop and compensation succeed |
| Exception | Cancelled | resolution requires cancellation |

### Rules

- MVP supports one package per order.
- Completed packing requires final weight and dimensions.
- Completion emits `packing.completed`.

## 10. Shipment and Delivery Lifecycle

### States

- Shipment Pending
- Courier Assigned
- Awaiting Pickup
- Picked Up
- In Transit
- Out for Delivery
- Delivered
- Delivery Failed
- Returned to Sender
- Cancelled
- Unknown Pending Review

### Transitions

| From | To | Guard |
|---|---|---|
| Shipment Pending | Courier Assigned | provider assigns courier |
| Shipment Pending | Awaiting Pickup | provider accepts without separate assignment event |
| Courier Assigned | Awaiting Pickup | shipment is ready |
| Awaiting Pickup | Picked Up | pickup confirmed |
| Picked Up | In Transit | transit begins |
| In Transit | Out for Delivery | final-mile delivery begins |
| Out for Delivery | Delivered | delivery confirmed |
| Out for Delivery | Delivery Failed | attempt fails |
| In Transit | Delivery Failed | provider reports delivery failure |
| Delivery Failed | In Transit | provider retries and resumes transit |
| Delivery Failed | Returned to Sender | return begins or is confirmed |
| Awaiting Pickup | Cancelled | cancellation succeeds before pickup |
| Courier Assigned | Cancelled | cancellation succeeds before pickup |
| Any nonterminal provider-mapped state | Unknown Pending Review | status cannot be safely mapped |
| Unknown Pending Review | canonical state | authorized reconciliation or valid later event |

### Monotonic Event Rules

1. Duplicate events that map to the current state are idempotent.
2. Backward transitions are rejected unless explicitly listed above.
3. Delivered, Returned to Sender, and Cancelled are protected terminal shipment states.
4. A late provider timestamp does not override a later locally accepted state automatically.
5. Conflicting terminal events require Unknown Pending Review and operator reconciliation.
6. Every ignored, quarantined, or reconciled provider event is recorded.

### Rules

- Delivered may still lead to a separate Return lifecycle but does not reopen shipment transit.
- Unknown Pending Review blocks automatic downstream completion until reconciled.

## 11. Return Lifecycle

### States

- Requested
- Under Review
- Approved
- Rejected
- In Transit
- Received
- Inspected
- Completed
- Cancelled

### Transitions

| From | To | Guard |
|---|---|---|
| Requested | Under Review | accepted for evaluation |
| Under Review | Approved | policy and evidence pass |
| Under Review | Rejected | policy or evidence fails |
| Requested | Cancelled | customer withdraws before approval |
| Approved | In Transit | return shipment begins |
| Approved | Received | direct/manual receipt is allowed |
| In Transit | Received | warehouse confirms arrival |
| Received | Inspected | inspection is performed |
| Inspected | Completed | disposition recorded and refund path initiated if needed |
| Approved | Cancelled | cancellation permitted before shipment |

### Terminal States

- Rejected
- Completed
- Cancelled

## 12. Refund Lifecycle

### States

- Not Required
- Requested
- Approved
- Rejected
- Processing
- Partially Completed
- Completed
- Failed

### Transitions

| From | To | Guard |
|---|---|---|
| Not Required | Requested | cancellation or return requires refund |
| Requested | Approved | authorized approval |
| Requested | Rejected | policy rejects refund |
| Approved | Processing | provider or manual refund starts |
| Processing | Partially Completed | partial refund confirms |
| Processing | Completed | full refund confirms |
| Processing | Failed | refund attempt fails |
| Failed | Processing | authorized retry |
| Partially Completed | Processing | additional refund starts |
| Partially Completed | Completed | remaining amount completes |

### Rules

- Refund amount cannot exceed captured amount.
- Completed and Rejected are terminal for that refund request.
- Retry uses the same idempotency identity where required by provider rules.

## 13. Background Job Lifecycle

### States

- Queued
- Running
- Retry Scheduled
- Completed
- Failed
- Dead Lettered
- Cancelled

### Transitions

| From | To |
|---|---|
| Queued | Running |
| Running | Completed |
| Running | Retry Scheduled |
| Running | Failed |
| Retry Scheduled | Queued |
| Failed | Retry Scheduled |
| Failed | Dead Lettered |
| Queued | Cancelled |
| Retry Scheduled | Cancelled |

### Rules

- Retry count and next-attempt time are recorded.
- Completed jobs are not re-executed for the same idempotency identity.
- Dead-letter replay is explicit and audited.

## 14. Cross-Lifecycle Invariants

1. Order cannot become Ready for Fulfillment unless payment and inventory guards pass.
2. Picking cannot complete with unresolved exceptions.
3. Packing cannot begin before picking completes.
4. Shipment cannot be created before packing completes.
5. Order cannot become Shipped without a valid shipment.
6. Order cannot become Delivered unless shipment is Delivered.
7. Reservation cannot be both Consumed and Released.
8. Refund totals cannot exceed captured payment totals.
9. Tenant suspension blocks new operational transitions except authorized support, recovery, or deactivation actions.
10. Every transition validates tenant ownership and permission.
11. Cancellation from Picking or Packing requires safe-stop and compensation completion.
12. A shipment in Unknown Pending Review cannot automatically complete the order.

## 15. Transition Record and Event Requirements

Every successful transition records:

- Transition ID.
- Aggregate version before and after.
- Previous state.
- New state.
- Tenant ID where applicable.
- Resource ID.
- Actor or system source.
- Timestamp.
- Reason where required.
- Correlation ID.
- Idempotency key where applicable.

Published events also contain schema version. Critical transitions use durable event publication.

## 16. Error Behavior

- Invalid transition: `invalid_state_transition`.
- Stale aggregate version: `state_conflict`.
- Missing guard: domain-specific validation error.
- Duplicate idempotent request: return prior outcome.
- Unknown provider state: enter Unknown Pending Review and alert operators.

## 17. Downstream Requirements

This document governs:

- Database status fields, versions, constraints, and transition history.
- API commands and guards.
- Event schemas.
- Authorization matrix.
- UI action visibility.
- Automated state-transition and concurrency tests.
- Medusa-to-business-state mapping.
- Operational reconciliation tools.

## 18. Acceptance Criteria

1. Every lifecycle has explicit states and transitions.
2. Terminal states are identified.
3. Invalid and stale transitions are rejected atomically.
4. Cross-lifecycle invariants prevent impossible combinations.
5. External callbacks and retries are idempotent.
6. Tenant and permission checks apply to every transition.
7. Critical transitions are audited and durably published.
8. Cancellation during fulfillment is controlled and compensated.
9. Out-of-order provider events follow monotonic rules.
10. Unknown states are quarantined safely.

## Change Summary

Final version incorporates all findings from `State_Machine_Audit_v1.0.0.md`: restricted order failure transitions, controlled cancellation during picking and packing, mandatory atomic concurrency guards, and explicit monotonic shipment-event rules.
