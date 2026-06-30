# Server-Sent Events Contract — Terra Commerce

**Document ID:** EVT-TC-002  
**Version:** 1.0.0  
**Status:** Final  
**Owner:** Gateway Lead  
**Last Updated:** 2026-06-30  
**Upstream:** FSD v1.0.1, System Architecture v1.0.0 Final, State Machine v1.0.0 Final, Domain Event Contract v1.0.0 Final  
**Audit:** `reviews/Contract_Suite_Audit_v1.0.0.md`

## 1. Endpoint and Headers

`GET /api/v1/events/stream`

Required response headers:

- `Content-Type: text/event-stream`
- `Cache-Control: no-cache, no-transform`
- `Connection: keep-alive`
- `X-Accel-Buffering: no`
- `X-Correlation-ID`

Client may send `Last-Event-ID`. Credentials must never be sent in query strings.

## 2. Frame Format

```text
id: audience-scoped-event-id
event: order.updated
retry: 3000
data: {"schema_version":1,"tenant_id":"...","resource_type":"order","resource_id":"...","occurred_at":"...","data":{}}

```

The `retry` value is advisory and may vary by environment.

## 3. Public Envelope

```json
{
  "schema_version": 1,
  "tenant_id": "uuid",
  "resource_type": "order",
  "resource_id": "uuid",
  "occurred_at": "2026-06-30T00:00:00Z",
  "correlation_id": "uuid-or-null",
  "data": {}
}
```

Raw internal domain events are never forwarded directly.

## 4. Audience Isolation

### Global superadmin

May receive authorized platform health, tenant lifecycle, plan usage, failed-job, and integration alerts.

### Tenant staff

May receive tenant-scoped order, inventory, WMS, shipment, report, and notification events according to current permissions.

### Customer

May receive only events for the authenticated customer's own orders, payments, shipments, returns, refunds, and notifications.

Replay IDs are scoped to the authenticated audience identity and authorization snapshot. A replay ID from another tenant, customer, user, or audience class must not return events.

## 5. Event Catalog

Platform: `platform.health.updated`, `tenant.updated`, `tenant.suspended`, `plan.usage.updated`, `operation.failed`.

Tenant staff: `order.created`, `order.updated`, `payment.updated`, `inventory.updated`, `inventory.alert`, `picking.updated`, `packing.updated`, `shipment.updated`, `return.updated`, `refund.updated`, `report.updated`, `notification.created`.

Customer: `customer.order.updated`, `customer.payment.updated`, `customer.shipment.updated`, `customer.return.updated`, `customer.refund.updated`, `customer.notification.created`.

System: `system.connected`, `system.refresh_required`, `system.permission_changed`, `system.tenant_suspended`, `system.shutdown`.

## 6. Projection and Filtering

- Tenant must match resolved tenant.
- Customer ownership must match authenticated customer.
- Staff event family requires current read permission.
- Internal notes, raw provider payloads, audit before/after values, secrets, and unnecessary personal data are prohibited.
- Events primarily signal changes; authoritative resources are fetched through APIs.

## 7. Connection Lifecycle

1. Authenticate and resolve audience.
2. Validate tenant status, permissions, and connection limits.
3. Open stream and send `system.connected`.
4. Send heartbeat comments at a configured interval.
5. Project and filter events.
6. Revalidate session and permissions periodically and before replay.
7. Close on logout, expiry, revocation, suspension, shutdown, or network loss.

## 8. Replay and Reconnection

- Replay is bounded by configured maximum age and maximum event count.
- Permission and ownership are revalidated before replay.
- Replay queries are audience scoped.
- Unknown, expired, unauthorized, or unavailable IDs cause `system.refresh_required`.
- Event IDs support replay lookup but do not imply global business ordering.
- Clients reconnect with exponential backoff and jitter.

## 9. Delivery Semantics

- At-least-once delivery is possible.
- Clients deduplicate by SSE event ID.
- Slow consumers may be disconnected.
- Gateway buffers are bounded.
- Missing SSE delivery cannot lose business truth because APIs and PostgreSQL remain authoritative.

## 10. Capacity and Error Behavior

Initial target: 50 concurrent platform-wide connections, with per-tenant and per-user limits.

Before stream establishment:

- `401` unauthenticated
- `403` unauthorized or suspended
- `409` connection conflict where applicable
- `429` limit exceeded
- `503` maintenance or unavailable stream infrastructure

After establishment, a terminal system event may be sent before close when safe.

## 11. Security

- HTTPS only.
- Approved-origin CORS.
- No credentials in URL.
- Periodic authorization revalidation.
- Audience-specific payloads.
- Proxy buffering disabled only for SSE route.

## 12. Acceptance Criteria

- No cross-tenant or cross-customer leakage.
- Replay IDs cannot cross audience boundaries.
- Replay limits and permission revalidation are explicit.
- `retry:` guidance and refresh fallback are defined.
- Duplicate events are safe.
- Suspension terminates relevant streams.

## Change Summary

Final version closes CONTRACT-M04 with audience-scoped replay IDs, permission revalidation, bounded replay, and explicit retry guidance.
