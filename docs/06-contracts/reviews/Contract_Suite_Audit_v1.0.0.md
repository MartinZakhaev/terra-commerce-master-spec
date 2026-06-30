# Contract Suite Audit — Terra Commerce

**Document ID:** REVIEW-TC-010  
**Version:** 1.0.0  
**Status:** Completed  
**Audit Date:** 2026-06-30  
**Reviewed:** OpenAPI, Domain Event, SSE, and Biteship Webhook contract drafts v1.0.0  
**Upstream:** FSD v1.0.1, System Architecture v1.0.0 Final, State Machine v1.0.0 Final, Database Design and Data Dictionary v1.0.0 Final

## 1. Executive Summary

The four drafts are materially aligned with the approved functional, architecture, state, and data documents. The suite correctly centralizes public APIs in the Go gateway, separates internal domain events from public SSE projections, and defines idempotent Biteship webhook processing.

No critical contradiction was found. Five medium-priority issues must be closed before finalization.

**Draft verdict:** Pass with required corrections.

## 2. Findings

### CONTRACT-M01 — OpenAPI Operations Need Stable IDs and Explicit Command Metadata

The endpoint inventory is complete at domain level, but final contract must require stable `operationId`, permission key, audience, idempotency, expected version, allowed source states, emitted events, and audit behavior for every command.

### CONTRACT-M02 — Cancellation and Transition Resource Naming Needs Normalization

The draft mixes generic transition endpoints and action-specific endpoints. Final contract must establish:

- generic transition endpoints only for global administrative lifecycle changes where justified
- action-specific command resources for order, payment, WMS, shipment, return, and refund transitions
- no verbs hidden in ambiguous PATCH operations

### CONTRACT-M03 — Domain Event Consumer Ordering and Schema Registry Need Stronger Rules

Final event contract must require:

- consumer-side aggregate-version tracking
- quarantine on version gaps where a consumer needs strict sequence
- a registry of event type, schema version, producer, consumers, and sensitivity classification
- no reuse of an event name for incompatible semantics

### CONTRACT-M04 — SSE Replay Scope and Event IDs Need Stronger Isolation

Final SSE contract must define that replay IDs are scoped to the authenticated audience and cannot be used to retrieve another tenant/customer stream. It must also define `retry:` guidance, bounded replay count/age, and permission revalidation before replay.

### CONTRACT-M05 — Biteship Acknowledgement Must Avoid False Success

The webhook draft permits success after receipt persistence even when processing is deferred. Final contract must distinguish:

- `received` from `processed`
- acknowledgement only after durable receipt and successful enqueue/claim for internal processing
- provider retry response when neither processing nor durable retry scheduling is guaranteed
- no claim about exact signature headers or status fields until verified from official provider documentation

## 3. Synchronization Matrix

| Area | Result |
|---|---|
| Gateway-only public APIs | Pass |
| Tenant resolution and authorization | Pass |
| State-machine command alignment | Pass with M01/M02 |
| Database idempotency and versioning | Pass |
| Durable outbox events | Pass |
| Event delivery semantics | Pass with M03 |
| SSE audience filtering | Pass with M04 |
| Biteship deduplication and atomicity | Pass with M05 |
| Error normalization | Pass |
| Sensitive-data exclusion | Pass |
| Observability and audit | Pass |

## 4. Finalization Gate

The suite may become Final after CONTRACT-M01 through CONTRACT-M05 are incorporated consistently across all four documents.

The machine-readable `openapi.yaml` remains a required implementation artifact, but the normative endpoint and behavior contract can become Final before that file is generated.

## 5. Audit Result

- Critical findings: **0**
- High findings: **0**
- Medium findings: **5**
- Material contradictions: **0**
- Missing contract domains: **0**

**Required action:** publish corrected Final versions of all four contracts and run a final suite verification.
