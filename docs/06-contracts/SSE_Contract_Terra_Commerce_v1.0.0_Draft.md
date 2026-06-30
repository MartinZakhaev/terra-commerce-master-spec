# Server-Sent Events Contract — Terra Commerce

**Document ID:** EVT-TC-002  
**Version:** 1.0.0  
**Status:** Draft  
**Owner:** Gateway Lead  
**Last Updated:** 2026-06-30  
**Upstream:** FSD v1.0.1, System Architecture v1.0.0 Final, State Machine v1.0.0 Final, Domain Event Contract v1.0.0 Draft

## 1. Endpoint

`GET /api/v1/events/stream`

Required:

- authenticated user or customer
- resolved tenant context
- `Accept: text/event-stream`

Optional:

- `Last-Event-ID`
- query `topics` only for authorized supported topic filters

## 2. Response Headers

- `Content-Type: text/event-stream`
- `Cache-Control: no-cache, no-transform`
- `Connection: keep-alive`
- `X-Accel-Buffering: no`
- `X-Correlation-ID`

## 3. SSE Frame

```text
id: 01J...
event: order.updated
data: {"schema_version":1,"tenant_id":"...","resource_type":"order","resource_id":"...","occurred_at":"...","data":{}}

```

## 4. Public Envelope

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

The internal domain-event envelope is never forwarded directly.

## 5. Audience Classes

### Global superadmin

May receive platform health, tenant lifecycle, plan usage, failed jobs, and integration alerts according to permission.

### Tenant staff

May receive tenant-scoped order, inventory, WMS, shipment, report, and notification events according to role permission.

### Customer

May receive only events related to the authenticated customer's own orders, payments, shipments, returns, refunds, and notifications.

## 6. Public Event Types

Platform:

- `platform.health.updated`
- `tenant.updated`
- `tenant.suspended`
- `plan.usage.updated`
- `operation.failed`

Tenant operations:

- `order.created`
- `order.updated`
- `payment.updated`
- `inventory.updated`
- `inventory.alert`
- `picking.updated`
- `packing.updated`
- `shipment.updated`
- `return.updated`
- `refund.updated`
- `report.updated`
- `notification.created`

Customer:

- `customer.order.updated`
- `customer.payment.updated`
- `customer.shipment.updated`
- `customer.return.updated`
- `customer.refund.updated`
- `customer.notification.created`

## 7. Filtering Rules

- Tenant ID must match resolved tenant.
- Customer resource ownership must match authenticated customer.
- Staff event class requires corresponding read permission.
- Internal notes, raw provider payloads, audit before/after values, secrets, and unrestricted personal data are prohibited.
- Payloads should carry enough data to update badges or indicate refresh, not replace authoritative API resources.

## 8. Connection Lifecycle

1. Gateway authenticates and resolves tenant/audience.
2. Gateway validates tenant status and connection plan limit.
3. Connection opens and sends `system.connected`.
4. Heartbeat comment is sent at a configured interval.
5. Events are projected and filtered.
6. Connection closes on logout, token expiry, tenant suspension, permission revocation, server shutdown, or network loss.

## 9. System Events

- `system.connected`
- `system.refresh_required`
- `system.permission_changed`
- `system.tenant_suspended`
- `system.shutdown`

## 10. Replay and Reconnection

- Client sends `Last-Event-ID` after disconnection.
- Server replays only events retained for the authenticated audience.
- Replay is bounded by retention and maximum count.
- If event ID is unavailable, too old, or audience rules changed, send `system.refresh_required` and require API refetch.
- Event IDs are monotonic within the replay store selected by technical design, but clients must not infer business ordering across aggregates.

## 11. Delivery Semantics

- At-least-once delivery is possible.
- Duplicate SSE events are possible.
- Clients deduplicate by SSE `id`.
- Business state must be fetched from APIs when precision is required.
- A dropped SSE event must not cause data loss because APIs remain authoritative.

## 12. Capacity and Limits

- Initial target: 50 concurrent connections platform-wide.
- Per-tenant and per-user limits follow plan and security policy.
- One user may have multiple browser tabs, subject to limits.
- Slow consumers may be disconnected and instructed to refresh.
- Gateway must use bounded buffers.

## 13. Error and Close Behavior

Before stream establishment, normal HTTP errors apply:

- `401` unauthenticated
- `403` unauthorized or tenant suspended
- `409` connection limit conflict where appropriate
- `429` rate or connection limit
- `503` maintenance or unavailable stream infrastructure

After establishment, terminal system events may be sent before close when safe.

## 14. Security

- HTTPS only.
- No credentials in query strings.
- CORS restricted to approved origins.
- Session/token revalidation occurs periodically or on security events.
- Event payloads are audience specific.
- Proxy buffering is disabled only for the SSE route.

## 15. Client Responsibilities

- Reconnect with backoff and jitter.
- Send `Last-Event-ID` when available.
- Deduplicate by event ID.
- Refetch authoritative resource on `system.refresh_required`.
- Stop reconnecting after explicit suspension/forbidden response until user action.

## 16. Acceptance Criteria

1. No cross-tenant or cross-customer event leakage.
2. Internal domain payloads are projected and filtered.
3. Reconnect and refresh fallback are defined.
4. Duplicate events are safe.
5. Connection limits protect the MVP resource target.
6. Tenant suspension terminates active streams.

## Change Summary

Initial SSE contract draft synchronized with the gateway architecture and domain-event contract.
