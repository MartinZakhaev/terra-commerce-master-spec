# ADR-0004 — Use Redis Streams for Internal Event Transport

**Status:** Final  
**Date:** 2026-06-30  
**Decision Owners:** Technical Lead  
**Related:** PRD v1.0.0, FSD v1.0.1

## Context

Terra Commerce requires asynchronous processing, event fan-out, retries, delivery tracking updates, notification triggers, and real-time UI refreshes. The MVP must remain lightweight on a 2 vCPU and 4 GB RAM server.

## Decision

Use Redis Streams as the MVP internal event transport.

Events must include:

- Event ID.
- Event type.
- Tenant ID where applicable.
- Resource type and ID.
- Timestamp.
- Payload.
- Schema version.
- Correlation ID where applicable.

Consumer groups are used for workers that require reliable processing. Retention, acknowledgement, retry, and dead-letter behavior must be defined in the event contract and operational runbook.

## Alternatives Considered

### Kafka
Rejected for MVP because operational and memory overhead are disproportionate to expected scale.

### Simple Redis Pub/Sub
Rejected as the primary event transport because messages are ephemeral and lack consumer acknowledgement and replay.

### Database outbox only
Not selected as the sole transport. An outbox pattern may still be added for transactionally critical events.

### RabbitMQ or NATS
Deferred because they add another service and operational burden without a proven MVP need.

## Consequences

### Positive

- Reuses Redis already required by the platform.
- Supports consumer groups and limited replay.
- Lower operational overhead than larger brokers.

### Negative

- Redis persistence and memory limits require careful configuration.
- Stream retention is not a permanent audit log.
- Transactional event publication may require an outbox pattern.

## Constraints

- Events must be idempotently consumable.
- Tenant context is mandatory for tenant-owned events.
- Sensitive data must be excluded.
- Consumers must acknowledge only after successful processing.
- Failed events must be retried and eventually dead-lettered.
- Stream data must not replace authoritative database state.

## Review Trigger

Review if event throughput, durability, retention, multi-region delivery, or independent service scaling exceed Redis Streams capabilities.
