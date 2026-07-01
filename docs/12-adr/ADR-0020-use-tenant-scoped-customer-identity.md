# ADR-0020 — Use Tenant-Scoped Customer Identity

**Status:** Final  
**Date:** 2026-07-01  
**Decision Owners:** Product Owner, Security Owner, Commerce Lead

## Context

The same person may shop at multiple independent tenant storefronts. Customer ownership and privacy must remain isolated by tenant.

## Decision

Customer identity is tenant scoped for MVP.

- Normalized email is unique by `(tenant_id, email_normalized)`.
- The same email may register independently in different tenant storefronts.
- Customer sessions are bound to both identity and resolved tenant.
- Password recovery, addresses, carts, orders, payments, shipments, returns, refunds, notifications, and SSE ownership are tenant/customer scoped.
- Tenant staff cannot search or correlate a customer's activity in another tenant.
- ZITADEL identity references may be shared at the provider level, but Terra Commerce customer profiles and authorization remain tenant scoped.

## Alternatives Considered

- One global customer account across tenants: rejected because it creates cross-tenant privacy, consent, support, and ownership complexity.
- Separate identity provider instance per tenant: rejected because it adds excessive administration for MVP.

## Consequences

Positive: strong tenant isolation and straightforward ownership rules.

Negative: customers may complete registration or profile setup independently for each tenant.

## Review Trigger

Review if a platform-wide customer account, loyalty program, or cross-tenant marketplace becomes a product requirement.