# Domain Event Contract — Terra Commerce

**Document ID:** EVT-TC-001  
**Version:** 1.0.0  
**Status:** Draft  
**Owner:** Backend Lead  
**Last Updated:** 2026-06-30  
**Upstream:** FSD v1.0.1, System Architecture v1.0.0 Final, State Machine v1.0.0 Final, Database Design v1.0.0 Final

## 1. Purpose

This contract defines internal domain events published through the durable outbox and Redis Streams. Events notify consumers of committed business state changes; PostgreSQL remains authoritative.

## 2. Event Envelope

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
  "actor": {
    "type": "user|customer|system|provider",
    "id": "uuid-or-null"
  },
  "data": {}
}
```

## 3. Global Rules

- Event names use past-tense dot notation.
- `event_id` is globally unique.
- `aggregate_version` matches the committed state version.
- Tenant-owned events require `tenant_id`.
- Events contain only data needed by consumers.
- Secrets and unnecessary personal data are prohibited.
- Consumers are idempotent by `event_id`.
- Ordering is guaranteed only per aggregate/version sequence, not globally.
- Breaking schema changes increment major schema version or event name.
- Outbox insertion occurs in the same transaction as the business state change.

## 4. Delivery Semantics

- At-least-once delivery.
- Consumer acknowledges only after successful handling.
- Duplicate delivery is expected.
- Retry uses bounded backoff.
- Exhausted events enter dead-letter handling.
- Consumers must reject or quarantine impossible aggregate-version regressions.

## 5. Tenant Events

- `tenant.created`
- `tenant.status_changed`
- `tenant.suspended`
- `tenant.reactivated`
- `tenant.plan_changed`
- `tenant.owner_invited`
- `tenant.owner_access_reset`

Required data includes tenant ID, previous/new status where applicable, plan references where applicable, and effective timestamp.

## 6. Catalog Events

- `product.created`
- `product.updated`
- `product.published`
- `product.deactivated`
- `product.archived`
- `variant.created`
- `variant.updated`
- `category.updated`

Catalog events expose stable public/operational references, not raw Medusa internals.

## 7. Customer and Cart Events

- `customer.registered`
- `customer.status_changed`
- `cart.created`
- `cart.claimed`
- `cart.updated`
- `checkout.started`
- `checkout.failed`

Cart events should avoid full line-item payloads unless a consumer explicitly requires them.

## 8. Order Events

- `order.created`
- `order.status_changed`
- `order.payment_pending_verification`
- `order.paid`
- `order.processing_started`
- `order.ready_for_fulfillment`
- `order.cancellation_requested`
- `order.cancelled`
- `order.delivered`
- `order.completed`
- `order.failed`

Required state-change fields:

```json
{
  "order_number": "TC-001",
  "state_before": "paid",
  "state_after": "processing",
  "reason_code": "optional",
  "transition_id": "uuid"
}
```

## 9. Payment Events

- `payment.started`
- `payment.pending_verification`
- `payment.authorized`
- `payment.captured`
- `payment.failed`
- `payment.cancelled`
- `payment.refund_pending`
- `payment.partially_refunded`
- `payment.refunded`

Monetary payloads use minor units and currency. Provider secrets and full payment instruments are prohibited.

## 10. Inventory Events

- `inventory.reserved`
- `inventory.released`
- `inventory.consumed`
- `inventory.reservation_expired`
- `inventory.adjusted`
- `inventory.received`
- `inventory.low_stock`
- `inventory.out_of_stock`

Inventory payloads include inventory item, SKU, warehouse, changed bucket, delta, before, after, and reference where applicable.

## 11. WMS Events

- `picking.created`
- `picking.started`
- `picking.paused`
- `picking.exception_reported`
- `picking.completed`
- `picking.cancelled`
- `packing.created`
- `packing.started`
- `packing.exception_reported`
- `packing.completed`
- `packing.cancelled`

Exception events include type and reference, not unrestricted evidence blobs.

## 12. Shipment Events

- `shipment.created`
- `shipment.courier_assigned`
- `shipment.awaiting_pickup`
- `shipment.picked_up`
- `shipment.in_transit`
- `shipment.out_for_delivery`
- `shipment.delivered`
- `shipment.delivery_failed`
- `shipment.returned_to_sender`
- `shipment.cancelled`
- `shipment.unknown_pending_review`
- `shipment.reconciled`

Payloads include shipment ID, order ID, canonical status, provider, tracking number when available, event time, and provider-event reference.

## 13. Return and Refund Events

Return:

- `return.requested`
- `return.approved`
- `return.rejected`
- `return.received`
- `return.inspected`
- `return.completed`
- `return.cancelled`

Refund:

- `refund.requested`
- `refund.approved`
- `refund.rejected`
- `refund.processing`
- `refund.partially_completed`
- `refund.completed`
- `refund.failed`

## 14. Notification and Reporting Events

- `notification.requested`
- `notification.sent`
- `notification.failed`
- `report.requested`
- `report.completed`
- `report.failed`

## 15. Consumer Registry

| Consumer | Event groups |
|---|---|
| Notification worker | tenant, order, payment, inventory, WMS, shipment |
| SSE projector | tenant, order, inventory, WMS, shipment, notification |
| Reporting projector | order, payment, inventory, shipment |
| Delivery synchronizer | shipment and provider events |
| Audit/reconciliation tools | selected critical state events |

Every consumer must document handled event versions and idempotency storage.

## 16. Compatibility Rules

Backward-compatible additions:

- optional field addition
- new event type
- new enum value only when consumers tolerate unknown values

Breaking changes:

- required field removal or semantic change
- type change
- identifier meaning change
- reordered business meaning

## 17. Acceptance Criteria

1. Every durable state transition has an event where downstream work depends on it.
2. Event versions align with aggregate versions.
3. Tenant context is mandatory for tenant-owned events.
4. Delivery and ordering semantics are explicit.
5. Sensitive data is excluded.
6. Consumers can process duplicates safely.

## Change Summary

Initial domain-event contract draft synchronized with the final state machine and outbox design.
