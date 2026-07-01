# Environment and Configuration Specification — Terra Commerce

**Document ID:** INF-TC-001  
**Version:** 1.0.0  
**Status:** Final  
**Owner:** Platform and DevOps Lead  
**Last Updated:** 2026-07-01  
**Upstream:** Final Architecture, Infrastructure TDD, Security Threat Model, Quality Strategy, Release Readiness Checklist  
**Audit:** `reviews/Infrastructure_Operations_Suite_Audit_v1.0.0.md`

## 1. Purpose

Defines environment classes, configuration ownership, source of truth, validation, secret handling, mutability, rotation, promotion, compatibility, drift control, recovery, and operational change procedures.

## 2. Environment Classes

- Local development.
- CI ephemeral.
- Integration/staging.
- Production.
- Isolated restore/recovery.
- Target-equivalent performance environment.

Production data may enter lower environments only after approved sanitization.

## 3. Configuration Principles

1. Configuration is explicit, versioned, validated, and environment specific.
2. Secrets remain outside source control and images.
3. Secure defaults are mandatory.
4. Missing critical configuration fails startup.
5. Runtime configuration and business-managed settings remain separate.
6. Every change has owner, review, rollback, compatibility, and audit evidence.

## 4. Configuration Categories

- Runtime: ports, URLs, timeouts, trusted proxies, CORS, body limits, worker/SSE limits.
- Data: PostgreSQL, Redis, object storage.
- Provider: Biteship, email, future payment provider.
- Security: authentication/session, service trust, encryption, rate limits.
- Observability: logs, metrics, tracing, alert integrations.

## 5. Naming

Use stable uppercase prefixes: `TC_GATEWAY_*`, `TC_COMMERCE_*`, `TC_WORKER_*`, `TC_POSTGRES_*`, `TC_REDIS_*`, `TC_SSE_*`, `TC_BITESHIP_*`, `TC_EMAIL_*`, `TC_STORAGE_*`, `TC_SECURITY_*`, and `TC_OBSERVABILITY_*`.

Renaming requires deprecation and migration guidance.

## 6. Machine-Readable Configuration Registry

Every key records:

- key and owning service
- source of truth
- type and allowed range/format
- required/optional
- secret classification
- default
- environment applicability
- mutable operationally or immutable per release
- reload/restart requirement
- dependent services and update order
- rotation frequency where applicable
- current secret/config version or reference
- last rotation/change timestamp
- owner and reviewer
- deprecation status

CI and startup validate against the registry schema.

## 7. Secret Handling and Rotation

- Separate credentials per service/environment.
- Least-privilege database, Redis, provider, storage, and deployment identities.
- Mask secrets in logs/UI and never return stored values.
- Rotate on schedule and immediately after suspected compromise.
- Record secret version/reference, not raw value.
- Rotation plans define dependency order, overlap period, validation, old-secret revocation, and rollback.
- Lower environments use distinct non-production credentials.

## 8. Runtime vs Business Settings

Feature flags, maintenance mode, plan limits, branding, and tenant warehouse settings are application-managed and authorized/audited.

Database URLs, provider secrets, service trust keys, memory limits, and transport settings are deployment-managed.

## 9. Startup and Compatibility Validation

Services validate required variables, formats, secure protocols, schema compatibility, provider prerequisites, internal trust configuration, and mutually consistent limits. Invalid or insecure configuration blocks startup/readiness.

## 10. Change Management

Production changes require owner, reviewer, release/change reference, non-secret before/after record, risk, compatibility, restart impact, rollback, and validation evidence. Emergency changes receive retrospective review.

## 11. Promotion and Drift

Promote reviewed templates/overlays, never manual production copies. Compare deployed configuration fingerprints and secret/config versions against the release manifest. Unauthorized drift alerts operations and is reconciled through deployment tooling.

## 12. Recovery

Version non-secret templates in infrastructure code. PostgreSQL backups cover application-managed settings. Secret restoration uses rotation/recovery procedures. Restore validation confirms approved references, versions, and startup compatibility.

## 13. Required Tests

Missing/invalid configuration, insecure protocol, schema mismatch, stale credential, secret redaction, environment isolation, rotation, dependency ordering, drift, rollback, and startup refusal.

## 14. Acceptance Criteria

1. Every key has owner, source, mutability, sensitivity, validation, and lifecycle metadata.
2. Secret rotation order and versioning are explicit.
3. Invalid configuration fails safely.
4. Promotion and drift are controlled.
5. Changes remain auditable and reversible.

## Change Summary

Final version closes OPS-M01 by adding configuration source, mutability, rotation/version metadata, dependency order, and last-change tracking.
