# ADR-0016 — Use Tenant-Date-Sequence Order Numbers

**Status:** Final  
**Date:** 2026-07-01  
**Decision Owners:** Product Owner, Commerce Lead, Database Lead

## Context

Orders need a human-readable reference for customers, operations, support, and providers while retaining an immutable internal identifier.

## Decision

Use the format:

`{TENANT_CODE}-{YYYYMMDD}-{6_DIGIT_SEQUENCE}`

Example: `ACME-20260701-000123`.

Rules:

- Internal UUID remains the authoritative primary identifier.
- Tenant code is immutable once commercial use begins.
- The sequence is unique within tenant and local calendar date.
- Number allocation is transactional and safe under concurrency.
- Cancelled or failed orders never cause number reuse.
- Order number is immutable and unique by `(tenant_id, order_number)`.
- Provider integrations use stable internal/provider references and may also carry the order number as display metadata.

## Alternatives Considered

- UUID only: rejected because it is difficult for customer and operational use.
- Global sequence: rejected because it leaks platform-wide order volume and creates unnecessary contention.
- Random short code: rejected because collision management and chronological support are weaker.

## Consequences

Positive: readable, tenant-scoped, sortable by date, and suitable for support.

Negative: requires a concurrency-safe tenant/date sequence allocator and stable tenant codes.

## Review Trigger

Review if legal invoice numbering or multi-store numbering introduces stricter requirements.