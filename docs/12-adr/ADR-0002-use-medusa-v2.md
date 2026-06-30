# ADR-0002 — Use Medusa.js v2 as the Commerce Engine

**Status:** Final  
**Date:** 2026-06-30  
**Decision Owners:** Product Owner, Technical Lead  
**Related:** PRD v1.0.0, FSD v1.0.1

## Context

Terra Commerce requires product, variant, pricing, cart, customer, order, inventory, fulfillment, return, and refund capabilities. Rebuilding these capabilities from scratch would increase delivery time and risk.

## Decision

Use Medusa.js v2 as the core commerce engine.

Medusa owns or provides the primary commerce capabilities for:

- Products and variants.
- Pricing.
- Customers.
- Carts.
- Orders.
- Promotions.
- Inventory.
- Fulfillment.
- Returns.
- Refund workflows.

Terra Commerce-specific behavior is implemented through approved custom modules, workflows, subscribers, and integrations. Tenant context and platform-specific authorization remain mandatory.

## Alternatives Considered

### Build a custom commerce engine
Rejected for MVP because of scope, risk, and time-to-market.

### Use a monolithic hosted commerce platform
Rejected because it would reduce control over multitenancy, OMS, WMS, gateway contracts, and deployment model.

### Use another headless commerce framework
Deferred because Medusa v2 aligns with the selected Node.js commerce runtime and modular extension model.

## Consequences

### Positive

- Faster delivery of mature commerce functions.
- Extensible workflows and modules.
- Reduced custom implementation surface.

### Negative

- Tenant isolation must be carefully added and tested.
- Medusa internal states may differ from Terra Commerce business states.
- Upgrades require compatibility review.

## Constraints

- Medusa is not directly exposed as the public API boundary.
- Terra Commerce business statuses require explicit mapping.
- Customizations must avoid unnecessary forks.
- Medusa data access must enforce tenant scope.
- Provider and workflow extensions require contract tests.

## Review Trigger

Review if required multitenant behavior cannot be implemented safely, if upgrade compatibility becomes unsustainable, or if measured resource use violates the MVP server target.
