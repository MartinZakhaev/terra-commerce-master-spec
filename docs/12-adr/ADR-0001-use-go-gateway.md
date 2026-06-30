# ADR-0001 — Use Go as the Public API Gateway

**Status:** Final  
**Date:** 2026-06-30  
**Decision Owners:** Product Owner, Technical Lead  
**Related:** PRD v1.0.0, FSD v1.0.1

## Context

Terra Commerce requires a single public API entry point for authentication validation, tenant resolution, authorization, rate limiting, response normalization, correlation IDs, structured logging, platform APIs, and SSE connections. The MVP must run on 2 vCPU and 4 GB RAM.

## Decision

Use Go for the public API gateway.

The gateway owns:

- Public API ingress.
- Authentication validation.
- Tenant resolution.
- Authorization enforcement.
- Rate limiting.
- Request validation and routing.
- Correlation IDs and structured logging.
- Response normalization.
- Global platform APIs.
- SSE connection handling.

The gateway must not duplicate Medusa commerce logic. Commerce-domain commands are delegated to Medusa or Terra Commerce modules behind documented contracts.

## Alternatives Considered

### Node.js gateway
Rejected for MVP because it would increase shared runtime memory pressure beside Medusa and provide less isolation between gateway and commerce-process resource usage.

### Direct frontend-to-Medusa access
Rejected because it weakens centralized tenant resolution, global authorization, rate limiting, API normalization, and SSE ownership.

### API gateway product only
Rejected as insufficient by itself because Terra Commerce requires custom tenant-aware orchestration and platform APIs.

## Consequences

### Positive

- Efficient long-lived SSE handling.
- Predictable memory usage.
- Centralized security boundary.
- Clear separation between platform concerns and commerce logic.

### Negative

- Two backend ecosystems must be maintained.
- Shared contracts between Go and Medusa are mandatory.
- Developers must avoid duplicating domain rules.

## Constraints

- Every tenant request carries validated tenant context.
- Internal calls use authenticated trusted context.
- Public endpoints must be defined in OpenAPI.
- Gateway authorization does not replace downstream resource-ownership validation.

## Review Trigger

Review if gateway complexity becomes a bottleneck, if independently scaled services are introduced, or if measured operational cost outweighs the separation benefits.
