# ADR-0021 — Use Official Medusa Extension Points

**Status:** Final  
**Date:** 2026-07-01  
**Decision Owners:** Technical Lead, Commerce Lead, Database Lead

## Context

Terra Commerce extends Medusa v2 with tenant ownership, OMS, WMS, delivery, audit, and event behavior. Direct coupling to undocumented Medusa tables would make upgrades unsafe.

## Decision

Use official Medusa v2 Modules, Workflows, Module Links, APIs, and supported extension mechanisms.

Rules:

- Terra Commerce never writes directly to undocumented Medusa core tables.
- Terra-owned persistence uses separate clearly owned tables and stable Medusa references.
- Cross-module behavior uses explicit workflows or interfaces.
- Pin an exact Medusa v2 version for every release.
- Upgrades require compatibility tests, migration review, and contract regression.
- A temporary adapter may isolate a documented Medusa version-specific API, but the dependency is recorded and tested.
- Unsupported core-table coupling requires a new ADR and is prohibited by default.

## Alternatives Considered

- Direct database modification: rejected because it risks corruption and upgrade breakage.
- Forking Medusa: rejected because maintenance cost is too high for MVP.
- Replacing Medusa with a fully custom commerce core: rejected because it delays delivery.

## Consequences

Positive: safer upgrades, clear ownership, and maintainable module boundaries.

Negative: some features may require additional extension tables or workflow orchestration instead of direct database access.

## Review Trigger

Review when an essential requirement cannot be implemented through supported extension points.