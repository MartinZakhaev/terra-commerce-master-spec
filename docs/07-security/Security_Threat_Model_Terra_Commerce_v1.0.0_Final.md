# Security Threat Model — Terra Commerce

**Document ID:** SEC-TC-002  
**Version:** 1.0.0  
**Status:** Final  
**Owner:** Security Owner and Technical Lead  
**Last Updated:** 2026-06-30  
**Upstream:** PRD v1.0.0 Final, FSD v1.0.1 Draft, System Architecture v1.0.0 Final, State Machine v1.0.0 Final, Database Design/Data Dictionary v1.0.0 Final, Contract Suite v1.0.0 Final, Authorization Matrix v1.0.0 Final  
**Audit:** `reviews/Authorization_Threat_Model_Audit_v1.0.0.md`

## 1. Method and Objectives

This model uses STRIDE and rates threats as Critical, High, Medium, or Low.

Security objectives:

1. Prevent cross-tenant and cross-customer access.
2. Prevent unauthorized financial, inventory, shipment, and lifecycle transitions.
3. Protect credentials, personal data, provider secrets, audit history, and backups.
4. Preserve webhook, event, SSE, and worker integrity.
5. Keep the MVP available under constrained infrastructure.
6. Detect, investigate, contain, recover from, and learn from incidents.

## 2. Assets and Trust Boundaries

Critical assets:

- tenant context and ownership rules
- credentials and sessions
- role/permission assignments
- orders, payment, refunds, inventory, shipments
- provider verification material
- PostgreSQL and backups
- audit, transitions, outbox, and idempotency records

Trust boundaries:

- browser to reverse proxy
- proxy to Go gateway
- gateway to Medusa/platform modules
- applications to PostgreSQL/Redis
- workers to streams/providers
- Biteship to webhook endpoint
- applications to object storage/email
- support mode into tenant context
- CI/CD and operators into production

## 3. Primary Threat Register

### TM-001 Cross-Tenant Object Access — Critical

Controls:

- server-resolved tenant context
- tenant-aware repositories
- composite tenant-safe foreign keys
- downstream ownership checks
- tenant-scoped cache, object, event, job, and SSE keys
- cross-tenant tests for APIs, repositories, workers, and replay

### TM-002 Cross-Customer Access — Critical

Controls:

- customer ownership checks for carts, addresses, orders, payments, shipments, returns, refunds, notifications, and SSE
- audience-scoped replay
- non-enumerable IDs only as defense in depth

### TM-003 Role and Permission Escalation — High

Controls:

- seeded roles for MVP
- assignment ceiling
- global/tenant scope separation
- deny-by-default conditional grants
- authorization-cache invalidation
- session revocation for sensitive changes
- audit and CI-validated operation-permission registry

### TM-004 Unauthorized Financial Action — Critical

Targets: payment confirmation, cancellation approval, refund creation/retry.

Controls:

- explicit permissions
- state and amount guards
- version checks
- idempotency
- audit and transition records
- dual control for large refunds
- support denied by default

### TM-005 Inventory Race and Oversell — High

Controls:

- row locking or atomic updates
- aggregate versioning
- reservation lifecycle
- non-negative constraints
- idempotent reserve/release/consume

### TM-006 Checkout Replay — High

Controls:

- idempotency key bound to request hash
- atomic reservation/order creation
- uniqueness constraints
- rate limits

### TM-007 Forged or Replayed Biteship Webhook — Critical

Controls:

- official authenticity verification
- raw-body verification before trust
- provider-event deduplication and payload hash
- tenant resolution from stored provider references
- monotonic shipment rules
- atomic state, tracking, transition, and outbox writes
- quarantine on conflict

### TM-008 Event Tampering, Duplication, or Ordering Failure — High

Controls:

- durable outbox
- event ID and aggregate version
- consumer idempotency
- version-gap handling
- private Redis
- dead letters and reconciliation

### TM-009 SSE Leakage or Replay Abuse — Critical

Controls:

