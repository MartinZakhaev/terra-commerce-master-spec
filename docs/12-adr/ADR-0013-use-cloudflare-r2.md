# ADR-0013 — Use Cloudflare R2 for Object Storage

**Status:** Final  
**Date:** 2026-07-01  
**Decision Owners:** Product Owner, Platform Lead, Security Owner  
**Related:** FSD v1.0.1, Infrastructure TDD v1.0.0, Security Threat Model v1.0.0

## Context

Terra Commerce needs durable object storage for public tenant media and private labels, reports, evidence, and temporary uploads without placing those files on the application VPS.

## Decision

Use platform-owned Cloudflare R2 with separate buckets by environment and access class.

Recommended separation:

- public media bucket per environment;
- private object bucket per environment.

Tenant object keys always begin with `tenants/{tenant_id}/` and continue with a stable resource category and identifier.

Rules:

- Product images and branding may be public through approved delivery paths.
- Shipping labels, reports, return evidence, and temporary uploads remain private.
- Private access uses short-lived authorized presigned URLs.
- Upload URLs are content-type and size constrained where supported.
- Object references, expiry, retention, and authorization are stored in PostgreSQL.
- Credentials are environment-specific and least privilege.
- No business-critical object exists only on local container disk.

## Alternatives Considered

- Local VPS storage: rejected because it weakens durability, scaling, and recovery.
- Tenant-owned buckets: rejected for MVP due to onboarding and policy complexity.
- Other S3-compatible providers: viable, but R2 is selected for Cloudflare integration and predictable egress characteristics.

## Consequences

Positive: durable external storage, S3-compatible APIs, separation from VPS disk, and straightforward signed access.

Negative: external provider dependency and the need to govern bucket lifecycle, CORS, and object retention carefully.

## Review Trigger

Review for regional data-residency requirements, significant cost changes, or tenant-managed storage requirements.