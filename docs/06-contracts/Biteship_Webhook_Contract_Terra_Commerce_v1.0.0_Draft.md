# Biteship Webhook Contract — Terra Commerce

**Document ID:** API-TC-002  
**Version:** 1.0.0  
**Status:** Draft  
**Owner:** Integration Lead  
**Last Updated:** 2026-06-30  
**Upstream:** FSD v1.0.1, System Architecture v1.0.0 Final, State Machine v1.0.0 Final, Database Design v1.0.0 Final

## 1. Endpoint

`POST /api/v1/webhooks/biteship`

The endpoint is public only to Biteship and protected by provider verification, rate limits, request-size limits, deduplication, and payload validation.

## 2. Processing Sequence

1. Accept raw request body and provider headers.
2. Verify provider authenticity using the supported signature/token mechanism.
3. Reject oversized or malformed payloads.
4. Extract provider event ID, provider shipment/order ID, event type, and event time.
5. Persist `provider_webhook_events` receipt before business processing.
6. Deduplicate by `(provider, provider_event_id)`.
7. Resolve shipment and tenant from trusted provider identifiers.
8. Map provider status to canonical shipment status.
9. Validate monotonic state transition.
10. Persist tracking event and shipment transition atomically.
11. Insert durable outbox event in the same transaction.
12. Return acknowledgement.
13. Process SSE and notifications asynchronously.

## 3. Required Provider Fields

The adapter must normalize these logical fields, regardless of exact provider field names:

- `provider_event_id`
- `event_type`
- `provider_order_id`
- `tracking_number` when available
- `provider_status`
- `event_time` when available
- courier/service references when available
- location and description when available

The exact Biteship payload mapping must be verified against the provider's current official documentation before implementation.

## 4. Canonical Status Mapping

Target statuses:

- `shipment_pending`
- `courier_assigned`
- `awaiting_pickup`
- `picked_up`
- `in_transit`
- `out_for_delivery`
- `delivered`
- `delivery_failed`
- `returned_to_sender`
- `cancelled`
- `unknown_pending_review`

Unknown or unmapped provider status enters `unknown_pending_review` and creates an operational alert.

## 5. Idempotency

- Unique provider event identity: `(provider, provider_event_id)`.
- Duplicate event with identical payload hash returns success without repeating side effects.
- Duplicate event ID with a different payload hash is quarantined as a provider conflict.
- Shipment transition uses aggregate version concurrency.
- Tracking events use provider event ID uniqueness where available.

## 6. Ordering and Reconciliation

- Duplicate current-state updates are idempotent.
- Backward transitions are rejected unless explicitly allowed by the final shipment state machine.
- Terminal state conflicts are quarantined.
- Provider event timestamp alone does not override a later locally accepted state.
- Unresolved conflicts enter `unknown_pending_review` and require authorized reconciliation.

## 7. Transaction Boundary

The following occur in one PostgreSQL transaction:

- update webhook processing record
- insert tracking event
- update shipment canonical state/version
- update order delivery projection where applicable
- insert state-transition record
- insert outbox event

SSE and email delivery occur after commit.

## 8. Response Behavior

### Successful new event

`200 OK`

```json
{
  "received": true,
  "duplicate": false,
  "correlation_id": "uuid"
}
```

### Successful duplicate

`200 OK`

```json
{
  "received": true,
  "duplicate": true,
  "correlation_id": "uuid"
}
```

### Authentication failure

`401` or `403` according to provider verification mechanism.

### Malformed request

`400 Bad Request`.

### Temporary internal failure

Return a provider-compatible retryable status, normally `500` or `503`, only when safe persistence has not completed.

If receipt has been durably persisted, asynchronous processing may return success and retry internally according to provider expectations.

## 9. Retry Policy

- Provider retry is accepted and deduplicated.
- Internal processing retries use bounded backoff.
- Permanent validation failures are quarantined, not retried indefinitely.
- Dead-letter operations are visible to authorized superadmins.

## 10. Security and Privacy

- HTTPS only.
- Verify signature/token before trusting payload identity.
- Never log secret headers.
- Store only sanitized payload references where full raw payload is unnecessary.
- Restrict webhook event access to authorized operators.
- Resolve tenant from stored shipment/provider references, never from untrusted body tenant fields.

## 11. Observability

Metrics:

- webhook received count
- verification failures
- duplicates
- mapping failures
- processing latency
- transition conflicts
- internal retries
- quarantined events

Logs include correlation ID, provider event ID, provider order ID, resolved tenant ID, and outcome without secrets.

## 12. Provider Mapping Registry

The implementation must maintain a versioned mapping table or code registry containing:

- provider event/status value
- canonical status
- allowed source canonical states
- terminal flag
- mapping version
- effective date

Mapping changes require contract review and regression tests.

## 13. Test Cases

- valid new event
- valid duplicate
- duplicate ID with changed payload
- invalid signature
- missing provider event ID
- unknown shipment
- unknown status
- out-of-order update
- conflicting terminal event
- transaction failure before commit
- retry after persisted receipt
- cross-tenant spoof attempt

## 14. Acceptance Criteria

1. Webhooks are verified before business trust.
2. Duplicate events never duplicate side effects.
3. Tenant is resolved from trusted stored references.
4. State transitions follow canonical shipment rules.
5. Tracking, shipment, transition, and outbox writes are atomic.
6. Unknown and conflicting events are quarantined.
7. Provider retries and internal retries are observable.

## Change Summary

Initial Biteship webhook contract draft synchronized with the final shipment state machine and database reliability model.