- authenticated stream
- tenant/customer/permission filtering
- audience-scoped replay IDs
- permission revalidation
- filtered projections only
- bounded replay/buffers
- termination on suspension/revocation

### TM-010 Session Theft or Fixation — High

Required controls:

- TLS
- secure HttpOnly/SameSite cookies if session based
- token rotation/revocation if token based
- login session regeneration
- revocation after reset, deactivation, suspension, or sensitive role change
- MFA for global privileged roles before production

### TM-011 Credential Stuffing — High

Controls:

- rate limits by account/network signals
- generic errors
- adaptive password hashing
- progressive delay or lock policy
- security alerts for privileged accounts

### TM-012 CSRF — High for cookie sessions

Controls:

- SameSite
- CSRF token or origin verification
- approved-origin CORS
- no state-changing GET requests

### TM-013 XSS and Content Injection — High

Controls:

- contextual encoding
- HTML sanitization
- CSP
- safe URL validation
- no raw untrusted HTML by default

### TM-014 Injection — Critical

Controls:

- parameterized queries
- allowlisted sorting/filtering
- no untrusted shell construction
- least-privilege database roles
- security tests for reports and dynamic queries

### TM-015 SSRF — High

Controls:

- no arbitrary URL fetching
- provider allowlists
- block private/link-local/metadata ranges
- redirect and DNS-rebinding validation
- controlled upload workflows

### TM-016 Malicious File Upload — High

Controls:

- size/type allowlists
- content inspection or image re-encoding where applicable
- randomized keys
- private storage by default
- no execution of uploads
- signed private access

### TM-017 Secret Exposure — Critical

Controls:

- secrets outside source control
- masked settings APIs
- log/event/SSE redaction
- CI secret scanning
- service-specific credentials
- documented rotation process

### TM-018 Audit Tampering — High

Controls:

- insert/select-only application role
- no normal update/delete methods
- restricted maintenance access
- off-host backup
- alert on privileged audit maintenance

### TM-019 Support Mode Abuse — Critical

Controls:

- dedicated permission
- tenant and reason
- short expiry and visible banner
- real actor plus effective tenant
- no secret/password access
- separate mutation permission
- dual control for emergency high-impact changes
- MFA and full audit

### TM-020 Denial of Service — High

Targets: login, search, reports, rates, checkout, SSE, webhooks, uploads.

Controls:

- route/account/network rate limits
- request/body/query limits
- async reports
- worker concurrency caps
- SSE limits and bounded buffers
- statement timeouts
- circuit breakers/backpressure
- resource alerts

### TM-021 Redis Failure or Poisoning — High

Controls:

- private access
- authentication/least privilege where supported
- memory limits and safe eviction
- envelope validation
- PostgreSQL authority
- degradation runbook

### TM-022 Database or Backup Exposure — Critical

Controls:

- private network
- least-privilege roles
- encrypted transport where needed
- encrypted off-host backups
- restricted restore access
- backup access audit and restore tests

### TM-023 Supply Chain Compromise — High

Controls:

- lockfiles
- dependency/container scanning
- trusted registries
- minimal images
- protected branches
- reviewed workflows
- pinned submodule commits

### TM-024 CI/CD Credential Abuse — Critical

Controls:

- short-lived credentials where possible
- protected environments and approvals
- no production secrets for untrusted PRs
- least privilege
- deployment audit and rollback

### TM-025 Excessive Personal-Data Retention — High

Controls:

- documented retention schedules
- data minimization
- archival/deletion workflows consistent with legal and historical requirements
- periodic retention jobs
- retention exceptions explicitly approved
- audit of bulk export and deletion actions

### TM-026 Insecure Report or Export Access — High

Controls:

- permission re-evaluation at download time
- tenant and requester scope
- expiring signed links
- export logging
- encryption in transit and at rest
- no predictable object keys

### TM-027 Bulk Enumeration and Privacy Abuse — High

Controls:

