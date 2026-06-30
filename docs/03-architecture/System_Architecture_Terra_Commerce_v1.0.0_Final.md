# System Architecture Document — Terra Commerce

**Document ID:** ARCH-TC-001  
**Version:** 1.0.0  
**Status:** Final  
**Owner:** Technical Lead  
**Last Updated:** 2026-06-30  
**Upstream:** PRD v1.0.0 Final, FSD v1.0.1 Draft, ADR-0001 through ADR-0007  
**Audit:** `reviews/System_Architecture_Audit_v1.0.0.md`

## 1. Purpose

This document defines the approved MVP system architecture for Terra Commerce, including runtime containers, module ownership, tenant isolation, request and event flows, integration boundaries, deployment, persistence, security, observability, backup and recovery, failure handling, and scaling direction.

## 2. Architecture Goals

1. Enforce tenant isolation at every boundary.
2. Keep deployment viable on 2 vCPU and 4 GB RAM.
3. Centralize public API ingress and security in the Go gateway.
4. Use Medusa.js v2 for core commerce capabilities.
5. Keep OMS and WMS as logical modules in a modular monolith.
6. Separate synchronous commands from asynchronous work.
7. Make integrations retryable and idempotent.
8. Preserve a measurable scaling path without premature microservices.

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

### Reverse Proxy

- TLS termination.
- Host and path routing.
- Static asset delivery where applicable.
- Security headers and request-size limits.
- SSE routing with buffering disabled.

### Go Gateway

Owns public ingress, authentication validation, tenant resolution, route-level authorization, rate limiting, correlation IDs, response normalization, superadmin APIs, Biteship webhook ingress, and SSE connections.

The gateway must not duplicate Medusa commerce logic.

### Medusa.js v2 Application

Owns products, variants, pricing, customers, carts, orders, promotions, inventory primitives, fulfillment, returns, and refund workflows.

Terra Commerce custom modules extend Medusa for tenant-aware commerce, OMS, WMS, inventory movements, plan enforcement hooks, and delivery workflows.

### Background Worker

Owns Redis Stream consumers, notifications, Biteship retries, delivery synchronization, heavy reports, cleanup, expiration tasks, and dead-letter handling.

### PostgreSQL

Stores authoritative transactional platform and commerce data, audit records, idempotency records, provider-event records, and reporting source data.

### Redis

Used for cache, rate-limit counters, short-lived locks, idempotency coordination, Redis Streams, and SSE fan-out support.

### Frontend Applications

- Superadmin web.
- Tenant admin web containing OMS and WMS modules.
- Storefront web.

Frontends consume only gateway APIs and never access Medusa, PostgreSQL, or Redis directly.

### External Object Storage

Stores tenant media, product images, generated reports, and shipping labels where applicable.

## 5. Logical Module Ownership

| Module | Owner |
|---|---|
| Platform administration | Go platform module |
| Identity and access | Gateway plus authentication module |
| Tenant configuration | Platform module |
| Catalog | Medusa |
| Customer and cart | Medusa |
| Orders and payments | Medusa plus payment adapter |
| OMS | Terra Commerce commerce module |
| Inventory and WMS | Medusa plus Terra Commerce module |
| Delivery | Biteship integration module |
| Events | Shared event infrastructure |
| Notifications | Worker module |
| Reporting | Worker/read-model module |
| Audit | Shared platform module |

Modules interact only through approved interfaces, workflows, repositories, or events. Direct access to another module's private internals is prohibited.

## 6. Internal Network and Trust Boundaries

- PostgreSQL and Redis are private services and must not be publicly exposed.
- Only approved application processes may access them.
- Internal service credentials are separate and least privilege.
- Public traffic terminates at the reverse proxy and enters through the gateway.
- External provider traffic is validated, rate limited where appropriate, and logged without secrets.
- Secrets are injected through environment or an approved secret-management mechanism and never committed to source control.

## 7. Tenant Resolution and Isolation

### Resolution Flow

1. Resolve tenant from approved host, path, authenticated membership, or trusted internal context.
2. Validate tenant existence and status.
3. Validate actor membership and permission.
4. Attach immutable tenant context to the request.
5. Propagate tenant context to downstream calls, database queries, jobs, events, cache keys, logs, files, and SSE.

### Isolation Rules

- Tenant-owned rows contain `tenant_id` or an enforceable ownership path.
- Tenant-aware repositories are mandatory.
- Cache keys use `tenant:{tenant_id}:...`.
- Object keys use `tenants/{tenant_id}/...`.
- Jobs and events persist tenant context.
- SSE payloads are filtered by tenant and audience.
- Cross-tenant tests are a release gate.

Plans, global settings, and feature definitions are explicitly global data and require global permissions.

## 8. Authorization Layers

1. Gateway authentication and route permission.
2. Tenant membership and tenant-status validation.
3. Downstream resource ownership validation.
4. Business-state eligibility.
5. Plan and feature availability where relevant.

Gateway checks never replace downstream ownership validation.

## 9. Synchronous Request Flow

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
    -> publish durable event where required
 -> normalized response
```

Provider calls use explicit timeouts and idempotency. Slow retryable work moves to the worker when product behavior permits.

## 10. Checkout and Order Flow

```text
Authenticated Customer
 -> Gateway
 -> Validate cart, tenant, prices, and inventory
 -> Transactional inventory reservation
 -> Order creation
 -> Payment initialization
 -> Durable domain event
 -> Customer and tenant UI refresh
