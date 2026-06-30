# Technical Design Document — Go Gateway

**Document ID:** TDD-TC-GW-001  
**Version:** 1.0.0  
**Status:** Draft  
**Owner:** Gateway Lead  
**Last Updated:** 2026-06-30  
**Upstream:** System Architecture v1.0.0 Final, OpenAPI Contract v1.0.0 Final, SSE Contract v1.0.0 Final, Authorization Matrix v1.0.0 Final, Security Threat Model v1.0.0 Final

## 1. Purpose

This document defines the implementation design for the Go public API gateway, including module layout, middleware order, tenant resolution, authorization, routing, request/response handling, idempotency, concurrency, SSE, webhook ingress, observability, failure behavior, and testing.

## 2. Responsibilities

The gateway owns:

- Public HTTP ingress under `/api/v1`.
- Authentication validation.
- Tenant resolution.
- Route-level authorization.
- Resource context propagation.
- Rate limiting.
- Correlation IDs.
- Request validation.
- Idempotency orchestration for public commands.
- Optimistic concurrency header handling.
- Response and error normalization.
- Platform administration APIs.
- Biteship webhook ingress.
- SSE connections and audience filtering.
- Internal routing to Medusa and platform modules.

The gateway does not own core commerce business rules.

## 3. Proposed Project Structure

```text
cmd/gateway/main.go
internal/
  app/
  config/
  http/
    router/
    middleware/
    handlers/
    response/
    validation/
  auth/
  tenant/
  authorization/
  idempotency/
  platform/
  commerceclient/
  webhook/
  sse/
  observability/
  health/
  security/
  errors/
  testkit/
pkg/contracts/
```

Package boundaries follow dependency inversion. Handlers depend on application interfaces, not infrastructure implementations.

## 4. Request Pipeline

Middleware order:

1. Panic recovery.
2. Request body and header limits.
3. Correlation ID generation/validation.
4. Structured request logging context.
5. Maintenance-mode check.
6. Authentication extraction and validation.
7. Tenant resolution.
8. Tenant status validation.
9. Route audience and permission check.
10. Rate limiting.
11. Idempotency acquisition where required.
12. Request schema validation.
13. Handler execution.
14. Error normalization.
15. Metrics and audit outcome hooks.

Webhook and public storefront routes use specialized pipelines while preserving correlation, limits, rate limiting, and validation.

## 5. Tenant Resolution

Supported resolvers are selected by a future Tenant URL ADR:

- Host/subdomain.
- Path prefix.
- Authenticated membership context.
- Trusted internal service context.

Resolution result:

```go
type TenantContext struct {
    TenantID   uuid.UUID
    Status     string
    PlanID     uuid.UUID
    Timezone   string
    Currency   string
    Source     string
}
```

Rules:

- Request-body tenant IDs are never authoritative.
- Conflicting host, path, and authenticated context is rejected.
- Tenant context is immutable within request scope.
- Internal downstream requests include signed or mutually authenticated context.

## 6. Authentication Interface

Authentication implementation remains behind an interface until the relevant ADR is final.

```go
type Principal struct {
    SubjectID   uuid.UUID
    SubjectType string
    SessionID   string
    GlobalRoles []string
    Memberships []MembershipClaim
}
```

The gateway requires:

- authentication validation
- session/token expiry
- revocation lookup
- audience classification
- privileged MFA claim for global sensitive actions

## 7. Authorization Engine

Authorization input:

- OpenAPI `operationId`
- principal
- tenant context
- permission registry entry
- resource ownership metadata when available
- support-session metadata
- plan/feature state

The operation-permission registry is generated from the final Authorization Matrix and OpenAPI metadata. CI rejects unmapped operations.

Route checks do not replace downstream resource ownership and state validation.

## 8. Handler Design

Handlers perform:

1. Parse validated path/query/body input.
2. Read principal and tenant context.
3. Call one application service or downstream client operation.
4. Map domain result to contract DTO.
5. Return normalized envelope.

Handlers must not:

- query PostgreSQL directly unless they belong to the platform module and use approved repositories
- contain state-machine logic duplicated from domain modules
- expose Medusa internal response shapes

## 9. Downstream Communication

The gateway communicates with:

- Platform application services in-process where deployed together.
- Medusa application over private HTTP or an approved internal interface.
- Redis for rate limits, short-lived idempotency coordination, SSE fan-out, and caches.
- PostgreSQL through platform repositories where required.

Internal calls require:

- correlation ID
- tenant context
- authenticated service identity
- deadline/timeout
- normalized error mapping
- retry only for safe/idempotent requests

## 10. Idempotency Design

For required operations:

1. Validate `Idempotency-Key` format.
2. Calculate canonical request hash.
3. Acquire scoped idempotency record/lock.
4. Reject same key with different request hash.
5. Return stored completed response for duplicates.
6. Execute command once.
7. Persist final resource/result reference and response.
8. Release transient lock.

