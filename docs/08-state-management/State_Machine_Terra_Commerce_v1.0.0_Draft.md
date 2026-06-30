# State Machine Specification — Terra Commerce

**Document ID:** SM-TC-001  
**Version:** 1.0.0  
**Status:** Draft  
**Owner:** Product Owner and Technical Lead  
**Last Updated:** 2026-06-30  
**Upstream:** PRD v1.0.0 Final, FSD v1.0.1 Draft, System Architecture v1.0.0 Final

## 1. Purpose

This document defines authoritative lifecycle states, allowed transitions, guards, side effects, idempotency expectations, and terminal conditions for Terra Commerce MVP.

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

## 2. Global State Rules

1. Every transition is tenant scoped where applicable.
2. Invalid transitions return a normalized `invalid_state_transition` error.
3. Transitions are server-side enforced.
4. Critical transitions are audited.
5. Duplicate transition requests must be idempotent where external callbacks, retries, or user resubmission are possible.
6. State changes publish durable events where downstream behavior depends on them.
7. Historical transition records are append-only.
8. Compensating transitions must be explicit; state must never be silently rewound.

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

| From | To | Guard | Primary Side Effects |
|---|---|---|---|
| Draft | Trial | owner and base configuration exist | activate trial dates, emit `tenant.status_changed` |
| Draft | Active | required configuration complete | enable normal operations |
| Trial | Active | activation approved | enable paid/active behavior |
| Trial | Past Due | trial expired and no active plan/payment resolution | restrict configured operations |
| Active | Past Due | billing or plan condition triggered | warning/restriction policy |
| Active | Suspended | authorized global action or severe policy violation | revoke sessions, terminate SSE |
| Past Due | Active | account resolved | restore normal operations |
| Past Due | Suspended | grace period expired or manual action | revoke sessions, terminate SSE |
| Suspended | Active | authorized reactivation | restore access, emit reactivation event |
| Suspended | Deactivated | authorized shutdown | block tenant operations |
| Active | Deactivated | explicit authorized action | revoke sessions, stop normal operations |
| Past Due | Deactivated | explicit authorized action | revoke sessions |
| Deactivated | Active | authorized restoration and valid configuration | restore tenant |
| Deactivated | Archived | archival prerequisites satisfied | make tenant historical/read-only |

### Terminal State

- Archived

### Rules

- Archived cannot return to an operational state in MVP.
- Suspension and deactivation invalidate tenant-user sessions and SSE connections.
- Support-mode access remains separately authorized and audited.

## 4. Order Lifecycle

### Business States

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

### Main Flow

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
| Pending Payment | Payment Pending Verification | manual payment evidence submitted or manual method used |
| Pending Payment | Paid | provider confirms payment |
| Payment Pending Verification | Paid | authorized verification succeeds |
| Pending Payment | Failed | unrecoverable payment or order creation failure |
| Paid | Processing | order accepted for processing |
| Processing | Ready for Fulfillment | inventory remains reserved and order is fulfillable |
| Ready for Fulfillment | Picking | picking task started |
| Picking | Packing | picking completed without unresolved exception |
| Packing | Ready for Shipment | packing completed and package data valid |
| Ready for Shipment | Shipped | shipment created and accepted for dispatch |
| Shipped | Delivered | canonical delivery state becomes Delivered |
| Delivered | Completed | completion policy satisfied |
| Pending Payment | Cancelled | cancellation allowed before payment |
| Payment Pending Verification | Cancelled | cancellation approved |
| Paid | Cancellation Requested | customer or staff requests cancellation |
| Processing | Cancellation Requested | cancellation requested before irreversible fulfillment |
| Ready for Fulfillment | Cancellation Requested | picking not yet started or stoppable |
| Cancellation Requested | Cancelled | cancellation approved and compensations complete |
| Cancellation Requested | Processing | cancellation rejected or withdrawn |
| Delivered | Return Requested | return policy permits |
| Completed | Return Requested | return window remains open |
| Return Requested | Returned | returned items received and accepted |
| Returned | Refund Pending | refund required and approved |
| Cancelled | Refund Pending | captured payment requires refund |
| Refund Pending | Refunded | refund provider or manual process confirms completion |
| Any nonterminal state | Failed | unrecoverable integrity or processing failure requiring manual review |

### Terminal States

- Completed
- Cancelled, unless refund is required
- Refunded
- Failed, unless manually recovered through an approved process

### Rules

- Order business state is distinct from payment, fulfillment, and shipment sub-states.
- A later mapping document must map Medusa internal states to these business states.
- No direct transition from Paid to Shipped is allowed.
- Customer-visible status may be a projection of multiple internal states.

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

