# Technical Design Document — Go Gateway

**Document ID:** TDD-TC-GW-001  
**Version:** 1.0.0  
**Status:** Final  
**Owner:** Gateway Lead  
**Last Updated:** 2026-06-30  
**Upstream:** Final Architecture, OpenAPI, SSE, Authorization Matrix, Threat Model  
**Audit:** `reviews/Technical_Design_Suite_Audit_v1.0.0.md`

## 1. Responsibilities

The Go gateway owns public `/api/v1` ingress, authentication validation, tenant resolution, route authorization, rate limiting, correlation IDs, request validation, idempotency orchestration, optimistic-concurrency headers, normalized errors/responses, platform APIs, Biteship webhook ingress, and SSE connections.

It does not own commerce state machines or duplicate Medusa/Terra domain rules.

## 2. Project Structure

```text
cmd/gateway
internal/{app,config,http,auth,tenant,authorization,idempotency,platform,commerceclient,webhook,sse,observability,health,security,errors,testkit}
pkg/contracts
```

Handlers depend on application interfaces. Infrastructure adapters implement those interfaces.

## 3. Request Pipeline

1. Panic recovery.
2. Body/header limits.
3. Correlation ID.
4. Structured request context.
5. Maintenance check.
6. Authentication.
7. Tenant resolution and status.
8. Audience/permission check.
9. Rate limit.
10. Idempotency acquisition when required.
11. Schema validation.
12. Handler.
13. Error normalization.
14. Metrics/audit outcome hooks.

Webhook, public catalog, authentication, and SSE routes use specialized variants without bypassing security controls.

## 4. Tenant Context

Tenant context is server-resolved from the approved URL model, authenticated membership, or trusted internal context. Body-provided tenant IDs are non-authoritative.

Context contains tenant ID, status, plan, timezone, currency, and resolution source. Conflicting sources are rejected. The context is immutable for the request.

## 5. Authentication and Authorization

Authentication remains behind a provider-neutral interface and returns principal, audience, session, roles/memberships, and privileged MFA claim where required.

The authorization engine evaluates the OpenAPI `operationId` against the machine-checkable permission registry, tenant membership, support session, plan/feature policy, and route audience. Downstream modules independently enforce resource ownership, state, and version.

## 6. Internal Gateway-to-Commerce Trust Contract

Every internal call must include:

- dedicated service identity
- intended internal audience
- tenant context and principal reference
- correlation ID
- operation ID
- request timestamp and nonce, or equivalent replay protection
- body/context integrity signature, or mutually authenticated transport with an equivalent trusted-context mechanism
- explicit deadline

Commerce validates service identity, audience, freshness, replay identity, signature/integrity, and tenant context. Plain client-forwarded tenant headers are never trusted. Keys are rotatable and requests outside the accepted clock/replay window are rejected.

## 7. Handler and Client Rules

Handlers parse validated input, load request context, invoke one application operation, map DTOs, and return normalized envelopes. They do not contain duplicated state logic or expose Medusa internals.

Downstream retries are allowed only for safe reads or idempotent commands. Every call has timeout, correlation ID, tenant context, and typed error mapping.

## 8. Idempotency

Required commands validate key format, calculate canonical request hash, acquire scoped PostgreSQL-backed identity, reject payload mismatch, replay completed responses, and store result/resource references. Redis may coordinate in-flight work but is not authoritative.

## 9. Optimistic Concurrency

Versioned commands require `If-Match` or contract-defined version. Missing precondition is rejected; stale version returns `state_conflict`. The gateway never automatically retries a state conflict.

## 10. Error Contract

Internal errors map to canonical public errors. Unknown errors expose only correlation ID. Status mappings are fixed by `openapi.yaml`; provider/database internals are never returned.

## 11. SSE Design

Components:

- connection authenticator
- audience and permission resolver
- bounded connection registry
- audience-scoped replay adapter
- event projector/filter
- heartbeat writer
- shutdown coordinator

Replay revalidates current permission and ownership. Raw domain events never reach clients. Slow consumers are disconnected. Suspension, revocation, expiry, and shutdown terminate streams safely.

## 12. Webhook Ingress

The gateway preserves raw body, verifies current official Biteship authentication through a versioned adapter, applies limits, persists/dispatches normalized receipt, and acknowledges only when processing or durable retry is guaranteed by the final webhook contract.

## 13. Platform Capabilities

In-process platform services may implement tenant, plan, usage, settings, feature flags, maintenance, support sessions, health, and failure inspection. They use explicit repositories and transaction boundaries.

## 14. Rate Limiting

Dimensions include route, IP/network signal, principal/customer, tenant, and provider identity. High-risk authentication and mutation routes fail closed if rate-limit enforcement is unavailable; low-risk read degradation is policy controlled.

## 15. Observability and Security

Every request records operation, tenant, pseudonymous principal, latency, status, downstream outcome, state conflict, idempotency result, and correlation ID. Logs exclude tokens, secrets, payment data, and unnecessary PII.

Security includes trusted-proxy configuration, CORS allowlist, CSRF when cookie-based, body limits, no arbitrary URL fetches, least-privilege service credentials, startup configuration validation, and secure shutdown.

## 16. Health and Shutdown

Liveness checks process health. Readiness validates required configuration and route-critical dependencies.

Shutdown stops new traffic, signals SSE clients where possible, drains requests, closes streams, flushes telemetry, and closes dependencies within a bounded timeout.

## 17. Testing

- Middleware, tenant resolution, permission registry, idempotency, error mapping.
- Internal trust signature/replay tests.
- Contract tests against `openapi.yaml`.
- Cross-tenant/customer tests.
- SSE replay/revocation/load tests.
- Webhook verification/acknowledgement tests.
- Integration and target-host load tests.

## 18. Acceptance Criteria

1. Every OpenAPI operation maps to a handler and authorization rule.
2. Client input cannot override tenant context.
3. Internal trust context is authenticated, integrity protected, and replay resistant.
4. Idempotency/concurrency match contracts.
5. SSE and webhook paths meet security and durability rules.
6. No commerce state logic is duplicated.

## Change Summary

Final version closes TDD-M01 by defining a concrete authenticated, integrity-protected, replay-resistant gateway-to-commerce trust contract.
