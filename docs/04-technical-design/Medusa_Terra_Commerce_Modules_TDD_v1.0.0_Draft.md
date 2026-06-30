# Technical Design Document — Medusa and Terra Commerce Modules

**Document ID:** TDD-TC-COM-001  
**Version:** 1.0.0  
**Status:** Draft  
**Owner:** Commerce Backend Lead  
**Last Updated:** 2026-06-30  
**Upstream:** FSD v1.0.1, System Architecture v1.0.0 Final, State Machine v1.0.0 Final, Database Design v1.0.0 Final, Contract Suite v1.0.0 Final, Authorization Matrix v1.0.0 Final

## 1. Purpose

This document defines the technical implementation of commerce, OMS, WMS, inventory, delivery, return, refund, and Medusa extension modules.

## 2. Design Boundaries

Medusa owns core product, variant, customer, cart, order, pricing, promotion, payment, inventory primitive, fulfillment, return, and refund capabilities.

Terra Commerce modules own:

- tenant-aware extensions and links
- business-state projections
- plan and feature checks
- OMS timelines and internal notes
- inventory movement ledger and reservations
- receiving, picking, packing, and exceptions
- Biteship orchestration
- durable transition and outbox records

No module may depend on undocumented Medusa database internals.

## 3. Proposed Module Structure

```text
src/
  modules/
    tenant-context/
    plan-enforcement/
    catalog-extension/
    customer-extension/
    order-extension/
    payment-state/
    fulfillment-state/
    inventory-ledger/
    receiving/
    picking/
    packing/
    warehouse-exception/
    shipment/
    returns/
    refunds/
    timeline/
    audit/
    outbox/
  workflows/
  subscribers/
  integrations/
    biteship/
    payment/
    object-storage/
  repositories/
  policies/
  testkit/
```

Each module exposes application-level interfaces and owns its tables and migrations.

## 4. Tenant Context Propagation

Every command and query receives an immutable context:

```ts
type CommerceContext = {
  tenantId: string
  principalId?: string
  principalType: 'user' | 'customer' | 'system'
  correlationId: string
  idempotencyKey?: string
  expectedVersion?: number
}
```

Rules:

- No repository operation runs without tenant context unless the data is explicitly global.
- Medusa IDs are resolved through tenant link tables.
- Tenant ownership is validated before state and business logic.
- Background workflows persist tenant context.

## 5. Repository Pattern

Repositories expose tenant-safe methods, for example:

```ts
getOrderById(ctx, orderId)
updateOrderState(ctx, orderId, expectedVersion, transition)
listInventory(ctx, filters)
```

Requirements:

- composite tenant-safe foreign keys where applicable
- version checks on mutable aggregates
- transaction object passed explicitly
- no generic unscoped `findById`
- append-only repositories expose insert/read only

## 6. Transaction Coordinator

A workflow transaction contains:

- state validation
- domain writes
- state transition record
- audit record where required
- outbox event

All occur in one PostgreSQL transaction. External provider calls occur outside local transactions and use pending states plus compensation or retry.

## 7. Catalog Extension

Responsibilities:

- tenant-product and tenant-variant links
- tenant-unique SKU
- dimensions and weight
- publish validation
- plan product-limit check
- image references
- archive behavior

Publishing workflow:

1. Resolve tenant product.
2. Validate variant, price, dimensions, category, and media rules.
3. Check plan/feature availability.
4. Publish through supported Medusa API.
5. Update Terra projection/version.
6. Record audit and outbox event.

## 8. Customer and Cart Extension

Responsibilities:

- tenant-scoped customer profile
- tenant-scoped normalized email
- anonymous cart ownership token
- authenticated cart claim
- customer ownership validation

MVP checkout requires authenticated customer. Claiming a cart rotates or invalidates the anonymous claim token.

## 9. Checkout Workflow

Steps:

1. Validate tenant Active/Trial eligibility.
2. Validate authenticated customer and cart ownership.
3. Acquire idempotency record.
4. Reprice and validate active products.
5. Validate shipping rate and warehouse configuration.
6. Lock affected inventory items.
7. Create Active reservations.
8. Create Medusa order and Terra order extension.
9. Create payment-state extension.
10. Insert transitions and outbox event.
11. Initialize payment outside or after local commit according to provider design.
12. Persist recoverable payment state.

Failure guarantees:

- no duplicate order for one idempotency identity
- no orphan active reservation without order reference
- compensation releases reservations when committed order cannot continue

## 10. Order and Payment State Modules

`order_extensions`, `payment_state_extensions`, and `fulfillment_state_extensions` are separate versioned aggregates.

Transition service requires:

- current state
- expected version
- allowed transition
- permission decision from trusted gateway context
- reason and actor when required

The service writes state, projection, transition record, audit, and outbox atomically.

