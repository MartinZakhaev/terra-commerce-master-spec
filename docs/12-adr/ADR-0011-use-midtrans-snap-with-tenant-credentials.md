# ADR-0011 — Use Midtrans Snap with Tenant-Owned Credentials

**Status:** Final  
**Date:** 2026-07-01  
**Decision Owners:** Product Owner, Commerce Lead, Security Owner  
**Related:** FSD v1.0.1, Payment State Machine v1.0.0, Commerce TDD v1.0.0

## Context

The MVP needs an Indonesian payment provider that minimizes custom payment UI and supports recoverable, idempotent payment processing.

## Decision

Use Midtrans Snap for customer payments.

Each tenant owns and supplies its own Midtrans merchant credentials. Terra Commerce stores credentials encrypted, versions them, masks them after entry, and resolves them only from trusted tenant context.

Processing rules:

1. Commit local order, inventory reservation, and pending payment state first.
2. Create the Snap transaction after commit using a stable tenant-scoped order/payment reference.
3. Use provider idempotency and local idempotency records.
4. Verify notifications using the officially supported Midtrans mechanism and, when needed, server-to-server status lookup.
5. Persist provider outcomes through a separate versioned transaction.
6. Treat timeouts as unknown outcomes until reconciled.
7. Never allow refunded amount to exceed captured amount.

Settlement occurs directly to each tenant's Midtrans merchant account. Terra Commerce is not the merchant of record for MVP.

## Alternatives Considered

- Midtrans Core API: rejected for MVP because it requires more payment UI and channel-specific implementation.
- Platform-owned Midtrans account: rejected because it centralizes settlement, compliance, credential blast radius, and reconciliation.
- Manual payment only: insufficient for the intended storefront flow.

## Consequences

Positive: faster implementation, familiar Indonesian payment methods, tenant-isolated settlement, and smaller platform credential blast radius.

Negative: each tenant must provision a Midtrans merchant account and credential support becomes part of onboarding.

## Review Trigger

Review if Terra Commerce becomes merchant of record, adds subscription billing, or requires payment flows unsupported by Snap.