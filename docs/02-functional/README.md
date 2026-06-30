# Functional Specifications

This directory contains the detailed functional behavior derived from the approved Terra Commerce PRD.

## Current document

- [`fsd/FSD_Terra_Commerce_v1.0.0_Draft.md`](fsd/FSD_Terra_Commerce_v1.0.0_Draft.md) — initial functional specification draft covering tenant lifecycle, administration, catalog, storefront, checkout, OMS, inventory, WMS, Biteship, SSE, notifications, reporting, audit, security, and failure handling.

## Planned supporting structure

- `use-cases/` — individually traceable actor flows.
- `business-rules/` — consolidated business rules and policy decisions.
- `process-flows/` — cross-module workflow descriptions and diagrams.
- `acceptance-criteria/` — requirement-to-verification mapping.

## Upstream authority

- `../01-product/prd/PRD_Terra_Commerce_v1.0.0_Final.md`
- `../00-governance/`

## Next review action

Review the FSD draft, resolve its open decisions through ADRs, and promote it to In Review before architecture and contract finalization.