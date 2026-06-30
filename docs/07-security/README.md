# Security

This directory contains the authoritative Terra Commerce authorization and threat-model documentation.

## Current authoritative documents

- [`Authorization_Matrix_Terra_Commerce_v1.0.0_Final.md`](Authorization_Matrix_Terra_Commerce_v1.0.0_Final.md) — final role, permission, ownership, state, version, plan, support-mode, and dual-control rules.
- [`Security_Threat_Model_Terra_Commerce_v1.0.0_Final.md`](Security_Threat_Model_Terra_Commerce_v1.0.0_Final.md) — final STRIDE-based threat model covering tenant isolation, financial actions, webhook/SSE security, infrastructure, privacy, incident response, and release gates.

## Historical drafts

- `Authorization_Matrix_Terra_Commerce_v1.0.0_Draft.md`
- `Security_Threat_Model_Terra_Commerce_v1.0.0_Draft.md`

## Reviews

- [`reviews/Authorization_Threat_Model_Audit_v1.0.0.md`](reviews/Authorization_Threat_Model_Audit_v1.0.0.md) — draft audit with five medium-priority findings.
- [`reviews/Authorization_Threat_Model_Final_Verification_v1.0.0.md`](reviews/Authorization_Threat_Model_Final_Verification_v1.0.0.md) — PASS; all findings closed.

## Downstream authority

These documents govern endpoint permission metadata, authorization middleware, resource-ownership checks, support-mode controls, dual-approval workflows, security tests, CI permission-registry validation, privacy controls, incident runbooks, and production release gates.

## Next recommended work

Create the Technical Design Documents for the Go gateway, Medusa/Terra Commerce modules, worker/event pipeline, and infrastructure deployment using all finalized upstream specifications.