- pagination and maximum result limits
- rate limits
- need-to-know field projection
- bulk-export permission separate from normal read
- anomaly alerts for high-volume personal-data access
- support and auditor views minimized

## 4. Controls by Layer

Reverse proxy:

- TLS, request limits, security headers, approved routes, SSE-specific buffering rule.

Gateway:

- authentication, tenant resolution, route permission, rate limiting, validation, correlation IDs, normalized errors.

Domain modules:

- resource ownership, state/version guards, plan/feature checks, transaction integrity, dual-control workflows.

Database:

- composite tenant-safe FKs, constraints, append-only permissions, least privilege, encrypted backups.

Redis/workers:

- private access, tenant-context validation, idempotent consumers, bounded retries, dead letters.

Frontends:

- output encoding, CSP, secure sessions, no secrets, backend remains authoritative.

## 5. Privacy, Retention, and Export Requirements

- Every personal-data class has a retention owner and schedule.
- Generated reports and labels use expiring authorized links.
- Bulk export requires dedicated permission.
- Download authorization is rechecked at access time.
- Export actions record actor, tenant, filter scope, row count or object reference, and timestamp.
- Personal data is minimized in warehouse, support, audit, event, SSE, and report views.
- Retention jobs are observable and failures alert operations.

## 6. Security Logging and Alerts

Alert on:

- repeated authentication failures
- cross-tenant/customer denials
- privileged role or owner changes
- support-session start or mutation
- large refund or dual-control override
- webhook verification/payload conflicts
- repeated transition conflicts
- unusual bulk reads or exports
- secret scanning findings
- audit maintenance access
- backup failure
- resource exhaustion

Logs redact credentials, tokens, secret headers, payment-sensitive data, and unnecessary personal data.

## 7. Incident Response and Rotation Readiness

Before production, Terra Commerce must have:

- named incident owner and escalation path
- severity classification
- evidence-preservation guidance
- credential and API-key rotation playbooks
- Biteship webhook-secret/token rotation procedure
- session/token mass revocation procedure
- database credential rotation
- object-storage and email credential rotation
- customer/tenant communication decision process
- recovery and validation checklist
- post-incident review with tracked corrective actions

Security incidents involving possible secret or session compromise require immediate revocation/rotation rather than waiting for full root-cause analysis.

## 8. Verification Plan

Required before production:

- authorization tests for every OpenAPI operation
- CI validation of operation-permission registry
- cross-tenant and cross-customer IDOR tests
- state/version negative tests
- concurrency and idempotency tests
- webhook forgery/replay tests
- SSE audience/replay tests
- file upload and SSRF tests
- privacy export/retention tests
- rate-limit/resource-exhaustion tests
- dependency/container/secret scans
- backup restore test
- support-mode and dual-control review
- incident tabletop exercise
- independent penetration test or focused external review before broad production use

## 9. Residual Risks

Accepted with monitoring:

- shared schema raises blast radius of scoping defects
- single VPS concentrates availability risk
- authentication details remain dependent on future ADR
- exact Biteship wire verification remains implementation-time provider work
- PostgreSQL RLS is deferred

## 10. Release Gates

Release is blocked when:

- a Critical threat lacks implemented mitigation
- cross-tenant tests fail
- a protected operation lacks authorization mapping
- sensitive data appears in logs, events, SSE, or reports
- backup restore has never passed
- webhook verification is absent
- high-impact action lacks required dual control
- incident and credential-rotation procedures are absent
- critical vulnerabilities remain unaccepted
- production secrets exist in source control

## 11. Acceptance Criteria

1. Assets, actors, and trust boundaries are complete.
2. Critical tenant, financial, webhook, SSE, support, privacy, and infrastructure threats have controls and tests.
3. Authorization rules and threat controls agree.
4. Retention, export, incident, and rotation requirements are explicit.
5. Residual risks and release gates are testable.

## Change Summary

Final version closes SEC-M04 and SEC-M05 by adding privacy retention/export threats and mandatory incident-response, evidence-preservation, session revocation, and credential-rotation readiness.
