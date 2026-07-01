# Architecture Decision Records

This directory contains final architecture and implementation decisions for Terra Commerce. ADRs are immutable historical records; later changes must create a new ADR that supersedes an earlier decision.

## Final baseline ADRs

1. [`ADR-0001-use-go-gateway.md`](ADR-0001-use-go-gateway.md) — Go owns public API ingress and SSE.
2. [`ADR-0002-use-medusa-v2.md`](ADR-0002-use-medusa-v2.md) — Medusa v2 is the commerce engine.
3. [`ADR-0003-use-shared-schema-multitenancy.md`](ADR-0003-use-shared-schema-multitenancy.md) — shared-database shared-schema tenancy.
4. [`ADR-0004-use-redis-streams.md`](ADR-0004-use-redis-streams.md) — Redis Streams is internal event transport.
5. [`ADR-0005-use-server-sent-events.md`](ADR-0005-use-server-sent-events.md) — SSE is real-time UI transport.
6. [`ADR-0006-use-git-submodules.md`](ADR-0006-use-git-submodules.md) — application repositories are pinned submodules.
7. [`ADR-0007-use-modular-monolith.md`](ADR-0007-use-modular-monolith.md) — MVP uses a modular monolith.
8. [`ADR-0008-use-nuxt-4.md`](ADR-0008-use-nuxt-4.md) — Nuxt 4, Vue 3, and TypeScript for all web applications.
9. [`ADR-0009-use-zitadel-and-gateway-sessions.md`](ADR-0009-use-zitadel-and-gateway-sessions.md) — hosted ZITADEL and gateway-owned browser sessions.
10. [`ADR-0010-use-child-subdomains-for-tenant-routing.md`](ADR-0010-use-child-subdomains-for-tenant-routing.md) — child subdomains resolve tenant context.
11. [`ADR-0011-use-midtrans-snap-with-tenant-credentials.md`](ADR-0011-use-midtrans-snap-with-tenant-credentials.md) — Midtrans Snap with tenant-owned merchant credentials.
12. [`ADR-0012-use-tenant-owned-biteship-credentials.md`](ADR-0012-use-tenant-owned-biteship-credentials.md) — Biteship credentials are tenant owned.
13. [`ADR-0013-use-cloudflare-r2.md`](ADR-0013-use-cloudflare-r2.md) — Cloudflare R2 stores public and private objects.
14. [`ADR-0014-use-resend.md`](ADR-0014-use-resend.md) — Resend provides transactional email.
15. [`ADR-0015-use-30-minute-reservation-expiry.md`](ADR-0015-use-30-minute-reservation-expiry.md) — inventory reservations expire after 30 minutes plus reconciliation grace.
16. [`ADR-0016-use-tenant-date-sequence-order-numbers.md`](ADR-0016-use-tenant-date-sequence-order-numbers.md) — tenant/date/sequence order numbers.
17. [`ADR-0017-use-tax-inclusive-configurable-tax.md`](ADR-0017-use-tax-inclusive-configurable-tax.md) — tax-inclusive pricing with configurable tax snapshots.
18. [`ADR-0018-use-configurable-manual-return-refund-policy.md`](ADR-0018-use-configurable-manual-return-refund-policy.md) — configurable manually reviewed returns and refunds.
19. [`ADR-0019-use-manual-plan-lifecycle.md`](ADR-0019-use-manual-plan-lifecycle.md) — tenant plan lifecycle remains manually administered for MVP.
20. [`ADR-0020-use-tenant-scoped-customer-identity.md`](ADR-0020-use-tenant-scoped-customer-identity.md) — customer identity and email uniqueness are tenant scoped.
21. [`ADR-0021-use-official-medusa-extension-points.md`](ADR-0021-use-official-medusa-extension-points.md) — only supported Medusa extension mechanisms are used.
22. [`ADR-0022-use-postgresql-check-constraints-and-defer-rls-partitioning.md`](ADR-0022-use-postgresql-check-constraints-and-defer-rls-partitioning.md) — text/check states; RLS and initial partitioning deferred.

## Audits

- [`reviews/ADR_Baseline_Audit_v1.0.0.md`](reviews/ADR_Baseline_Audit_v1.0.0.md) — PASS for ADR-0001 through ADR-0007.
- [`reviews/ADR_Decision_Closure_Audit_v1.0.0.md`](reviews/ADR_Decision_Closure_Audit_v1.0.0.md) — PASS for ADR-0008 through ADR-0022; no unresolved MVP decision remains.

## Status

ADR-0001 through ADR-0022 are **Final**. The next ADR sequence is `ADR-0023`.