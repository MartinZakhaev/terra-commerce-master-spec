# Biteship Webhook Contract — Terra Commerce

**Document ID:** API-TC-002  
**Version:** 1.0.0  
**Status:** Final  
**Owner:** Integration Lead  
**Last Updated:** 2026-06-30  
**Upstream:** FSD v1.0.1, System Architecture v1.0.0 Final, State Machine v1.0.0 Final, Database Design v1.0.0 Final  
**Audit:** `reviews/Contract_Suite_Audit_v1.0.0.md`

## 1. Endpoint

`POST /api/v1/webhooks/biteship`

The endpoint accepts only Biteship-originated webhook traffic and is protected by the provider's officially supported verification mechanism, request-size limits, rate limits, deduplication, and payload validation.

Exact signature headers, token names, event names, and payload fields must be confirmed against current official Biteship documentation before implementation. This contract defines the normalized Terra Commerce behavior.

## 2. Normalized Processing Sequence

1. Read raw body and provider headers.
2. Verify authenticity before trusting provider identifiers.
3. Reject malformed or oversized input.
4. Normalize provider event ID, event type, provider order/shipment ID, status, event time, tracking, courier, service, location, and description where present.
5. Persist durable webhook receipt.
6. Deduplicate by provider and provider event ID.
7. Resolve shipment and tenant from trusted stored provider references.
8. Map provider status to the canonical shipment state.
9. Apply monotonic transition rules and aggregate-version concurrency.
10. Atomically persist tracking event, shipment state, order delivery projection, state transition, and outbox event.
11. Acknowledge only after processing succeeds or a durable internal retry has been safely scheduled.
12. Deliver SSE and notifications asynchronously after commit.

## 3. Receipt and Processing States

Webhook records use normalized states:

- `received`
- `processing`
- `processed`
- `retry_scheduled`
- `failed`
- `quarantined`

`received` means the request was durably stored. `processed` means all business writes committed. `retry_scheduled` means durable internal retry ownership is guaranteed.

## 4. Canonical Shipment Statuses

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

Unmapped or conflicting states enter `unknown_pending_review` and create an operational alert.

## 5. Idempotency and Conflict Rules

- Unique identity: `(provider, provider_event_id)`.
- Same event ID and same payload hash: return duplicate success without side effects.
- Same event ID and different payload hash: quarantine as provider conflict.
- Tracking event uniqueness uses provider event ID where available.
- Shipment transition uses expected aggregate version.
- Duplicate current-state updates are idempotent.

## 6. Ordering Rules

- Backward transitions are rejected unless the final state machine explicitly allows them.
- Protected terminal states cannot be replaced automatically.
- Provider timestamp alone cannot override a later locally accepted state.
- Conflicting terminal events enter `unknown_pending_review`.
- Ignored, quarantined, and reconciled events remain observable.

## 7. Atomic Transaction Boundary

One PostgreSQL transaction includes:

- webhook processing-state update
- tracking-event insert
- shipment state/version update
- order delivery projection update where applicable
- state-transition insert
- outbox-event insert

No SSE or email delivery occurs inside this transaction.

## 8. Acknowledgement Rules

### Processed synchronously

Return provider-compatible success after the transaction commits.

### Duplicate already processed

Return success with no repeated side effects.

### Durable retry scheduled

Success may be returned only when webhook receipt and retry ownership are durably persisted and the provider contract permits acknowledgement before business completion.

### No durable processing guarantee

Return a provider-compatible retryable failure status, typically `500` or `503`, so the provider can retry.

The response body may expose only generic receipt metadata; provider-specific response expectations take precedence once officially verified.

## 9. Security and Tenant Resolution

- HTTPS only.
- Verification occurs before business trust.
- Secret headers are never logged.
- Tenant is resolved from stored shipment/provider references, never from body-provided tenant data.
- Raw payload storage is minimized and sanitized.
- Operator access to webhook records is restricted and audited.

## 10. Retry and Dead-Letter Policy

- Provider retries are accepted and deduplicated.
- Internal retries use bounded backoff.
- Permanent validation failures are quarantined.
- Exhausted processing enters dead-letter handling.
- Manual replay is explicit, authorized, and audited.

## 11. Observability

Metrics include receipt count, verification failure, duplicate count, mapping failure, processing latency, conflicts, retries, quarantines, and dead letters.

Logs include correlation ID, provider event ID, provider order ID, resolved tenant ID, and outcome without secrets or unnecessary personal data.

## 12. Versioned Provider Mapping Registry

The implementation maintains a registry containing:

- provider event/status value
- canonical status
- allowed source canonical states
- terminal flag
- mapping version
- effective date
- official documentation reference

Mapping changes require review and regression tests.

## 13. Required Tests

- valid new event
- valid duplicate
- duplicate ID with changed payload
- invalid authenticity verification
- missing event identity
- unknown shipment
- unknown status
- out-of-order event
- conflicting terminal event
- transaction failure
- durable retry scheduling
- retry without durable scheduling
- cross-tenant spoof attempt

## 14. Acceptance Criteria

- Authenticity is verified before business trust.
- Duplicate events do not duplicate side effects.
- Tenant resolution uses trusted stored references.
- Shipment transitions match the final state machine.
- Business writes and outbox publication are atomic.
- `received`, `processed`, and `retry_scheduled` are distinct.
- Success is never returned when neither processing nor durable retry is guaranteed.

## Change Summary

Final version closes CONTRACT-M05 by separating receipt from processing, requiring durable retry ownership before deferred acknowledgement, and deferring exact Biteship wire details to verified official documentation.
