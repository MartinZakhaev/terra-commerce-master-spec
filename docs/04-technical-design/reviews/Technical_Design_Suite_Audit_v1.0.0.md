# Technical Design Suite Audit — Terra Commerce

**Document ID:** REVIEW-TC-014  
**Version:** 1.0.0  
**Status:** Completed  
**Audit Date:** 2026-06-30  
**Reviewed:** Go Gateway, Medusa/Terra Commerce Modules, Worker/Event Pipeline, and Infrastructure Deployment TDD drafts v1.0.0  
**Upstream:** All Final architecture, state, data, contract, ADR, and security documents

## 1. Executive Summary

The four TDD drafts are materially aligned with the finalized upstream specifications. They define clear ownership for public ingress, commerce workflows, asynchronous processing, and deployment operations while preserving shared-schema tenancy, durable events, state-machine enforcement, and the 2 vCPU / 4 GB RAM constraint.

No critical or high contradiction was found. Six medium-priority corrections are required before finalization.

**Draft verdict:** Pass with required corrections.

## 2. Findings

### TDD-M01 — Gateway-to-Commerce Authentication Needs a Concrete Trust Contract

The gateway draft requires trusted internal context but does not define replay protection and request integrity.

**Required correction:** Internal calls must use a service identity plus signed request context or mutually authenticated transport, timestamp/nonce or equivalent replay control, audience restriction, deadline, and tenant-context integrity validation. Commerce must independently reject untrusted tenant headers.

### TDD-M02 — Checkout Commit and Payment Initialization Boundary Needs One Normative Pattern

The commerce draft allows multiple possible payment initialization timings, which could lead to inconsistent implementations.

**Required correction:** Define the MVP default: create order/reservation/payment-pending state and outbox atomically, commit, then initialize external payment idempotently. On provider success/failure, transition payment/order through a second transaction. A provider that requires pre-creation must receive a dedicated future design.

### TDD-M03 — Outbox Publisher Publish/Acknowledge Gap Needs Consumer and Reconciliation Detail

The worker draft recognizes duplicate delivery after a crash but needs a concrete reconciliation rule.

**Required correction:** Redis messages must retain the stable outbox/event ID, published outbox rows remain queryable, a periodic reconciler detects long-publishing/stale rows, and consumers deduplicate against durable consumer receipts. Outbox status must never be treated as proof that every consumer completed.

### TDD-M04 — Report Authorization Snapshot Could Become Stale

The worker draft validates permission when a report begins, but access may be revoked before file download.

**Required correction:** Report generation stores requester and permission context for audit, while download always revalidates current tenant membership, permission, and object ownership. Revocation prevents future download even when generation completed.

### TDD-M05 — Deployment Resource Budget Exceeds Safe 4 GB Envelope at Upper Bounds

The infrastructure draft's maximum component guidance can exceed available memory before kernel and filesystem cache needs.

**Required correction:** Define a conservative initial hard-limit profile whose total remains below the host envelope, add swap policy as emergency protection rather than capacity, and define load-test promotion criteria before increasing limits.

### TDD-M06 — Database Migration Failure and Mixed-Version Compatibility Need Explicit Gates

The infrastructure draft mentions compatibility but does not define deployment blocking checks.

**Required correction:** Every release must declare minimum/maximum compatible schema version per service, migration status must gate readiness, destructive migrations require a completed expand-contract sequence, and worker/gateway/commerce startup must refuse an unsupported schema.

## 3. Synchronization Matrix

| Area | Result |
|---|---|
| Gateway public ownership | Pass |
| Tenant and authorization propagation | Pass with TDD-M01 |
| OpenAPI/idempotency/concurrency | Pass |
| Commerce module ownership | Pass |
| Checkout/payment consistency | Pass with TDD-M02 |
| Inventory/WMS state machines | Pass |
| Durable outbox and events | Pass with TDD-M03 |
| SSE projection isolation | Pass |
| Reporting/privacy | Pass with TDD-M04 |
| Biteship integration | Pass |
| Deployment/network isolation | Pass |
| 2 vCPU / 4 GB resource target | Pass with TDD-M05 |
| Migration/release compatibility | Pass with TDD-M06 |
| Backup/recovery/security | Pass |

## 4. Finalization Gate

The suite may become Final after TDD-M01 through TDD-M06 are incorporated consistently into the affected documents.

## 5. Audit Result

- Critical findings: **0**
- High findings: **0**
- Medium findings: **6**
- Material contradictions: **0**
- Missing technical domains: **0**

**Required action:** publish corrected Final versions and perform final suite verification.
