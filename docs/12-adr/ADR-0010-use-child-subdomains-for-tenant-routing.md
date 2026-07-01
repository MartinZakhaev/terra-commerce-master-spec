# ADR-0010 — Use Child Subdomains for Tenant Routing

**Status:** Final  
**Date:** 2026-07-01  
**Decision Owners:** Product Owner, Gateway Lead, Frontend Lead  
**Related:** FSD v1.0.1, System Architecture v1.0.0, Go Gateway TDD v1.0.0

## Context

The gateway needs a stable, secure, and simple tenant-resolution model for storefront and tenant administration while keeping custom-domain automation outside MVP.

## Decision

Use child subdomains with separate namespaces:

- Storefront: `{tenant-slug}.shop.<platform-domain>`.
- Tenant administration: `{tenant-slug}.admin.<platform-domain>`.
- Global administration: `platform.<platform-domain>`.

The Go gateway resolves tenant context from the validated host. Request-body or arbitrary client headers never override host-derived tenant context.

Rules:

- Tenant slug is globally unique and normalized.
- Reserved slugs include platform infrastructure names such as `www`, `api`, `admin`, `platform`, `assets`, `static`, `mail`, and `support`.
- Conflicting authenticated membership and host context are rejected.
- Tenant slug is immutable after activation unless an audited migration procedure is approved.
- Wildcard DNS and TLS certificates are managed through Cloudflare.
- Custom tenant domains are deferred.

## Alternatives Considered

- Path tenancy: simpler DNS, but weaker tenant branding and more routing/caching complexity.
- Shared host with tenant selector: rejected because it increases context-confusion risk.
- Automated custom domains: deferred due to certificate, DNS, verification, and support complexity.

## Consequences

Positive: explicit tenant context, same-origin frontend/API deployment, simple branding, and clear isolation boundaries.

Negative: wildcard DNS/TLS dependency and operational care when changing slugs.

## Review Trigger

Review when custom domains become a product requirement or when multi-region routing is introduced.