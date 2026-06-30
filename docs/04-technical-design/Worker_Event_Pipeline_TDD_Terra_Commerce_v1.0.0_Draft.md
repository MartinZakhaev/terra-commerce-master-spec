# Technical Design Document — Worker and Event Pipeline

**Document ID:** TDD-TC-WRK-001  
**Version:** 1.0.0  
**Status:** Draft  
**Owner:** Backend and Platform Lead  
**Last Updated:** 2026-06-30  
**Upstream:** System Architecture v1.0.0 Final, Domain Event Contract v1.0.0 Final, SSE Contract v1.0.0 Final, Database Design v1.0.0 Final, Security Threat Model v1.0.0 Final

## 1. Purpose

This document defines the worker topology, transactional outbox publisher, Redis Streams, consumer groups, retries, dead letters, event ordering, notification/report processing, SSE projection, reconciliation, scheduling, observability, and recovery.

## 2. Runtime Components

```text
PostgreSQL Outbox
      |
Outbox Publisher
      |
Redis Streams
  |       |        |        |
Notify   SSE     Reports   Delivery
Worker Projector Worker    Worker
      \      |       /        /
       Dead-Letter and Reconciliation
```

MVP may run these logical consumers in one worker process with bounded concurrency. Each handler remains separately identifiable and extractable.

## 3. Process Structure

```text
cmd/worker
src/
  bootstrap/
  outbox/
  streams/
  consumers/
    notifications/
    sse-projector/
    reporting/
    delivery/
    cleanup/
    reconciliation/
  scheduler/
  retry/
  deadletter/
  idempotency/
  observability/
  security/
  testkit/
```

## 4. Transactional Outbox Publisher

Publisher loop:

1. Select pending rows where `available_at <= now()`.
2. Claim bounded batch using `FOR UPDATE SKIP LOCKED`.
3. Set claim owner/time and publishing state.
4. Commit claim transaction.
5. Publish event to configured Redis Stream.
6. Mark row published with timestamp.
7. On failure, increment attempts and reschedule with backoff.
8. Move to failed/dead-letter handling after threshold.

Rules:

- `event_id` remains stable across retries.
- Unique aggregate/version/event constraint prevents duplicate logical outbox entries.
- A publisher crash after Redis publish but before database acknowledgement may redeliver; consumers must be idempotent.
- Stale claims are recoverable after lease expiry.

## 5. Redis Stream Topology

Recommended streams:

- `tc:events:domain`
- `tc:events:notifications`
- `tc:events:sse`
- `tc:events:reports`
- `tc:events:delivery`
- `tc:events:deadletter`

A simpler single domain stream may be used initially, with consumer groups per concern. Stream names and consumer groups are global infrastructure names; tenant ID remains in each event.

Consumer groups:

- `notification-workers`
- `sse-projectors`
- `reporting-workers`
- `delivery-workers`
- `reconciliation-workers`

## 6. Event Validation

Before handling, each consumer validates:

- envelope schema
- supported event type/version
- required tenant scope
- aggregate identity and version
- payload size
- sensitivity classification expectations

Malformed or unsupported events are quarantined rather than repeatedly retried without bound.

## 7. Consumer Idempotency

Each consumer stores processing identity by:

- consumer name
- event ID
- result status
- processed timestamp
- optional aggregate version

Rules:

- Completed event IDs return previous success without repeating side effects.
- In-progress leases can be reclaimed after timeout.
- Provider/API calls use their own idempotency keys.
- Idempotency retention must exceed maximum event replay/retry period.

## 8. Aggregate Ordering

Strict-order consumers maintain last processed aggregate version.

Handling:

- equal/lower version already processed: validate and ignore
- next expected version: process
- higher version gap: defer and retry for bounded period
- persistent gap: quarantine and trigger reconciliation

Consumers not dependent on strict aggregate order may process independently but must still be idempotent.

## 9. Retry Policy

Retry classification:

- transient: timeout, connection failure, rate limit, temporary provider error
- conflict/retryable: dependency not yet ready, version gap
- permanent: invalid schema, unknown required identifier, prohibited transition
- security: signature or tenant mismatch; quarantine immediately

Backoff uses exponential delay with jitter and configured maximum. Retry count and next attempt are observable.

## 10. Dead-Letter Handling

Dead-letter record includes:

- original event and stream ID
- consumer
- tenant and aggregate
- failure category
- sanitized error
- attempts
- first/last failure time
- replay status and actor

Manual replay requires permission, reason, current-state validation, idempotency, and audit. Replaying must not bypass state machines.

## 11. Notification Worker

Responsibilities:

- consume business events
- resolve recipient roles and preferences
- create persistent in-app notification
- request email delivery
- emit `notification.sent` or `notification.failed`

Notification failure never rolls back the originating business transaction.

