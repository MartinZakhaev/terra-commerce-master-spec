# Functional Specifications

This directory contains the detailed functional behavior derived from the approved Terra Commerce PRD.

## Current document

- [`fsd/FSD_Terra_Commerce_v1.0.0_Draft.md`](fsd/FSD_Terra_Commerce_v1.0.0_Draft.md) — initial functional specification draft covering tenant lifecycle, administration, catalog, storefront, checkout, OMS, inventory, WMS, Biteship, SSE, notifications, reporting, audit, security, and failure handling.

## Review status

- [`reviews/FSD_PRD_Consistency_Review_v1.0.0.md`](reviews/FSD_PRD_Consistency_Review_v1.0.0.md) — completed PRD–FSD audit. Result: conditionally aligned; revision required before promotion to In Review.

## Planned supporting structure

- `use-cases/` — individually traceable actor flows.
- `business-rules/` — consolidated business rules and policy decisions.
- `process-flows/` — cross-module workflow descriptions and diagrams.
- `acceptance-criteria/` — requirement-to-verification mapping.

## Upstream authority

- `../01-product/prd/PRD_Terra_Commerce_v1.0.0_Final.md`
- `../00-governance/`

## Next action

Revise the FSD to resolve all high-priority audit findings, then complete acceptance-criteria traceability before moving the document to In Review.