### Allowed Transitions

| From | To | Guard |
|---|---|---|
| Not Started | Pending | payment session created |
| Pending | Authorized | provider authorization succeeds |
| Pending | Captured | provider captures immediately |
| Pending | Pending Verification | manual payment flow selected |
| Pending Verification | Captured | authorized staff confirms payment |
| Pending | Failed | provider failure |
| Pending | Cancelled | payment session cancelled or expired |
| Authorized | Captured | capture succeeds |
| Authorized | Cancelled | authorization voided |
| Captured | Refund Pending | approved refund initiated |
| Refund Pending | Partially Refunded | partial refund succeeds |
| Refund Pending | Refunded | full refund succeeds |
| Partially Refunded | Refund Pending | additional refund initiated |
| Partially Refunded | Refunded | remaining refundable amount completed |

### Rules

- Captured amount cannot exceed order payable amount.
- Total refunded amount cannot exceed captured amount.
- Payment callbacks and manual confirmation are idempotent.
- Failed refund attempts remain Refund Pending with failure metadata until retry or rejection.

## 6. Inventory Reservation Lifecycle

### States

- Pending
- Active
- Released
- Consumed
- Expired
- Failed

### Allowed Transitions

| From | To | Guard |
|---|---|---|
| Pending | Active | atomic reservation succeeds |
| Pending | Failed | insufficient stock or transaction failure |
| Active | Released | order cancelled or reservation explicitly released |
| Active | Consumed | fulfillment deduction completes |
| Active | Expired | expiration policy reached and order not protected from expiry |
| Expired | Active | not allowed in MVP; create a new reservation instead |

### Rules

- Only Active reservations reduce available stock.
- Release, consume, and expire operations are idempotent.
- Consumed, Released, and Expired are terminal for that reservation record.

## 7. Fulfillment Lifecycle

### States

- Not Ready
- Ready
- In Progress
- Exception
- Completed
- Cancelled

### Allowed Transitions

| From | To | Guard |
|---|---|---|
| Not Ready | Ready | payment and order guards satisfied |
| Ready | In Progress | picking starts |
| In Progress | Exception | stock or warehouse exception reported |
| Exception | In Progress | authorized resolution permits continuation |
| Exception | Cancelled | cancellation resolution chosen |
| In Progress | Completed | picking and packing complete |
| Ready | Cancelled | order cancelled before fulfillment starts |

## 8. Picking Lifecycle

### States

- Not Created
- Ready
- In Progress
- Paused
- Exception
- Completed
- Cancelled

### Allowed Transitions

| From | To | Guard |
|---|---|---|
| Not Created | Ready | order enters Ready for Fulfillment |
| Ready | In Progress | authorized picker starts task |
| In Progress | Paused | picker pauses task |
| Paused | In Progress | same or reassigned picker resumes |
| In Progress | Exception | missing, damaged, wrong, or partial stock reported |
| Exception | In Progress | exception resolved for continuation |
| In Progress | Completed | all required lines confirmed |
| Ready | Cancelled | order cancelled before start |
| Paused | Cancelled | authorized cancellation |
| Exception | Cancelled | resolution requires cancellation |

### Rules

- Completed tasks are immutable except through explicit correction workflow.
- Completion emits `picking.completed`.

## 9. Packing Lifecycle

### States

- Not Created
- Ready
- In Progress
- Exception
- Completed
- Cancelled

### Allowed Transitions

| From | To | Guard |
|---|---|---|
| Not Created | Ready | picking completed |
| Ready | In Progress | authorized packer starts |
| In Progress | Exception | mismatch or package issue found |
| Exception | In Progress | issue resolved |
| In Progress | Completed | items verified and package data valid |
| Ready | Cancelled | order cancelled before packing |
| Exception | Cancelled | resolution requires cancellation |

### Rules

- MVP supports one package per order.
- Completed packing requires final weight and dimensions.
- Completion emits `packing.completed`.

## 10. Shipment and Delivery Lifecycle

### Canonical States

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

### Allowed Transitions

