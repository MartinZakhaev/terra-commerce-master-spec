# State Management

This directory contains the authoritative lifecycle and transition rules for Terra Commerce.

## Current authoritative document

- [`State_Machine_Terra_Commerce_v1.0.0_Final.md`](State_Machine_Terra_Commerce_v1.0.0_Final.md) — final state-machine specification for tenant, order, payment, inventory reservation, fulfillment, picking, packing, shipment, return, refund, and background-job lifecycles.

## Historical draft

- [`State_Machine_Terra_Commerce_v1.0.0_Draft.md`](State_Machine_Terra_Commerce_v1.0.0_Draft.md) — retained for audit traceability.

## Reviews

- [`reviews/State_Machine_Audit_v1.0.0.md`](reviews/State_Machine_Audit_v1.0.0.md) — draft audit with four medium-priority findings.
- [`reviews/State_Machine_Final_Verification_v1.0.0.md`](reviews/State_Machine_Final_Verification_v1.0.0.md) — PASS; all findings closed.

## Downstream authority

This specification governs database status fields and constraints, OpenAPI commands, domain events, authorization rules, UI actions, reconciliation tools, and state-transition tests.

## Next recommended artifact

Create the Database Design and Data Dictionary using the final architecture and state-machine specification as authoritative inputs.