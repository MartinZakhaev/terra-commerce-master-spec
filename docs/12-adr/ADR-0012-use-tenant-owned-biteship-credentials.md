# ADR-0012 — Use Tenant-Owned Biteship Credentials

**Status:** Final  
**Date:** 2026-07-01  
**Decision Owners:** Product Owner, Integration Lead, Security Owner  
**Related:** FSD v1.0.1, Biteship Webhook Contract v1.0.0, Shipment State Machine v1.0.0

## Context

Biteship account credentials authorize shipment creation and can affect wallet, billing, and operational responsibility. A platform-wide credential would concentrate cost and security risk.

## Decision

Each tenant owns and supplies its own Biteship API credentials for each environment.

Terra Commerce:

- stores credentials encrypted and versioned;
- masks credentials after entry;
- validates credentials during configuration;
- resolves credentials only from trusted tenant context;
- supports overlap during rotation and then revokes the old version;
- never logs raw credentials;
- keeps shipment and webhook reconciliation tenant scoped.

Webhook authenticity and exact payload mapping must follow current official Biteship documentation at implementation time. When a notification cannot independently establish trusted final state, Terra Commerce verifies status through a server-to-server Biteship lookup before a sensitive terminal transition.

## Alternatives Considered

- One platform-owned credential: rejected because billing, wallet, suspension, and compromise would affect every tenant.
- Manual shipment entry: rejected because it breaks the required automated delivery flow.

## Consequences

Positive: tenant-isolated billing and blast radius, simpler settlement responsibility, and independent credential rotation.

Negative: tenant onboarding requires Biteship setup and support must handle tenant-specific credential failures.

## Review Trigger

Review if Biteship provides a formal multi-merchant platform program or Terra Commerce assumes centralized shipping billing.