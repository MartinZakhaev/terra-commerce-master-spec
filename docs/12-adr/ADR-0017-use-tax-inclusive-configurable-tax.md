# ADR-0017 — Use Tax-Inclusive Prices with Configurable Display Tax

**Status:** Final  
**Date:** 2026-07-01  
**Decision Owners:** Product Owner, Commerce Lead, Finance Reviewer

## Context

MVP needs deterministic tax snapshots without implementing a jurisdiction engine or making Terra Commerce responsible for deciding each tenant's legal tax obligations.

## Decision

Product prices are tax-inclusive by default.

Each tenant may configure a fixed tax label and rate for calculation and invoice/order breakdown. Terra Commerce snapshots tax label, rate, and amount on the order.

Rules:

- No automatic jurisdiction, nexus, exemption, or legal tax determination in MVP.
- Tenant remains responsible for legal tax configuration and compliance.
- Historical order tax snapshots never change after order creation.
- Tax calculation uses integer minor units and documented rounding.
- A zero-rate configuration is permitted.
- Tax changes affect only future pricing calculations.

## Alternatives Considered

- Tax-exclusive prices: rejected for MVP because customer totals become less predictable and tenant configuration becomes more complex.
- Automated tax engine: deferred due to jurisdiction, compliance, provider, and cost complexity.
- No tax fields: rejected because order snapshots and reporting need an explicit breakdown.

## Consequences

Positive: simple customer pricing, deterministic snapshots, and low implementation complexity.

Negative: tenants with complex tax obligations need manual configuration or a future integration.

## Review Trigger

Review before international commerce, automated invoicing compliance, tax exemptions, or multi-jurisdiction calculation.