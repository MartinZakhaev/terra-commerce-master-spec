# ADR-0015 — Use a 30-Minute Reservation Expiry

**Status:** Final  
**Date:** 2026-07-01  
**Decision Owners:** Product Owner, Commerce Lead, Operations Lead

## Context

Inventory is reserved before payment completes. Abandoned checkout sessions must not hold stock indefinitely.

## Decision

Inventory reservations for unpaid orders expire 30 minutes after local order creation, followed by a two-minute reconciliation grace period.

At expiry, the worker locks the order, payment, and reservation records, checks current payment status, and either keeps or consumes the reservation for a confirmed payment or releases it idempotently for an unpaid, failed, cancelled, or expired payment.

Payment callbacks and the expiration worker use version checks so only one transition succeeds. A late confirmed payment after stock release blocks fulfillment until stock is safely reserved again or an authorized refund is completed.

The Midtrans payment expiry should match the reservation window where supported.

## Consequences

This returns abandoned stock automatically while preserving a reasonable payment window. It also requires status reconciliation and a late-payment conflict path.

## Review Trigger

Review after conversion and payment-channel data shows a different duration is preferable.