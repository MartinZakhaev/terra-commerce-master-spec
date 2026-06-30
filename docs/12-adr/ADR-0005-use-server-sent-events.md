# ADR-0005 — Use Server-Sent Events for Real-Time UI Updates

**Status:** Final  
**Date:** 2026-06-30  
**Decision Owners:** Product Owner, Technical Lead  
**Related:** PRD v1.0.0, FSD v1.0.1

## Context

Terra Commerce requires server-to-client updates for orders, payments, inventory, warehouse tasks, shipments, alerts, and notifications. Clients send commands through normal HTTP APIs; the dominant real-time direction is server to client.

## Decision

Use Server-Sent Events through the Go gateway for MVP real-time UI updates.

Requirements:

- Authenticated connections.
- Tenant-scoped subscriptions.
- Customer-scoped filtering where applicable.
- Audience-specific payload filtering.
- Heartbeats.
- Automatic client reconnection.
- `Last-Event-ID` support where replay is available.
- Full API refresh fallback when replay is unavailable.
- Per-tenant and per-user connection limits.

SSE events notify clients of changes; clients fetch authoritative state from APIs when necessary.

## Alternatives Considered

### WebSocket
Rejected for MVP because bidirectional persistent communication is not required for the primary use cases and would add protocol and connection-management complexity.

### Polling only
Rejected because it increases repeated API traffic and delays operational updates.

### Managed real-time provider
Deferred to avoid additional cost and dependency during MVP.

## Consequences

### Positive

- Simple browser-native server-to-client transport.
- Efficient fit with the Go gateway.
- Lower complexity than WebSocket for one-way updates.

### Negative

- Client commands still require separate HTTP requests.
- Connection limits and proxy timeouts require tuning.
- Replay is bounded by the event-retention design.

## Constraints

- SSE must not expose raw internal events directly.
- Payloads must be audience filtered.
- Tenant suspension terminates relevant streams.
- Reverse proxy buffering must be disabled for SSE endpoints.
- Connections must not exhaust the 2 vCPU and 4 GB RAM target.

## Review Trigger

Review if collaborative bidirectional features, high-frequency client messages, mobile-background constraints, or connection scale justify WebSocket or a managed real-time service.
