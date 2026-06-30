# Domain Event Contract — Terra Commerce

**Document ID:** EVT-TC-001  
**Version:** 1.0.0  
**Status:** Final  
**Owner:** Backend Lead  
**Last Updated:** 2026-06-30  
**Upstream:** FSD v1.0.1, System Architecture v1.0.0 Final, State Machine v1.0.0 Final, Database Design v1.0.0 Final  
**Audit:** `reviews/Contract_Suite_Audit_v1.0.0.md`

## 1. Envelope

```json
{
  "event_id": "uuid",
  "event_type": "order.created",
  "schema_version": 1,
  "tenant_id": "uuid-or-null",
  "aggregate_type": "order",
  "aggregate_id": "uuid",
  "aggregate_version": 3,
  "occurred_at": "2026-06-30T00:00:00Z",
  "correlation_id": "uuid-or-null",
  "causation_id": "uuid-or-null",
  "actor": {"type":"user|customer|system|provider","id":"uuid-or-null"},
  "data": {}
}
```

## 2. Delivery and Ordering

- At-least-once delivery.
- `event_id` is globally unique.
- Consumers deduplicate by `event_id`.
- Ordering is guaranteed only per aggregate through increasing `aggregate_version`.
- Each consumer that depends on sequence stores the last successfully processed aggregate version.
- Duplicate or lower versions are idempotently ignored after validation.
- A version gap is retried, deferred, or quarantined according to consumer policy; it is never silently accepted when strict order is required.
- Outbox insertion occurs in the same transaction as the state change.
- Consumer acknowledgement occurs only after successful handling.

## 3. Schema Registry Requirement

A maintained event registry must record:

- event type
- schema version
- owning producer
- known consumers
- aggregate type
- tenant/global scope
- sensitivity classification
- required and optional fields
- compatibility policy
- deprecation status

An existing event name must never be reused for incompatible semantics.

## 4. Global Rules

- Names use past-tense dot notation.
- Tenant-owned events require `tenant_id`.
- `aggregate_version` equals the committed aggregate version.
- Payloads exclude secrets and unnecessary personal data.
- Breaking changes require a new incompatible schema version or event name.
- Redis Streams are transport, not the source of truth.

## 5. Event Catalog

### Tenant and platform

`tenant.created`, `tenant.status_changed`, `tenant.suspended`, `tenant.reactivated`, `tenant.plan_changed`, `tenant.owner_invited`, `tenant.owner_access_reset`.

### Catalog

`product.created`, `product.updated`, `product.published`, `product.deactivated`, `product.archived`, `variant.created`, `variant.updated`, `category.updated`.

### Customer, cart, checkout

`customer.registered`, `customer.status_changed`, `cart.created`, `cart.claimed`, `cart.updated`, `checkout.started`, `checkout.failed`.

### Order

`order.created`, `order.status_changed`, `order.payment_pending_verification`, `order.paid`, `order.processing_started`, `order.ready_for_fulfillment`, `order.cancellation_requested`, `order.cancelled`, `order.delivered`, `order.completed`, `order.failed`.

Order transition data includes order number, state before/after, transition ID, and optional reason code.

### Payment

`payment.started`, `payment.pending_verification`, `payment.authorized`, `payment.captured`, `payment.failed`, `payment.cancelled`, `payment.refund_pending`, `payment.partially_refunded`, `payment.refunded`.

Amounts use minor units and currency.

### Inventory

`inventory.reserved`, `inventory.released`, `inventory.consumed`, `inventory.reservation_expired`, `inventory.adjusted`, `inventory.received`, `inventory.low_stock`, `inventory.out_of_stock`.

Payloads identify inventory item, warehouse, SKU, quantity bucket, before/delta/after, and business reference where applicable.

### WMS

`picking.created`, `picking.started`, `picking.paused`, `picking.exception_reported`, `picking.completed`, `picking.cancelled`, `packing.created`, `packing.started`, `packing.exception_reported`, `packing.completed`, `packing.cancelled`.

### Shipment

`shipment.created`, `shipment.courier_assigned`, `shipment.awaiting_pickup`, `shipment.picked_up`, `shipment.in_transit`, `shipment.out_for_delivery`, `shipment.delivered`, `shipment.delivery_failed`, `shipment.returned_to_sender`, `shipment.cancelled`, `shipment.unknown_pending_review`, `shipment.reconciled`.

### Return and refund

`return.requested`, `return.approved`, `return.rejected`, `return.received`, `return.inspected`, `return.completed`, `return.cancelled`, `refund.requested`, `refund.approved`, `refund.rejected`, `refund.processing`, `refund.partially_completed`, `refund.completed`, `refund.failed`.

### Notification and reports

`notification.requested`, `notification.sent`, `notification.failed`, `report.requested`, `report.completed`, `report.failed`.

## 6. Consumer Registry

| Consumer | Event families |
|---|---|
| Notification worker | tenant, order, payment, inventory, WMS, shipment |
| SSE projector | tenant, order, inventory, WMS, shipment, notification |
| Reporting projector | order, payment, inventory, shipment |
| Delivery synchronizer | shipment/provider |
| Reconciliation tooling | critical state and provider conflicts |

Every consumer records supported schema versions, idempotency storage, ordering requirement, and dead-letter behavior.

## 7. Compatibility

Backward-compatible changes include optional field addition and new event types. A new enum value is compatible only when consumers are required to tolerate unknown values.

Breaking changes include required field removal, field type change, semantic identifier change, or incompatible event meaning.

## 8. Acceptance Criteria

- Every downstream-dependent committed transition has a durable event.
- Event and aggregate versions align.
- Strict-order consumers track aggregate versions.
- Version gaps are handled explicitly.
- Registry ownership and sensitivity are documented.
- Duplicate delivery is safe.

## Change Summary

Final version closes CONTRACT-M03 by adding consumer aggregate-version tracking, version-gap handling, and a mandatory event schema registry.