## 11. Inventory Ledger

Quantity buckets:

- physical saleable
- reserved
- incoming
- damaged

Every mutation produces an append-only movement specifying bucket, before, delta, after, reference, actor, and correlation ID.

Reservation algorithm:

1. Lock inventory row.
2. Verify `physical - reserved >= requested`.
3. Increment reserved.
4. Insert Active reservation.
5. Insert movement.
6. Insert outbox event where required.

Consume decrements physical and reserved atomically. Release decrements reserved only.

## 12. Receiving Module

Draft receiving records may be edited until completion.

Completion:

- validates lines and expected totals
- locks all inventory rows in deterministic order
- applies incoming/physical/damaged bucket equations
- writes movement per bucket change
- sets receiving completed with version check
- records audit, transition, and outbox

Completion is idempotent.

## 13. Picking Module

One picking task per order in MVP.

Core operations:

- create from `order.ready_for_fulfillment`
- assign/start
- pause/resume
- confirm line
- report exception
- cancel with compensation
- complete

Completion requires every required line confirmed and no open exception. Task assignment is enforced unless manager override exists.

## 14. Packing Module

One packing task and package per order.

Completion requires:

- picking completed
- all items verified
- positive final package dimensions and weight
- no unresolved packing exception

Completion transitions order to Ready for Shipment and emits durable event.

## 15. Warehouse Exception Module

Exception types:

- missing stock
- damaged stock
- wrong SKU
- partial quantity
- package mismatch

Resolutions:

- continue
- adjust inventory
- remove item where policy allows
- partial fulfillment where approved
- cancel
- escalate

Resolution commands are permission and state guarded and audited.

## 16. Shipment Module

Shipment creation:

1. Validate payment Captured or approved equivalent.
2. Validate fulfillment and packing complete.
3. Validate package and selected rate.
4. Acquire shipment idempotency.
5. Persist Shipment Pending locally.
6. Call Biteship through adapter.
7. Persist provider identifiers and resulting state.
8. Insert transition/outbox.

Timeout handling must distinguish unknown provider outcome from confirmed failure. Reconciliation queries or idempotent provider references prevent duplicate shipment creation.

## 17. Biteship Adapter

Adapter responsibilities:

- map normalized rate request/response
- map shipment request/response
- map official webhook payload to normalized fields
- verify provider authenticity
- map provider status through versioned registry
- expose typed provider errors

Provider-specific field names remain isolated from domain modules.

## 18. Return and Refund Modules

Return module stores request, items, review, receipt, inspection, and disposition.

Refund module:

- validates captured amount and prior refunds
- supports approval and dual control above threshold
- acquires idempotency
- calls payment adapter
- persists pending/completed/failed states
- emits payment and refund events

Provider failure never silently changes requested amount.

## 19. Timeline and Audit

Order timeline is a customer/internal projection built from authoritative transitions and selected events.

Audit records contain actor, effective tenant, action, resource, sanitized before/after data, IP/user-agent context from gateway, and correlation ID.

Audit and transition records are append-only.

## 20. Plan and Feature Enforcement

Enforcement service resolves effective plan and tenant overrides.

Checks are applied before:

- product creation
- user invitation
- report generation
- SSE connection allocation
- optional modules and features

Hard limit rejects. Soft limit permits with warning/event.

## 21. Error Model

Domain errors are typed and mapped by gateway:

- validation
- ownership/forbidden
- not found
- state conflict
- invalid transition
- plan limit
- provider unavailable
- idempotency conflict

No provider or database internals leak through public errors.

## 22. Observability

Module metrics:

- workflow latency and failures
- state conflicts
- reservation contention
- receiving/picking/packing throughput
- provider request outcomes
- outbox backlog
- reconciliation count

Logs include tenant, aggregate, operation, correlation ID, and outcome without secrets or excessive personal data.

## 23. Testing

- Unit tests for policies and transition guards.
- Repository tests for tenant-safe access.
- Transaction tests for state + transition + outbox atomicity.
- Inventory race tests.
- Idempotency and retry tests.
- Medusa extension compatibility tests.
- Contract tests for gateway-facing DTOs.
- Provider adapter fixtures.
- End-to-end checkout, cancellation, receiving, fulfillment, shipment, return, and refund tests.

## 24. Acceptance Criteria

1. Medusa and Terra ownership boundaries are explicit.
2. No unscoped tenant repository method exists.
3. State changes, transitions, audit, and outbox are atomic.
4. Inventory arithmetic remains valid under concurrency.
5. WMS workflows match final state machines.
6. Biteship details remain adapter-isolated.
7. Financial actions enforce amount, permission, idempotency, and dual-control rules.

## Change Summary

Initial Medusa and Terra Commerce modules technical design synchronized with finalized functional, state, data, contract, and security documents.