| From | To | Guard |
|---|---|---|
| Shipment Pending | Courier Assigned | provider assigns courier |
| Shipment Pending | Awaiting Pickup | provider accepts shipment without explicit assignment event |
| Courier Assigned | Awaiting Pickup | shipment ready for pickup |
| Awaiting Pickup | Picked Up | courier confirms pickup |
| Picked Up | In Transit | transit begins |
| In Transit | Out for Delivery | last-mile delivery begins |
| Out for Delivery | Delivered | proof or provider confirmation received |
| Out for Delivery | Delivery Failed | delivery attempt fails |
| In Transit | Delivery Failed | provider reports failure |
| Delivery Failed | In Transit | provider retries delivery and resumes transit |
| Delivery Failed | Returned to Sender | return process begins/completes |
| Awaiting Pickup | Cancelled | shipment cancellation succeeds before pickup |
| Courier Assigned | Cancelled | provider cancellation succeeds before pickup |
| Any state | Unknown Pending Review | provider state cannot be mapped safely |
| Unknown Pending Review | mapped canonical state | authorized reconciliation or later valid event |

### Rules

- Duplicate provider events do not duplicate side effects.
- Out-of-order events are ignored, quarantined, or reconciled according to monotonic-state rules.
- Delivered is terminal except for later return workflow, which is represented separately.

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

### Allowed Transitions

| From | To | Guard |
|---|---|---|
| Requested | Under Review | request accepted for evaluation |
| Under Review | Approved | policy and evidence pass |
| Under Review | Rejected | policy or evidence fails |
| Requested | Cancelled | customer withdraws before approval |
| Approved | In Transit | return shipment or handover begins |
| Approved | Received | manual in-person receipt allowed |
| In Transit | Received | warehouse confirms arrival |
| Received | Inspected | inspection starts/completes |
| Inspected | Completed | disposition recorded and any refund path initiated |
| Approved | Cancelled | approved request cancelled before shipment where allowed |

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

### Allowed Transitions

| From | To | Guard |
|---|---|---|
| Not Required | Requested | cancellation or return requires refund |
| Requested | Approved | authorized approval |
| Requested | Rejected | policy rejects refund |
| Approved | Processing | provider or manual refund initiated |
| Processing | Partially Completed | partial refund confirmed |
| Processing | Completed | full refund confirmed |
| Processing | Failed | provider/manual process fails |
| Failed | Processing | authorized retry |
| Partially Completed | Processing | additional refund initiated |
| Partially Completed | Completed | remaining amount completed |

### Rules

- Refund amount cannot exceed captured amount.
- Completed and Rejected are terminal for the request unless a new refund request is created.
- Refund processing is idempotent.

## 13. Background Job Lifecycle

### States

- Queued
- Running
- Retry Scheduled
- Completed
- Failed
- Dead Lettered
- Cancelled

### Allowed Transitions

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
- Completed jobs are not re-executed unless a new idempotency identity is created.
- Dead-letter replay is an explicit audited operation.

## 14. Cross-State Invariants

1. Order cannot enter Ready for Fulfillment unless payment and inventory guards pass.
2. Picking cannot complete with unresolved exceptions.
3. Packing cannot complete before picking completes.
4. Shipment cannot be created before packing completes.
5. Order cannot become Shipped without a valid shipment record.
6. Order cannot become Delivered unless shipment is Delivered.
7. Inventory reservation cannot be both Consumed and Released.
8. Refund totals cannot exceed captured payment totals.
9. Tenant suspension blocks new operational transitions except authorized support, recovery, or deactivation actions.
10. Every state-changing command validates tenant ownership and permission.

## 15. Event Requirements

Every successful state transition emits or records:

- Transition ID.
- Previous state.
- New state.
- Tenant ID where applicable.
- Resource ID.
- Actor or system source.
- Timestamp.
- Reason where required.
- Correlation ID.
- Schema version for published events.

Critical transitions use durable event publication.

## 16. Error Behavior

- Invalid transition: `invalid_state_transition`.
- Stale version or concurrent update: `state_conflict`.
- Missing guard: domain-specific validation error.
- Duplicate idempotent request: return prior successful outcome.
- Unknown provider state: map to Unknown Pending Review and alert authorized operators.

## 17. Downstream Requirements

This specification governs:

- Database status fields and constraints.
- API operations.
- Event schemas.
- Authorization matrix.
- UI actions and visibility.
- Automated transition tests.
- Medusa-to-business-state mapping.
- Operational reconciliation tools.

## 18. Acceptance Criteria

1. Every lifecycle has explicit states and allowed transitions.
2. Terminal states are identified.
3. Invalid transitions are rejected server-side.
4. Cross-state invariants prevent impossible business combinations.
5. External callbacks are idempotent.
6. Tenant and permission checks apply to every transition.
7. Critical transitions are audited and durably published.
8. Provider-unknown states are quarantined safely.

## Change Summary

Initial consolidated state-machine draft derived from the approved PRD, reviewed FSD, and final System Architecture.
