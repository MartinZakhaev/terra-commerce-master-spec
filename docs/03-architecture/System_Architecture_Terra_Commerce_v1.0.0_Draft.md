# System Architecture Document — Terra Commerce

**Document ID:** ARCH-TC-001  
**Version:** 1.0.0  
**Status:** Draft  
**Owner:** Technical Lead  
**Last Updated:** 2026-06-30  
**Upstream:** PRD v1.0.0 Final, FSD v1.0.1 Draft, ADR-0001 through ADR-0007

## 1. Purpose

This document defines the MVP system architecture for Terra Commerce: component ownership, runtime topology, tenant isolation, request and event flows, integration boundaries, deployment, security, observability, failure handling, and scaling direction.

## 2. Architecture Goals

1. Enforce tenant isolation at every boundary.
2. Keep deployment viable on 2 vCPU and 4 GB RAM.
3. Centralize public API security in the Go gateway.
4. Use Medusa.js v2 for core commerce behavior.
5. Keep OMS and WMS as logical modules in a modular monolith.
6. Separate synchronous commands from asynchronous processing.
7. Make integrations retryable and idempotent.
8. Preserve a clear future scaling path.

## 3. System Context

```text
Superadmin / Tenant Staff / Customer
                |
          Web Applications
                |
          Reverse Proxy
                |
            Go Gateway
        ________|________
       |        |        |
    Medusa   Platform    SSE
       |      Modules   Streams
       |        |
   PostgreSQL  Redis <-> Worker
                         |
             Biteship / Email / Object Storage
```

All public application APIs pass through the Go gateway.

## 4. Runtime Containers

### 4.1 Reverse Proxy

- TLS termination.
- Host and path routing.
- Static asset delivery where applicable.
- Security headers and request-size limits.
- SSE routing with proxy buffering disabled.

### 4.2 Go Gateway

Owns:

- Public API ingress.
- Authentication validation.
- Tenant resolution.
- Route-level authorization.
- Rate limiting.
- Correlation IDs and request logging.
- Response normalization.
- Superadmin and platform APIs.
- Biteship webhook ingress.
- SSE connection management.

It must not duplicate Medusa commerce logic.

### 4.3 Medusa.js v2 Application

Owns:

- Products, variants, pricing, customers, carts, orders, promotions, inventory primitives, fulfillment, returns, and refund workflows.
- Terra Commerce custom modules for tenant-aware commerce, OMS, WMS, inventory movements, and delivery integration.

### 4.4 Background Worker

Owns:

- Redis Stream consumers.
- Notifications.
- Biteship retries and synchronization.
- Heavy reports.
- Cleanup and expiration tasks.
- Dead-letter support.

### 4.5 PostgreSQL

Stores transactional platform and commerce data, audit records, idempotency records, provider-event records, and reporting source data.

### 4.6 Redis

Used for cache, rate limits, short-lived locks, idempotency coordination, Redis Streams, and SSE fan-out support.

### 4.7 Frontend Applications

- Superadmin web.
- Tenant admin web containing OMS and WMS modules.
- Storefront web.

Frontends consume gateway APIs only. Static builds are preferred where feasible.

### 4.8 External Object Storage

Stores tenant media, product images, private report files, and shipping labels where applicable.

## 5. Logical Modules

| Module | Owner |
|---|---|
| Platform administration | Go platform module |
| Identity and access | Gateway plus authentication module |
| Tenant configuration | Platform module |
| Catalog | Medusa |
| Customer and cart | Medusa |
| Orders and payments | Medusa plus payment adapter |
| OMS | Terra Commerce module |
| Inventory and WMS | Medusa plus Terra Commerce module |
| Delivery | Biteship integration module |
| Events | Shared event infrastructure |
| Notifications | Worker module |
| Reporting | Worker/read-model module |
| Audit | Shared platform module |

Modules may interact only through approved interfaces, workflows, repositories, or events. Direct access to another module's private internals is prohibited.

## 6. Tenant Resolution and Isolation

### 6.1 Resolution

