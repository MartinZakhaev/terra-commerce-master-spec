# Architecture

This directory contains the approved Terra Commerce system architecture and supporting architecture reviews.

## Current authoritative document

- [`System_Architecture_Terra_Commerce_v1.0.0_Final.md`](System_Architecture_Terra_Commerce_v1.0.0_Final.md) — final system architecture covering runtime containers, module ownership, tenant isolation, request and event flows, Biteship integration, deployment, persistence, backup, security, observability, failure handling, and scaling.

## Historical draft

- [`System_Architecture_Terra_Commerce_v1.0.0_Draft.md`](System_Architecture_Terra_Commerce_v1.0.0_Draft.md) — retained for audit traceability.

## Reviews

- [`reviews/System_Architecture_Audit_v1.0.0.md`](reviews/System_Architecture_Audit_v1.0.0.md) — draft audit with four medium-priority findings.
- [`reviews/System_Architecture_Final_Verification_v1.0.0.md`](reviews/System_Architecture_Final_Verification_v1.0.0.md) — PASS; all findings closed and Final status verified.

## Upstream authorities

- PRD v1.0.0 Final.
- FSD v1.0.1 Draft.
- ADR-0001 through ADR-0007 Final.

## Next downstream work

Create the authoritative state-machine specifications, followed by database design, data dictionary, and cross-application contracts.