Deduplication key combines event ID, channel, template, and recipient.

## 12. Email Delivery

Email adapter:

- provider-neutral interface
- tenant branding/template context
- explicit timeout
- provider idempotency where supported
- sanitized failure logging
- retry classification

Secrets are supplied through approved configuration.

## 13. SSE Projector

The projector converts internal domain events into audience-specific public events.

Steps:

1. Validate internal event.
2. Resolve public event types and candidate audiences.
3. Build minimal payload.
4. Exclude sensitive fields.
5. Persist bounded replay record or publish to gateway fan-out channel.
6. Attach audience-scope identity.

Customer projectors must resolve customer ownership from authoritative data, not trust event payload alone when ownership is ambiguous.

## 14. Reporting Worker

Report lifecycle:

1. Validate request and permission snapshot.
2. Mark report running.
3. Execute tenant-scoped query with statement timeout and row limits.
4. Generate file using streaming output.
5. Store in tenant-scoped object key.
6. Mark complete and emit event.
7. Produce expiring authorized download.

Failures mark report failed and preserve sanitized details. Large reports must not load entire datasets into memory.

## 15. Delivery Worker

Responsibilities:

- retry shipment creation with idempotency
- synchronize uncertain provider outcomes
- process deferred webhook records
- reconcile unknown shipment states
- emit normalized shipment events

Rules:

- Tenant resolved from stored shipment/provider reference.
- Provider-specific details remain in adapter.
- Unknown outcome is not treated as safe failure.
- Reconciliation is permission-controlled when manually initiated.

## 16. Scheduled Jobs

Schedules may include:

- reservation expiration after policy is approved
- stale outbox claim recovery
- pending webhook retry
- report/file expiry cleanup
- notification retry
- usage counter rollover
- retention/archival jobs
- health and reconciliation scans

A distributed lock prevents duplicate scheduler ownership. Job commands remain idempotent.

## 17. Worker Concurrency

Concurrency is configured per handler class:

- outbox publisher: small bounded batch
- notifications: moderate
- reports: very low due to memory/CPU cost
- delivery: low and provider-rate-aware
- SSE projection: moderate and non-blocking

Global worker concurrency must respect 2 vCPU / 4 GB RAM. Backpressure is preferred over unbounded queues.

## 18. Redis Failure Behavior

- Outbox rows remain pending when Redis publication fails.
- No published status is recorded until publish succeeds.
- Consumers stop safely when Redis is unavailable.
- PostgreSQL business truth remains intact.
- Recovery resumes from outbox and pending stream entries.
- Cache failure must not cause event corruption.

## 19. Pending Entry Recovery

Workers inspect consumer-group pending entries:

- reclaim entries whose consumer lease expired
- preserve delivery count
- avoid stealing active long-running jobs without heartbeat
- dead-letter poison messages after policy threshold

## 20. Security

- Redis and PostgreSQL are private.
- Service identities have least privilege.
- Tenant context is validated before every tenant-owned side effect.
- Events are size-limited and schema-validated.
- Secrets and excessive personal data are prohibited.
- Manual replay and reconciliation are audited.
- Report generation enforces tenant filtering and export authorization.

## 21. Observability

Metrics:

- outbox pending age/count
- publish rate/failure
- stream length and consumer lag
- pending entries
- handler latency/success/failure
- retry and dead-letter count
- version gaps
- report duration/size
- notification channel outcomes
- delivery provider outcomes

Structured logs include event ID, consumer, tenant, aggregate, attempt, correlation ID, and result.

## 22. Shutdown and Deployment

Graceful shutdown:

1. Stop claiming new work.
2. Finish or safely release active leases.
3. Acknowledge only completed handlers.
4. Flush metrics/logs.
5. Close Redis/PostgreSQL/provider clients.

Worker deployment must support restart without lost or duplicated business side effects.

## 23. Testing

- Outbox crash-window tests.
- Consumer duplicate tests.
- Version-gap and ordering tests.
- Pending-entry reclaim tests.
- Retry classification tests.
- Dead-letter replay authorization tests.
- Redis outage/recovery tests.
- Notification deduplication tests.
- SSE projection leakage tests.
- Report memory and tenant-scope tests.
- Provider timeout/unknown-outcome tests.
- Load tests with bounded concurrency.

## 24. Acceptance Criteria

1. A committed critical state change cannot silently lose its event.
2. Duplicate events do not duplicate side effects.
3. Strict-order consumers detect version gaps.
4. Redis outage preserves recoverability from PostgreSQL.
5. Dead-letter replay is safe and audited.
6. SSE and reports preserve tenant/customer isolation.
7. Worker concurrency fits the target infrastructure.

## Change Summary

Initial worker and event-pipeline technical design synchronized with the final event, SSE, data, architecture, and security specifications.
