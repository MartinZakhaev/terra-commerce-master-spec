# Data

This directory contains the authoritative Terra Commerce database design, data dictionary, implementation decision addendum, and their audit history.

## Current authoritative documents

- [`Database_Design_Terra_Commerce_v1.0.0_Final.md`](Database_Design_Terra_Commerce_v1.0.0_Final.md) — final PostgreSQL design covering tenant ownership, Medusa extensions, OMS/WMS, inventory, shipments, idempotency, outbox, audit, indexes, concurrency, migrations, and recovery.
- [`Data_Dictionary_Terra_Commerce_v1.0.0_Final.md`](Data_Dictionary_Terra_Commerce_v1.0.0_Final.md) — final field definitions, status domains, ownership paths, integrity rules, and sensitive-data classifications.
- [`Data_Implementation_Decision_Addendum_Terra_Commerce_v1.0.0_Final.md`](Data_Implementation_Decision_Addendum_Terra_Commerce_v1.0.0_Final.md) — final closure of identity, provider, numbering, tax, policy, storage, reservation, Medusa, PostgreSQL, and retention implementation decisions.

## Historical drafts

- [`Database_Design_Terra_Commerce_v1.0.0_Draft.md`](Database_Design_Terra_Commerce_v1.0.0_Draft.md)
- [`Data_Dictionary_Terra_Commerce_v1.0.0_Draft.md`](Data_Dictionary_Terra_Commerce_v1.0.0_Draft.md)

## Reviews

- [`reviews/Database_Design_Data_Dictionary_Audit_v1.0.0.md`](reviews/Database_Design_Data_Dictionary_Audit_v1.0.0.md) — draft audit with six medium-priority findings.
- [`reviews/Database_Design_Data_Dictionary_Final_Verification_v1.0.0.md`](reviews/Database_Design_Data_Dictionary_Final_Verification_v1.0.0.md) — PASS; all original findings closed.
- [`../12-adr/reviews/ADR_Decision_Closure_Audit_v1.0.0.md`](../12-adr/reviews/ADR_Decision_Closure_Audit_v1.0.0.md) — PASS; all remaining implementation decisions closed.

## Downstream authority

These documents jointly govern database migrations, repository implementations, OpenAPI schemas, event payload references, authorization queries, test fixtures, backup validation, and data-retention implementation.

## Status

The database and data-dictionary baseline is Final and its former open implementation register is closed through ADR-0022.