PostgreSQL records are authoritative. Redis may coordinate in-flight requests but cannot be the only idempotency store.

## 11. Concurrency Design

Versioned commands require `If-Match` or explicit version field. The gateway parses the expected version and passes it downstream.

- Stale version maps to `409 state_conflict`.
- Missing required version maps to `428 Precondition Required` or normalized validation error according to machine-readable OpenAPI.
- Gateway never retries state conflicts automatically.

## 12. Error Mapping

Internal errors map to canonical public errors. Unknown internal errors return `internal_error` with correlation ID and no sensitive detail.

Recommended status mapping:

- validation: 400
- authentication: 401
- forbidden: 403
- not found: 404
- state conflict: 409
- plan/rate limit: 409 or 429 according to contract
- invalid state: 422 or 409 according to operation schema
- provider unavailable: 503
- maintenance: 503

Exact mappings are fixed in `openapi.yaml`.

## 13. SSE Implementation

Components:

- connection authenticator
- audience resolver
- connection registry
- replay store adapter
- domain-event projector
- permission filter
- bounded per-connection buffer
- heartbeat writer
- shutdown coordinator

Rules:

- Audience-scoped replay IDs.
- Permission and tenant revalidation before replay.
- Raw domain events never reach clients.
- Slow consumers are disconnected with refresh guidance.
- Tenant suspension and session revocation terminate streams.
- Proxy buffering is disabled only for SSE route.

## 14. Biteship Webhook Ingress

The gateway:

- preserves raw body
- validates official provider authentication
- applies request limits and rate limits
- extracts normalized receipt metadata
- calls the integration application service
- returns acknowledgement only according to the final webhook contract

Provider-specific parsing is isolated behind a versioned adapter.

## 15. Platform Modules in Gateway

In-process platform capabilities may include:

- tenant management
- plan and usage management
- global settings and feature flags
- maintenance mode
- support-session management
- platform health and failure inspection

Each capability uses application services and repositories with explicit transaction boundaries.

## 16. Rate Limiting

Rate-limit dimensions:

- route class
- source IP/network signal
- principal/customer
- tenant
- provider webhook identity

Policies distinguish login, password reset, catalog reads, rates, checkout, reports, SSE connection attempts, and webhooks.

Redis-backed counters use bounded TTL. Failure behavior is fail-closed for high-risk mutation and authentication routes, configurable for low-risk reads.

## 17. Observability

Every request records:

- correlation ID
- operation ID
- audience
- tenant ID where applicable
- principal type and pseudonymous ID
- status code
- latency
- downstream dependency outcome
- idempotency and state-conflict indicators

Logs exclude tokens, secrets, raw payment data, and unnecessary personal data.

Metrics include request count/latency/errors, active SSE connections, replay misses, authorization denials, rate-limit events, webhook outcomes, and downstream timeouts.

## 18. Security Controls

- Secure defaults and explicit timeouts.
- Request body limits.
- Header normalization.
- Trusted proxy configuration.
- CORS allowlist.
- CSRF control when cookie sessions are selected.
- Output encoding delegated to clients but response content remains structured.
- No arbitrary outbound URL fetching.
- Secret configuration never returned.
- Service credentials use least privilege.

## 19. Configuration

Configuration is environment-driven and validated at startup:

- listener and shutdown timeouts
- trusted proxies and CORS origins
- PostgreSQL and Redis connections
- Medusa internal URL and service credential
- SSE heartbeat, buffer, replay age/count
- rate-limit profiles
- body-size limits
- provider verification configuration
- telemetry settings

Startup fails on missing required secure configuration.

## 20. Health and Shutdown

Endpoints:

- liveness: process is running
- readiness: required dependencies and routing are usable

Graceful shutdown:

1. Stop accepting new requests.
2. Send `system.shutdown` to SSE clients where safe.
3. Drain requests within timeout.
4. Close SSE connections.
5. Flush telemetry.
6. Close dependencies.

## 21. Testing Strategy

- Unit tests for middleware, tenant resolution, permission registry, error mapping, idempotency, and projectors.
- Contract tests against `openapi.yaml`.
- Integration tests with Medusa, PostgreSQL, and Redis.
- Cross-tenant and cross-customer tests.
- SSE replay, revocation, slow-consumer, and suspension tests.
- Webhook verification and acknowledgement tests.
- Load tests on target VPS profile.
- Fuzz tests for parsers and webhook adapters where useful.

## 22. Acceptance Criteria

1. Every OpenAPI operation has a handler and permission mapping.
2. Tenant context cannot be overridden by client input.
3. Idempotency and concurrency behavior match final contracts.
4. SSE is audience-safe and bounded.
5. Webhook acknowledgement matches durable processing guarantees.
6. Gateway contains no duplicated commerce state logic.
7. Logs and metrics are sufficient for audit and operations.

## Change Summary

Initial Go gateway technical design synchronized with all finalized architecture, contract, security, and data documents.