1. Resolve tenant from approved host, path, authenticated membership, or trusted internal context.
2. Validate tenant existence and status.
3. Validate actor membership and permission.
4. Attach immutable tenant context to the request.
5. Propagate tenant context to downstream calls, database queries, jobs, events, cache keys, logs, files, and SSE.

### 6.2 Isolation Rules

- Tenant-owned rows contain `tenant_id` or an enforceable ownership path.
- Tenant-aware repositories are mandatory.
- Cache keys use `tenant:{tenant_id}:...`.
- Object keys use `tenants/{tenant_id}/...`.
- Jobs and events persist tenant context.
- SSE payloads are filtered by tenant and audience.
- Cross-tenant tests are a release gate.

Global plans, feature definitions, and platform settings are explicitly classified as global data.

## 7. Authorization Layers

1. Gateway authentication and route permission.
2. Tenant membership and tenant-status validation.
3. Downstream resource ownership validation.
4. Business-state eligibility.
5. Plan and feature availability where applicable.

Gateway checks never replace downstream ownership validation.

## 8. Synchronous Request Flow

```text
Client
 -> Reverse Proxy
 -> Go Gateway
    -> authenticate
    -> resolve tenant
    -> authorize
    -> validate
 -> Medusa or Platform Module
    -> validate ownership and state
    -> execute transaction
    -> persist data
    -> publish or queue event
 -> normalized response
```

Provider calls use explicit timeouts and idempotency. Slow retryable work moves to the worker where product behavior permits.

## 9. Checkout and Order Flow

```text
Authenticated Customer
 -> Gateway
 -> Cart, price, tenant, and inventory validation
 -> Transactional inventory reservation
 -> Order creation
 -> Payment initialization
 -> Domain event
 -> Customer and tenant UI refresh
```

Required guarantees:

- Duplicate idempotency keys return the original result.
- Reservation and order creation remain consistent.
- Historical price, currency, and address snapshots are retained.
- Payment initialization failure has a defined recoverable or rollback path.

## 10. OMS, WMS, and Delivery Flow

```text
Payment Confirmed
 -> Ready for Fulfillment
 -> Picking
 -> Packing
 -> Final Package Data
 -> Ready for Shipment
 -> Biteship Shipment
 -> Tracking Updates
 -> Delivered / Failed / Returned
```

Warehouse exceptions create explicit records and pause progression. Stock receiving updates incoming, physical, and damaged quantities through auditable movements.

## 11. Event Architecture

### 11.1 Domain Events

Events contain event ID, type, tenant ID where applicable, resource type and ID, timestamp, payload, schema version, and correlation ID where applicable.

### 11.2 Redis Streams

- Consumer groups process asynchronous work.
- Consumers acknowledge only after success.
- Retries are bounded and use backoff.
- Failed events move to a dead-letter process.
- Consumers are idempotent.
- Streams are not permanent audit storage.

For transactionally critical events, technical design must evaluate a database outbox.

### 11.3 SSE

Internal events are transformed into audience-safe SSE events. Raw internal payloads are never exposed directly. Clients use `Last-Event-ID` where replay exists and otherwise perform a full API refresh.

## 12. Biteship Integration

### Outbound

- Rate requests may be synchronous with strict timeouts.
- Shipment creation is idempotent and retryable.
- Credentials come from secure configuration.

### Inbound

```text
Biteship Webhook
 -> Gateway verification
 -> provider-event deduplication
 -> tenant and shipment resolution
 -> normalized state transition
 -> persistence
 -> Redis Stream
 -> SSE and notifications
```

Unknown events are quarantined for review.

## 13. Data and Transaction Boundaries

Strong transaction boundaries are required for:

- Tenant provisioning where practical.
- Order creation and inventory reservation.
- Reservation release.
- Manual payment confirmation.
- Stock receiving completion.
- Fulfillment inventory deduction.
- Cancellation and compensating inventory action.
- Shipment identity and idempotency persistence.
- Refund initiation record.

External providers cannot join local database transactions and therefore require idempotent state machines and compensating behavior.

## 14. Caching and Files

Caching rules:

