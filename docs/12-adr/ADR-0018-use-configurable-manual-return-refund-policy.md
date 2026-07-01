# ADR-0018 — Use a Configurable Manual Return and Refund Policy

**Status:** Final  
**Date:** 2026-07-01  
**Decision Owners:** Product Owner, Order Management Lead, Finance Reviewer

## Context

The MVP needs predictable return eligibility and refund behavior without automating every merchant policy or carrier return flow.

## Decision

Use a tenant-configurable return-request window with a default of seven calendar days after delivery and an allowed MVP range of zero to 30 days.

Rules:

- Returns require manual review and approval.
- Requested quantity cannot exceed delivered and not previously returned quantity.
- Refund normally starts after receipt and inspection of approved return items.
- Partial returns and partial refunds are supported.
- Refunded total cannot exceed captured amount.
- Shipping fee is non-refundable by default but may be included through an authorized manual decision.
- Return evidence is private and follows storage and privacy controls.
- No automatic return-shipping label is provided in MVP.
- Every approval, rejection, inspection, disposition, and refund action is state guarded and audited.
- Large refunds follow the configured dual-control threshold.

## Alternatives Considered

- One fixed platform-wide policy: rejected because tenant businesses differ.
- Fully automatic refunds on request: rejected due to fraud and inventory risk.
- No return workflow: rejected because OMS requirements include returns and refunds.

## Consequences

Positive: flexible tenant policy with controlled financial and inventory effects.

Negative: manual operational workload and the need for clear tenant-facing policy content.

## Review Trigger

Review when automated return labels, exchange workflows, or category-specific rules become required.