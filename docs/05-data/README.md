# Data

This directory contains the authoritative Terra Commerce database design, data dictionary, and their audit history.

## Current authoritative documents

- [`Database_Design_Terra_Commerce_v1.0.0_Final.md`](Database_Design_Terra_Commerce_v1.0.0_Final.md) — final PostgreSQL design covering tenant ownership, Medusa extensions, OMS/WMS, inventory, shipments, idempotency, outbox, audit, indexes, concurrency, migrations, and recovery.
- [`Data_Dictionary_Terra_Commerce_v1.0.0_Final.md`](Data_Dictionary_Terra_Commerce_v1.0.0_Final.md) — final field definitions, status domains, ownership paths, integrity rules, and sensitive-data classifications.

## Historical drafts

- [`Database_Design_Terra_Commerce_v1.0.0_Draft.md`](Database_Design_Terra_Commerce_v1.0.0_Draft.md)
- [`Data_Dictionary_Terra_Commerce_v1.0.0_Draft.md`](Data_Dictionary_Terra_Commerce_v1.0.0_Draft.md)

## Reviews

- [`reviews/Database_Design_Data_Dictionary_Audit_v1.0.0.md`](reviews/Database_Design_Data_Dictionary_Audit_v1.0.0.md) — draft audit with six medium-priority findings.
- [`reviews/Database_Design_Data_Dictionary_Final_Verification_v1.0.0.md`](reviews/Database_Design_Data_Dictionary_Final_Verification_v1.0.0.md) — PASS; all findings closed.

## Downstream authority

These documents govern database migrations, repository implementations, OpenAPI schemas, event payload references, authorization queries, test fixtures, backup validation, and data-retention implementation.

## Next recommended artifact

Create the OpenAPI, domain-event, SSE, and Biteship webhook contracts using the final architecture, state machines, database design, and data dictionary as authoritative inputs.