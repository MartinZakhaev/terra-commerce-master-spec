# Technical Design Document — Worker and Event Pipeline

**Document ID:** TDD-TC-WRK-001  
**Version:** 1.0.0  
**Status:** Final  
**Owner:** Backend and Platform Lead  
**Last Updated:** 2026-06-30  
**Upstream:** Final Architecture, Domain Event Contract, SSE Contract, Database Design, Threat Model  
**Audit:** `reviews/Technical_Design_Suite_Audit_v1.0.0.md`

## 1. Runtime Model

Logical components:

- transactional outbox publisher
- notification consumer
- SSE projector
- reporting consumer
- delivery/reconciliation consumer
- scheduler and cleanup consumer
- dead-letter and replay tooling

MVP may run these in one worker process with separate bounded concurrency pools and metrics.

## 2. Outbox Publisher

Publisher claims pending rows using `FOR UPDATE SKIP LOCKED`, records claim owner/time, commits the claim, publishes the stable `event_id` to Redis Streams, and then marks the row published.

Rules:

- Event ID and aggregate version never change across retries.
- Stale claims are reclaimed after lease expiry.
- A crash after Redis publish but before PostgreSQL acknowledgement may duplicate delivery.
- Consumers therefore use durable receipts keyed by consumer and event ID.
- `published` means accepted by transport, not completed by every consumer.

## 3. Publish/Acknowledge Reconciliation

A periodic reconciler scans:

- rows stuck in `publishing` beyond claim lease
- pending rows older than operational threshold
- published rows without expected transport metadata where tracked
- repeated publication failures

Stale publishing rows return to pending or failed according to attempt policy. Reconciliation never creates a new logical event ID. Operational tooling exposes outbox state separately from consumer completion.

## 4. Redis Streams and Consumer Groups

Recommended global stream names are stable and tenant IDs remain inside envelopes. Consumer groups are separated by notification, SSE, reporting, delivery, and reconciliation concerns.

All consumers validate schema, event type/version, tenant scope, aggregate identity/version, payload size, and sensitivity class before processing.

## 5. Consumer Idempotency and Ordering

Durable consumer receipts store consumer, event ID, processing lease, status, timestamps, and optional aggregate version.

- Completed event: return prior success.
- Expired in-progress lease: reclaim safely.
- Strict-order consumer stores last processed aggregate version.
- Lower/equal version is validated and ignored.
- Next version is processed.
- Version gap is deferred and retried.
- Persistent gap is quarantined and reconciled.

Receipt retention exceeds the maximum event replay and retry period.

## 6. Retry and Dead Letters

Failures are classified as transient, dependency-not-ready/version-gap, permanent validation, or security/tenant mismatch.

Retries use exponential backoff with jitter and maximum delay. Permanent and security failures are quarantined immediately or after minimal validation retry. Exhausted transient failures become dead letters.

Manual replay requires permission, reason, current-state validation, idempotency, and audit. Replay never bypasses state machines.

## 7. Notification and Email

Notification worker resolves recipients from authoritative roles/preferences, creates persistent in-app notifications, requests email delivery, and emits notification outcomes.

Deduplication combines source event, channel, template, and recipient. Email adapters are provider-neutral, time-bounded, retry-aware, and redact secrets and PII.

## 8. SSE Projection

The SSE projector validates domain events, resolves candidate audiences, builds minimal public projections, excludes sensitive fields, and writes audience-scoped replay/fan-out records.

Customer ownership is verified from authoritative data when the event does not provide a sufficient trusted ownership relation.

## 9. Reporting Worker

Generation flow:

1. Validate original request and store requester/tenant/permission context for audit.
2. Mark report running.
3. Run tenant-scoped queries with row and statement limits.
4. Stream output to bounded temporary storage/object storage.
5. Mark complete and emit event.

Download authorization is never based solely on the generation-time snapshot. Every download revalidates current tenant membership, current permission, report ownership/scope, and object expiry. Revocation prevents future download.

Large reports must stream and must not load complete datasets into memory.

## 10. Delivery and Reconciliation

Delivery consumer retries shipment creation using stable idempotency, synchronizes uncertain provider outcomes, processes durable webhook retries, and reconciles unknown states.

Provider timeout is treated as unknown outcome. Tenant is resolved from stored provider/shipment references. Manual reconciliation is permission controlled and audited.

## 11. Scheduler

Scheduled jobs may include reservation expiration, stale claim recovery, webhook retry, report/file cleanup, notification retry, usage rollover, retention jobs, and health/reconciliation scans.

A distributed lock or equivalent ownership lease prevents duplicate scheduler execution. All jobs are idempotent.

## 12. Concurrency and Backpressure

Separate bounded limits are configured per workload. Reports and provider work remain low concurrency; notifications and projection may be moderate. Global concurrency is capped for 2 vCPU / 4 GB RAM.

When downstream capacity is exhausted, workers stop claiming new work rather than building unbounded in-memory queues.

## 13. Redis Failure and Recovery

When Redis publication fails, outbox rows remain recoverable in PostgreSQL. Consumers stop safely on Redis loss. Pending stream entries and outbox rows resume after recovery. Redis is not authoritative business state.

Pending entries may be reclaimed only after lease/heartbeat expiry, preserving delivery count and avoiding active-job theft.

## 14. Security and Observability

Services use private Redis/PostgreSQL access and least-privilege identities. Tenant context is validated before side effects. Events are size/schema checked. Reports enforce export permissions. Manual replay is audited.

Metrics cover outbox age/count, publication failures, stream lag, pending entries, handler latency, retries, dead letters, version gaps, report size/duration, notifications, provider outcomes, and reconciliation.

## 15. Shutdown

Stop claiming work, complete or safely release leases, acknowledge only completed handlers, flush telemetry, and close clients. Restart must not lose or duplicate business side effects beyond the documented at-least-once model.

## 16. Testing

- Crash after publish/before acknowledgement.
- Stale claim reconciliation.
- Duplicate consumer delivery.
- Version-gap handling.
- Pending-entry reclaim.
- Redis outage/recovery.
- Dead-letter replay authorization.
- SSE leakage tests.
- Report permission revocation before download.
- Provider unknown-outcome tests.
- Bounded-concurrency load tests.

## 17. Acceptance Criteria

1. Critical committed changes remain publishable after failures.
2. Outbox publication and consumer completion are not conflated.
3. Durable receipts prevent duplicate side effects.
4. Strict-order consumers detect gaps.
5. Report downloads revalidate current authorization.
6. Redis outage remains recoverable from PostgreSQL.
7. Concurrency remains bounded.

## Change Summary

Final version closes TDD-M03 and TDD-M04 by defining outbox reconciliation/durable consumer receipts and current-permission revalidation for report downloads.