- Tenant namespace on every tenant-owned key.
- Bounded TTL.
- Explicit invalidation for critical settings.
- No secrets in general-purpose cache.
- Safe degradation to authoritative data where possible.

File rules:

- External S3-compatible storage preferred.
- Tenant-scoped object keys.
- File type and size validation.
- Signed or authorized access for private reports and labels.
- Database stores metadata and references, not large binaries.

## 15. Deployment Architecture

```text
Single Linux VPS: 2 vCPU / 4 GB RAM

reverse-proxy
├── gateway
├── commerce
├── worker
├── postgres
└── redis

External:
├── S3-compatible object storage
├── Biteship
├── email provider
└── future payment provider
```

Planning memory targets:

| Component | Target |
|---|---:|
| Gateway | 100–250 MB |
| Medusa | 500–900 MB |
| Worker | 250–600 MB |
| PostgreSQL | 700–1,200 MB |
| Redis | 100–300 MB |
| Proxy and OS | 100–250 MB |

Controls include Node.js heap limits, controlled worker concurrency, PostgreSQL tuning, Redis memory limits, static frontend builds, and external object storage. Load testing is mandatory.

## 16. Failure Handling

- Application processes use health checks, graceful shutdown, and bounded restart behavior.
- PostgreSQL failure blocks mutations and returns service-unavailable responses.
- Redis failure may degrade cache, but event, rate-limit, idempotency, and SSE impacts must be explicit in the runbook.
- Biteship rate failures are retryable user-facing errors.
- Shipment failures enter retryable pending or failed states without duplicate creation.
- Email failure never rolls back the business transaction.

## 17. Observability

Required telemetry:

- Structured logs and correlation IDs.
- Tenant ID where appropriate.
- Request latency and error rate.
- Authentication and authorization failures.
- Worker delay and failures.
- Redis Stream lag.
- Active SSE connections.
- Biteship request and webhook outcomes.
- PostgreSQL, Redis, CPU, memory, and disk health.
- Critical-action audit trail.

Sensitive data and secrets must be redacted.

## 18. Security Controls

- HTTPS only.
- Secrets outside source control.
- Secure password hashing.
- Parameterized database access.
- Input validation and output encoding.
- CSRF protection where required.
- Security headers.
- Rate limiting.
- Tenant and ownership checks.
- Webhook verification.
- Least-privilege service credentials.
- Dependency and container scanning.
- Append-only audit behavior from normal application paths.

A dedicated threat model is required downstream.

## 19. Scaling Path

1. Tune application, database, and Redis configuration.
2. Move object storage externally.
3. Separate PostgreSQL.
4. Separate Redis.
5. Add workers.
6. Scale gateway and Medusa behind a load balancer.
7. Add shared SSE coordination.
8. Extract modules only when ADR-0007 criteria are met.

## 20. Repository and Release Architecture

The master-spec repository owns architecture and cross-application contracts. Application repositories live as pinned submodules under `apps/`.

A coordinated release records approved specification versions, exact submodule commits, database migrations, deployment version, release notes, and rollback instructions.

## 21. Open Decisions

Future ADRs are required for frontend framework, authentication/session strategy, payment provider, tenant URL model, Biteship credential ownership, object storage, email provider, reservation expiration, order numbering, tax policy, and customer identity uniqueness.

## 22. Downstream Documents

This architecture governs technical design, database design, state machines, OpenAPI, event/SSE/webhook contracts, authorization matrix, infrastructure specification, threat model, test strategy, and operational runbook.

## 23. Acceptance Criteria

1. Every public request passes through the gateway.
2. Tenant context is preserved across all boundaries.
3. Gateway and Medusa ownership do not conflict.
4. OMS and WMS remain logical modules.
5. Redis Streams and SSE have distinct responsibilities.
6. Biteship operations are idempotent and retryable.
7. Critical transaction boundaries are identified.
8. Deployment can be load tested on 2 vCPU and 4 GB RAM.
9. Security, observability, and failure behavior are explicit.
10. The scaling path preserves core domain ownership.

## Change Summary

Initial architecture draft derived from the approved PRD, reviewed FSD, and final ADR baseline.
