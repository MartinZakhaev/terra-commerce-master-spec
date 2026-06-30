# Architecture Decision Records

This directory contains final architecture decisions for Terra Commerce. ADRs are immutable historical records; later changes must create new ADRs that supersede earlier decisions.

## Final baseline ADRs

1. [`ADR-0001-use-go-gateway.md`](ADR-0001-use-go-gateway.md) — Go is the public API gateway and SSE owner.
2. [`ADR-0002-use-medusa-v2.md`](ADR-0002-use-medusa-v2.md) — Medusa.js v2 is the core commerce engine.
3. [`ADR-0003-use-shared-schema-multitenancy.md`](ADR-0003-use-shared-schema-multitenancy.md) — MVP uses shared-database shared-schema multitenancy.
4. [`ADR-0004-use-redis-streams.md`](ADR-0004-use-redis-streams.md) — Redis Streams is the internal event transport.
5. [`ADR-0005-use-server-sent-events.md`](ADR-0005-use-server-sent-events.md) — SSE is the real-time UI transport.
6. [`ADR-0006-use-git-submodules.md`](ADR-0006-use-git-submodules.md) — application repositories are pinned as submodules under `apps/`.
7. [`ADR-0007-use-modular-monolith.md`](ADR-0007-use-modular-monolith.md) — MVP uses a modular-monolith architecture.

## Audit

- [`reviews/ADR_Baseline_Audit_v1.0.0.md`](reviews/ADR_Baseline_Audit_v1.0.0.md) — PASS; no material contradiction or missing locked architecture decision.

## Status

ADR-0001 through ADR-0007 are **Final** as of 2026-06-30.

## Next ADR sequence

The next architecture decision must use `ADR-0008`.

Likely upcoming ADR topics include frontend framework, authentication/session strategy, payment provider, tenant URL model, Biteship credential ownership, object storage, email provider, and reservation expiration.
