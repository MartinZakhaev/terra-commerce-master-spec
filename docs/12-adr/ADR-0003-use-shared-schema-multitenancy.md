# ADR-0003 — Use Shared-Database Shared-Schema Multitenancy

**Status:** Final  
**Date:** 2026-06-30  
**Decision Owners:** Product Owner, Technical Lead, Security Owner  
**Related:** PRD v1.0.0, FSD v1.0.1

## Context

The MVP must support multiple tenants on a single 2 vCPU and 4 GB RAM server. Dedicated databases or schemas per tenant would increase operational overhead and resource usage. At the same time, tenant data isolation is a critical security requirement.

## Decision

Use one PostgreSQL database and one shared schema for MVP.

Tenant-owned records must include `tenant_id` directly or have a documented, enforceable ownership path through a parent entity.

Isolation is enforced through:

- Mandatory tenant context.
- Tenant-aware repositories and services.
- Server-side authorization and ownership checks.
- Database constraints and tenant-aware indexes.
- Tenant-scoped cache keys, files, events, jobs, and logs.
- Automated cross-tenant isolation tests.

PostgreSQL Row-Level Security is not mandatory for MVP but remains a future defense-in-depth option.

## Alternatives Considered

### Database per tenant
Rejected for MVP because of connection, migration, backup, and operational overhead.

### Schema per tenant
Rejected because schema migration and connection management complexity is disproportionate to MVP scale.

### Shared schema with Row-Level Security from day one
Deferred because it adds operational and query-context complexity before the access patterns are validated.

## Consequences

### Positive

- Lowest infrastructure overhead.
- Simple migration and backup model.
- Efficient use of constrained server resources.

### Negative

- Application defects could create cross-tenant exposure.
- Every data-access path must preserve tenant context.
- Unique constraints often require tenant-aware composite keys.

## Constraints

- No client-provided tenant ID is trusted without validation.
- Global records must be explicitly classified as global.
- Tenant-owned unique values use tenant-aware uniqueness where appropriate.
- Background jobs and events must retain tenant context.
- Cross-tenant test coverage is a release gate.

## Review Trigger

Review when tenant count, compliance requirements, enterprise isolation needs, noisy-neighbor behavior, or database scaling justify schema or database separation.
