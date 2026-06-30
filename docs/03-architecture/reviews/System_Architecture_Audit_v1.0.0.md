# System Architecture Audit

**Document ID:** REVIEW-TC-004  
**Version:** 1.0.0  
**Status:** Completed  
**Audit Date:** 2026-06-30  
**Reviewed Document:** `System_Architecture_Terra_Commerce_v1.0.0_Draft.md`  
**Upstream Sources:** PRD v1.0.0 Final, FSD v1.0.1 Draft, ADR-0001 through ADR-0007

## 1. Executive Summary

The draft is materially aligned with the PRD, FSD, and final ADR baseline. It correctly defines gateway and Medusa ownership, modular-monolith boundaries, shared-schema tenancy, Redis Streams, SSE, Biteship integration, deployment topology, resource constraints, observability, and scaling direction.

The audit identified no critical contradiction, but found four medium-priority completeness gaps that must be resolved before finalization.

**Draft verdict:** Pass with required corrections.

## 2. Findings

### ARCH-M01 — Backup and Recovery Architecture Is Too Shallow

The draft mentions backups only indirectly. The final document must define:

- PostgreSQL backup ownership.
- Backup storage outside the application VPS.
- Restore testing.
- Recovery point and recovery time targets as later operational values.
- Redis Streams as non-authoritative and therefore not the sole recovery source.

### ARCH-M02 — Persistent Volume and Secret Boundaries Are Not Explicit

The final document must distinguish:

- Persistent PostgreSQL data.
- Redis persistence where enabled.
- Ephemeral application containers.
- External object storage.
- Secret injection through environment or secret-management mechanisms.

### ARCH-M03 — Transactional Event Publication Needs a Stronger Rule

The draft only says technical design must evaluate an outbox. For critical events, architecture must require a durable publication strategy so a committed transaction cannot silently fail to produce its required event.

The exact implementation may be outbox or an equivalent documented mechanism.

### ARCH-M04 — Internal Network and Data Access Rules Need Clarification

The final document must state:

- PostgreSQL and Redis are not publicly exposed.
- Only approved application processes access them.
- Frontends never access Medusa, PostgreSQL, or Redis directly.
- Service credentials are separate and least privilege.

## 3. Confirmed Consistencies

| Area | Result |
|---|---|
| Go gateway ownership | Pass |
| Medusa commerce ownership | Pass |
| Modular monolith | Pass |
| Shared-schema multitenancy | Pass |
| Tenant context propagation | Pass |
| OMS and WMS logical modules | Pass |
| Redis Streams versus SSE | Pass |
| Biteship idempotency | Pass |
| 2 vCPU / 4 GB RAM target | Pass |
| Static frontend preference | Pass |
| Git submodule release model | Pass |
| Scaling path | Pass |

## 4. Finalization Gate

The document may become Final when ARCH-M01 through ARCH-M04 are incorporated.

No new ADR is required because the corrections clarify existing approved decisions rather than introduce a conflicting architecture choice.

## 5. Audit Conclusion

- Critical findings: **0**
- High findings: **0**
- Medium findings: **4**
- Material contradictions: **0**

**Required action:** Publish a corrected Final architecture document including backup/recovery, persistence, durable event publication, and internal network rules.
