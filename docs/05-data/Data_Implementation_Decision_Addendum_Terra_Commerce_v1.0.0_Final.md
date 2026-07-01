# Data Implementation Decision Addendum — Terra Commerce

**Document ID:** DATA-TC-003  
**Version:** 1.0.0  
**Status:** Final  
**Owner:** Technical Lead and Database Lead  
**Last Updated:** 2026-07-01  
**Applies To:** Database Design and Data Dictionary v1.0.0 Final

## 1. Purpose

This addendum closes the implementation decisions listed in Database Design Section 18 and records the required data-model implications. It is authoritative together with the existing final Database Design and Data Dictionary.

## 2. Closed Decisions

| Database open item | Final rule |
|---|---|
| Medusa extension APIs | Official Medusa extension points only; Terra-owned tables use stable references (ADR-0021) |
| Enums versus checks | Text/varchar with synchronized `CHECK` constraints for evolving states (ADR-0022) |
| Row-Level Security | Deferred for MVP; composite tenant-safe FKs and tenant-aware repositories remain mandatory (ADR-0022) |
| Partition thresholds | Review at 10 million rows, 20 GB, or measured maintenance/query pressure (ADR-0022) |
| Authentication/session storage | ZITADEL identity references plus gateway-owned server-side sessions (ADR-0009) |
| Payment provider details | Midtrans Snap with tenant-owned credential records and provider attempt references (ADR-0011) |
| Reservation duration | 30 minutes plus two-minute reconciliation grace (ADR-0015) |
| Order-number algorithm | Tenant code + local date + six-digit tenant/date sequence (ADR-0016) |
| Retention | Protected business/audit history is not automatically deleted; operational retention requires approved policy (ADR-0022) |

## 3. Required Data Additions

### Tenant integration credentials

Create encrypted, versioned integration configuration owned by tenant:

- `tenant_payment_integrations`
- `tenant_delivery_integrations`

Fields include tenant, provider, environment, encrypted credential reference/value, status, version, verification time, rotation time, and audit timestamps. Raw credentials are never returned.

### Identity and sessions

- `users` store external identity reference instead of a required local password hash.
- Customer profiles remain unique by `(tenant_id, email_normalized)`.
- Server-side session records contain opaque session identity, subject, tenant where applicable, expiry, revocation, refresh-credential reference, and security metadata.

### Order numbering

Create a concurrency-safe tenant/date sequence allocator. `order_extensions.order_number` remains immutable and unique by tenant.

### Tax snapshots

Order pricing snapshots include tax label, rate, included amount, rounding result, and tenant configuration version used at checkout.

### Return policy

Tenant/store settings include return window days with a valid range of 0–30 and default 7. Return records preserve the effective policy snapshot.

### Object storage

Object references include tenant, bucket/access class, object key, media type, size, checksum where applicable, privacy class, expiry, and lifecycle status. Object keys use tenant prefixes.

### Reservation expiry

`inventory_reservations.expires_at` is required for payment-backed Active reservations. Expiration, release, consume, and late-payment conflict transitions remain versioned and idempotent.

## 4. Integrity Rules

- Credentials cannot be shared across tenants through normal data relationships.
- Session tenant context must match membership/customer ownership.
- Order-number allocation cannot reuse issued values.
- Tax and policy snapshots are immutable after order creation.
- Private object access always revalidates tenant and resource ownership.
- Reservation release cannot make reserved quantity negative.

## 5. Acceptance Criteria

1. Every Database Design Section 18 item is closed.
2. New identity, integration, numbering, tax, policy, storage, and expiry data is tenant safe.
3. Existing database design constraints remain authoritative.
4. Migrations derived from this addendum follow the final Infrastructure TDD and migration runbook.

## Change Summary

Initial final addendum closes the remaining data implementation decisions using ADR-0009, ADR-0011, ADR-0015, ADR-0016, ADR-0021, and ADR-0022.