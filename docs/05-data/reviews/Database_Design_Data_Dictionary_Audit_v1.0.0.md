# Database Design and Data Dictionary Audit

**Document ID:** REVIEW-TC-008  
**Version:** 1.0.0  
**Status:** Completed  
**Audit Date:** 2026-06-30  
**Reviewed:** `Database_Design_Terra_Commerce_v1.0.0_Draft.md`, `Data_Dictionary_Terra_Commerce_v1.0.0_Draft.md`  
**Upstream:** PRD v1.0.0 Final, FSD v1.0.1 Draft, System Architecture v1.0.0 Final, State Machine v1.0.0 Final

## 1. Executive Summary

The drafts provide strong coverage of tenant ownership, platform entities, Medusa extension boundaries, OMS/WMS, inventory, shipment, return/refund, audit, idempotency, durable outbox, indexing, concurrency, migration, and recovery.

No critical contradiction was found. Six medium-priority corrections are required before finalization.

**Draft verdict:** Pass with required corrections.

## 2. Findings

### DATA-M01 — Same-Tenant Foreign-Key Enforcement Is Not Concrete Enough

The drafts require tenant-aware application checks, but several tables can technically reference a parent belonging to another tenant when only independent UUID foreign keys are used.

**Required correction:** Define composite unique keys such as `(tenant_id, id)` on tenant-owned parent tables and composite foreign keys from tenant-owned children wherever practical. At minimum, order, warehouse, inventory item, reservation, tasks, shipment, return, refund, notification, audit, and outbox relationships must be tenant-safe.

### DATA-M02 — Payment and Fulfillment State Persistence Is Ambiguous

`order_extensions` contains payment and fulfillment status projections, while Medusa owns payment and fulfillment records. The drafts do not define how Terra Commerce stores state-machine versions and transition-safe references for those sub-lifecycles.

**Required correction:** Add `payment_state_extensions` and `fulfillment_state_extensions`, or define an equivalent supported Medusa extension mechanism. `order_extensions` may retain projections, but those projections must not be the only concurrency authority.

### DATA-M03 — Inventory Quantity Semantics Are Ambiguous

The dictionary describes `physical_qty` in a way that could include or exclude damaged stock, while the approved formula is `available = physical - reserved`.

**Required correction:** Define `physical_qty` as saleable on-hand stock excluding damaged stock. `damaged_qty` is physically present but unavailable and tracked separately. Receiving equations and movement dimensions must state exactly which bucket changes.

### DATA-M04 — Idempotency Uniqueness with Nullable Tenant Is Unsafe

A normal PostgreSQL unique constraint permits multiple null `tenant_id` values, so global operations could reuse the same scope and key.

**Required correction:** Use separate partial unique indexes for tenant and global records, or store a non-null scope-owner key.

### DATA-M05 — Outbox Ordering and Duplicate Publication Need Stronger Keys

The outbox lacks aggregate version or event sequence uniqueness. Multiple retries or transaction bugs could create duplicate logical events.

**Required correction:** Add `aggregate_version`, unique `(aggregate_type, aggregate_id, aggregate_version, event_type)`, claim/lock metadata, and publisher-safe `FOR UPDATE SKIP LOCKED` guidance.

### DATA-M06 — Append-Only and Recipient Constraints Need Database-Level Rules

Audit, transition, movement, and tracking tables are described as append-only, but enforcement is not specified. Notifications require exactly one recipient but no concrete check constraint is given.

**Required correction:** Define database permissions/triggers or repository restrictions preventing normal update/delete on append-only tables, plus an XOR check for notification recipient columns.

## 3. Synchronization Matrix

| Upstream Requirement | Draft Coverage | Result |
|---|---|---|
| Shared-schema tenancy | Broad | Pass with DATA-M01 |
| One tenant/store/warehouse | Unique tenant constraints | Pass |
| Plan management and limits | Plans and usage counters | Pass |
| Product/customer/order integration | Medusa link/extension tables | Pass |
| Final state machines | Status domains and versions | Pass with DATA-M02 |
| Safe inventory reservation | Inventory items/reservations/movements | Pass with DATA-M03 |
| Receiving, picking, packing | Detailed tables | Pass |
| Biteship and tracking | Shipment and webhook tables | Pass |
| Idempotency | Dedicated records and keys | Pass with DATA-M04 |
| Durable event publication | Outbox | Pass with DATA-M05 |
| Audit and transition history | Append-only tables | Pass with DATA-M06 |
| Backup and recovery | PostgreSQL authority and restore | Pass |
| Medusa ownership | Extension references | Pass |

## 4. Finalization Gate

The documents may become Final after DATA-M01 through DATA-M06 are incorporated consistently in both the Database Design and Data Dictionary.

No new ADR is required because these corrections strengthen implementation safety within the approved shared-schema and modular-monolith decisions.

## 5. Audit Result

- Critical findings: **0**
- High findings: **0**
- Medium findings: **6**
- Material contradictions: **0**
- Missing major business domains: **0**

**Required action:** publish corrected Final database design and data dictionary documents.
