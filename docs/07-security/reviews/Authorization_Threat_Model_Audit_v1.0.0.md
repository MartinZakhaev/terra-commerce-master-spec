# Authorization Matrix and Threat Model Audit

**Document ID:** REVIEW-TC-012  
**Version:** 1.0.0  
**Status:** Completed  
**Audit Date:** 2026-06-30  
**Reviewed:** Authorization Matrix v1.0.0 Draft and Security Threat Model v1.0.0 Draft  
**Upstream:** FSD v1.0.1, System Architecture v1.0.0 Final, State Machine v1.0.0 Final, Database Design/Data Dictionary v1.0.0 Final, Contract Suite v1.0.0 Final

## 1. Executive Summary

The drafts are detailed and materially synchronized with the approved functional, architecture, state, data, and contract documents. They correctly cover tenant isolation, customer self-access, financial actions, WMS, shipment operations, support mode, webhook security, SSE isolation, infrastructure threats, and release gates.

No critical contradiction was found. Five medium-priority corrections are required before finalization.

**Draft verdict:** Pass with required corrections.

## 2. Findings

### SEC-M01 — Default Role Grants Need Explicit Deny-By-Default Semantics

Several matrix cells use `C` without defining the exact permission key or whether the condition is enabled by default.

**Required correction:** State that all conditional grants are denied by default and require an explicit permission assignment or policy condition. List the main conditional permissions for finance, warehouse staff, support, and operations.

### SEC-M02 — Endpoint Coverage Needs a Formal Completeness Rule

The matrix provides examples but does not guarantee that every OpenAPI operation is mapped.

**Required correction:** Require a machine-checkable operation-to-permission registry keyed by OpenAPI `operationId`, with CI failure for missing, duplicate, or unknown permission mappings.

### SEC-M03 — Separation of Duties for Financial and Privileged Actions Is Incomplete

The drafts allow highly privileged users to initiate and approve sensitive actions without a second control.

**Required correction:** Require configurable dual control for large refunds, tenant archival, owner replacement, global secret changes, and emergency support-mode mutations. MVP thresholds may be operational configuration, but the control requirement must exist.

### SEC-M04 — Threat Model Lacks Explicit Data Retention and Privacy-Abuse Threats

The model covers disclosure but does not explicitly cover excessive retention, insecure exports, or bulk enumeration of personal data.

**Required correction:** Add threats and controls for privacy minimization, report/download authorization, expiring links, export logging, retention schedules, and bulk-query limits.

### SEC-M05 — Security Incident and Key-Rotation Readiness Needs Stronger Release Requirements

Secret exposure and operational alerting are covered, but the final model must require incident ownership, credential rotation playbooks, webhook-secret rotation, session revocation, evidence preservation, and post-incident review.

## 3. Synchronization Matrix

| Area | Result |
|---|---|
| OpenAPI roles and audiences | Pass with SEC-M02 |
| State-machine guards | Pass |
| Tenant and customer ownership | Pass |
| Plan and feature checks | Pass |
| Financial actions | Pass with SEC-M03 |
| WMS and shipment roles | Pass with SEC-M01 |
| Support mode | Pass with SEC-M03 |
| SSE and event audiences | Pass |
| Biteship webhook security | Pass |
| Data model controls | Pass |
| Privacy and retention | Pass with SEC-M04 |
| Incident response | Pass with SEC-M05 |

## 4. Finalization Gate

The documents may become Final after SEC-M01 through SEC-M05 are incorporated consistently.

## 5. Audit Result

- Critical findings: **0**
- High findings: **0**
- Medium findings: **5**
- Material contradictions: **0**
- Missing major trust boundaries: **0**

**Required action:** publish corrected Final versions and run final verification.