```

Guarantees:

- Duplicate idempotency keys return the original result.
- Reservation and order creation remain consistent.
- Historical price, currency, tax, discount, shipping, and address snapshots are retained.
- Payment initialization failure follows a documented recoverable or rollback path.

## 11. OMS, WMS, and Delivery Flow

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

Warehouse exceptions create explicit records and pause progression. Stock receiving updates incoming, physical, and damaged quantities through auditable inventory movements.

## 12. Event Architecture

### Domain Event Envelope

Every event contains event ID, event type, tenant ID where applicable, resource type and ID, timestamp, payload, schema version, and correlation ID where applicable.

### Durable Publication Rule

A transactionally critical state change must use a durable publication mechanism so a committed database transaction cannot silently lose its required event.

The implementation must use a database outbox or an equivalent documented and tested mechanism.

### Redis Streams

- Consumer groups process asynchronous work.
- Consumers acknowledge only after success.
- Retries are bounded and use backoff.
- Failed events enter a dead-letter process.
- Consumers are idempotent.
- Streams are not permanent audit storage.

### SSE

Internal events are transformed into audience-safe SSE events. Raw internal payloads are never exposed directly. Clients use `Last-Event-ID` where replay exists and otherwise perform a full API refresh.

## 13. Biteship Integration

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
 -> durable event publication
 -> Redis Stream
 -> SSE and notifications
```

Unknown events are quarantined for review.

## 14. Data and Transaction Boundaries

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

## 15. Caching and File Architecture

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

## 16. Deployment and Persistence

```text
Single Linux VPS: 2 vCPU / 4 GB RAM

reverse-proxy      ephemeral configuration/container
├── gateway        stateless application process
├── commerce       stateless application process
├── worker         stateless application process
├── postgres       persistent volume
└── redis          bounded persistence where configured

External:
├── S3-compatible object storage
├── backup storage
├── Biteship
├── email provider
└── future payment provider
```

Application containers are replaceable. PostgreSQL data uses a persistent volume. Redis persistence is configured according to event and coordination requirements but Redis is never the sole authoritative recovery source.

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

## 17. Backup and Recovery Architecture

- PostgreSQL receives scheduled backups owned by the operations function.
- Backups are stored outside the application VPS.
- Restore procedures are tested before production acceptance and periodically thereafter.
- Recovery Point Objective and Recovery Time Objective are defined in the operational runbook.
- Object-storage retention and recovery follow provider capabilities and tenant-data policy.
- Redis Streams and cache are reconstructable operational data and are not the sole backup source for business truth.
- Release rollback instructions must account for schema compatibility.

## 18. Failure Handling

- Application processes use health checks, graceful shutdown, and bounded restart behavior.
- PostgreSQL failure blocks mutations and returns service-unavailable responses.
- Redis failure may degrade cache, but event, rate-limit, idempotency, and SSE impacts must be explicit in the runbook.
- Critical operations must not silently lose required event publication.
- Biteship rate failures return retryable user-facing errors.
- Shipment failures enter retryable pending or failed states without duplicate creation.
- Email failure never rolls back the business transaction.

## 19. Observability

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
- Backup success and restore-test status.
- Critical-action audit trail.

Sensitive data and secrets must be redacted.

## 20. Security Controls

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

## 21. Scaling Path

1. Tune application, database, and Redis configuration.
2. Keep object storage external.
3. Separate PostgreSQL.
4. Separate Redis.
5. Add workers.
6. Scale gateway and Medusa behind a load balancer.
7. Add shared SSE coordination.
8. Extract modules only when ADR-0007 criteria are met.

## 22. Repository and Release Architecture

The master-spec repository owns architecture and cross-application contracts. Application repositories live as pinned submodules under `apps/`.

A coordinated release records approved specification versions, exact submodule commits, database migrations, deployment version, release notes, backup implications, and rollback instructions.

## 23. Open Decisions

Future ADRs are required for frontend framework, authentication/session strategy, payment provider, tenant URL model, Biteship credential ownership, object storage, email provider, reservation expiration, order numbering, tax policy, and customer identity uniqueness.

## 24. Downstream Documents

This architecture governs technical design, database design, state machines, OpenAPI, event/SSE/webhook contracts, authorization matrix, infrastructure specification, threat model, test strategy, and operational runbook.

## 25. Acceptance Criteria

1. Every public request passes through the gateway.
2. PostgreSQL and Redis are private internal services.
3. Tenant context is preserved across all boundaries.
4. Gateway and Medusa ownership do not conflict.
5. OMS and WMS remain logical modules.
6. Redis Streams and SSE have distinct responsibilities.
7. Critical events use durable publication.
8. Biteship operations are idempotent and retryable.
9. Persistent and ephemeral runtime data are clearly separated.
10. Backup and restore ownership is explicit.
11. Deployment can be load tested on 2 vCPU and 4 GB RAM.
12. Security, observability, and failure behavior are explicit.
13. The scaling path preserves core domain ownership.

## Change Summary

Final version incorporates all findings from `System_Architecture_Audit_v1.0.0.md`, including backup and recovery, persistence boundaries, durable event publication, and internal network access rules.
