# Technical Design Document — Medusa and Terra Commerce Modules

**Document ID:** TDD-TC-COM-001  
**Version:** 1.0.0  
**Status:** Final  
**Owner:** Commerce Backend Lead  
**Last Updated:** 2026-06-30  
**Upstream:** Final FSD, Architecture, State Machine, Database Design, Contract Suite, Authorization Matrix  
**Audit:** `reviews/Technical_Design_Suite_Audit_v1.0.0.md`

## 1. Ownership

Medusa owns products, variants, pricing, customers, carts, orders, promotions, payment primitives, inventory primitives, fulfillment, returns, and refund workflows.

Terra Commerce modules own tenant links/context, plan enforcement, business state projections, payment/fulfillment state extensions, OMS timeline and notes, inventory ledger and reservation behavior, receiving, picking, packing, warehouse exceptions, shipments, provider adapters, transition history, audit, and durable outbox.

Undocumented Medusa internals must not become public or cross-module contracts.

## 2. Module Layout

```text
src/modules/
  tenant-context
  plan-enforcement
  catalog-extension
  customer-extension
  order-extension
  payment-state
  fulfillment-state
  inventory-ledger
  receiving
  picking
  packing
  warehouse-exception
  shipment
  returns
  refunds
  timeline
  audit
  outbox
src/workflows
src/subscribers
src/integrations/{biteship,payment,object-storage}
src/repositories
src/policies
src/testkit
```

Each module owns interfaces, tables/migrations, validation, and tests.

## 3. Trusted Commerce Context

Every operation receives verified tenant, actor type/ID, correlation ID, operation ID, optional idempotency key, and expected version from the gateway trust contract.

Commerce independently validates service identity, context integrity/freshness, tenant ownership, and state. No unscoped repository method is permitted for tenant-owned data.

## 4. Repository and Transaction Rules

- Tenant-safe repository methods only.
- Composite tenant foreign keys where applicable.
- Mutable aggregates use version comparison or row lock.
- Transaction handle is explicit.
- Append-only repositories expose insert/read only.
- State write, transition history, audit where required, and outbox insert share one PostgreSQL transaction.
- External provider calls never occur inside long-held database transactions.

## 5. Catalog and Customer Extensions

Catalog extension maintains tenant-product/variant links, tenant-unique SKU, dimensions, publish checks, media references, plan limits, and archival.

Customer/cart extension maintains tenant customer profiles, normalized tenant-scoped email, anonymous cart claim tokens, and customer/cart ownership. Claiming rotates or invalidates the anonymous token. MVP checkout requires authentication.

## 6. Normative Checkout and Payment Pattern

MVP checkout uses this required sequence:

1. Validate active tenant, customer/cart ownership, products, prices, warehouse, shipping rate, plan, and features.
2. Acquire checkout idempotency.
3. Begin PostgreSQL transaction.
4. Lock inventory rows in deterministic order.
5. Create Active reservations and movement records.
6. Create Medusa order and Terra order extension.
7. Create payment-state extension in `pending` or `not_started` according to payment adapter.
8. Insert transition, audit where required, and durable outbox event.
9. Commit.
10. Initialize the external payment idempotently after commit.
11. In a second transaction, persist authorization/capture/failure and resulting order transition plus outbox event.

If payment initialization fails or times out, the committed order remains in a documented recoverable payment state. A retry uses the same payment idempotency identity. Reservation expiration/compensation follows policy.

A provider requiring pre-transaction payment creation needs a separate approved design and must not alter this default silently.

## 7. State Modules

Order, payment, fulfillment, reservation, picking, packing, shipment, return, and refund use separate authoritative versions where defined.

Transition service validates:

- tenant ownership
- permission decision context
- expected version
- source state and guards
- dual approval where required
- idempotency

It atomically writes aggregate, projection, transition, audit, and outbox.

## 8. Inventory and Receiving

Buckets:

- physical saleable
- reserved
- incoming
- damaged

Every mutation writes an append-only bucket movement.

Reservation locks the inventory row, verifies availability, increments reserved, creates Active reservation, and writes movement/outbox. Consume decrements physical and reserved. Release decrements reserved.

Receiving completion locks inventory rows in deterministic order, applies accepted/damaged/rejected equations, writes movements, marks completion by version, and records audit/transition/outbox. It is idempotent.

## 9. Picking, Packing, and Exceptions

One picking and one packing task per order in MVP.

Picking completion requires all lines confirmed and no open exceptions. Packing completion requires picking complete, verified items, positive package dimensions/weight, and no open exception.

Task assignment is enforced unless a manager override permission exists.

Exception resolution supports continue, inventory correction, item removal where allowed, partial fulfillment where approved, cancellation, or escalation. Every resolution is state guarded and audited.

## 10. Shipment and Biteship

Shipment creation validates payment, fulfillment, packing, package data, selected service, permission, version, and idempotency.

Recommended flow:

1. Persist local Shipment Pending and idempotency identity.
2. Commit.
3. Call Biteship with stable external reference.
4. Persist provider result in a new transaction.
5. Handle timeout as unknown outcome, not safe failure.
6. Reconcile before retrying creation when provider outcome is uncertain.

Biteship-specific fields, authentication, status mapping, and typed errors remain isolated in a versioned adapter and mapping registry.

## 11. Returns and Refunds

Return module stores request, items, review, receipt, inspection, and disposition.

Refund module validates captured/refunded totals, state, permission, version, idempotency, and configurable dual approval above threshold. Provider operations occur outside the local transaction; pending/completed/failed transitions are persisted with outbox events.

## 12. Timeline, Audit, Plan, and Feature Enforcement

Order timeline projects authoritative transitions and selected events with customer/internal visibility classes.

Audit stores real/effective actor, tenant, action, resource, sanitized before/after data, request context, and correlation ID.

Plan checks apply before limited resource creation and feature use. Hard limits reject; soft limits emit warnings/events.

## 13. Error and Observability Model

Typed domain errors cover validation, forbidden/ownership, not found, state conflict, invalid transition, plan limit, provider unavailable, and idempotency conflict. Public mapping belongs to the gateway.

Metrics include workflow latency, transition conflicts, inventory contention, WMS throughput, provider outcomes, outbox backlog, payment recovery, and reconciliation.

## 14. Testing

- Tenant-safe repository tests.
- State/transition policy tests.
- Checkout two-transaction payment tests.
- Payment timeout/retry and reservation compensation tests.
- Atomic state/transition/audit/outbox tests.
- Inventory concurrency tests.
- WMS lifecycle tests.
- Medusa compatibility tests.
- Biteship/payment adapter fixtures.
- End-to-end commerce flows.

## 15. Acceptance Criteria

1. Ownership boundaries are enforceable.
2. No unscoped tenant access exists.
3. Checkout uses the normative commit-then-payment pattern.
4. State, transition, audit, and outbox writes are atomic.
5. Inventory and WMS remain correct under concurrency.
6. Provider uncertainty cannot create duplicate financial or shipment effects.

## Change Summary

Final version closes TDD-M02 by defining the normative checkout commit and post-commit idempotent payment-initialization pattern.
