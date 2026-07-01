# Environment and Configuration Specification — Terra Commerce

**Document ID:** INF-TC-001  
**Version:** 1.0.0  
**Status:** Draft  
**Owner:** Platform and DevOps Lead  
**Last Updated:** 2026-07-01  
**Upstream:** Final Architecture, Infrastructure TDD, Security Threat Model, Quality Strategy, and Release Readiness Checklist

## 1. Purpose

Defines environment classes, configuration ownership, variable naming, secret handling, validation, compatibility, promotion, drift control, and operational change procedures.

## 2. Environment Classes

- Local development.
- CI ephemeral.
- Integration/staging.
- Production.
- Isolated restore/recovery environment.
- Target-equivalent performance environment.

Production data may enter lower environments only after approved sanitization.

## 3. Configuration Principles

1. Configuration is explicit, versioned, validated, and environment specific.
2. Secrets are external to source control and images.
3. Secure defaults are mandatory.
4. Missing critical configuration fails startup.
5. Application configuration and business settings remain separate.
6. Every change has owner, audit trail, rollback, and compatibility assessment.

## 4. Configuration Categories

### Runtime

- Service ports and URLs.
- Timeouts and shutdown deadlines.
- Trusted proxies and CORS origins.
- Body and upload limits.
- Worker concurrency.
- SSE heartbeat, replay, and buffer limits.

### Data

- PostgreSQL connection and pool limits.
- Redis connection, memory, stream, and TTL policy.
- Object-storage endpoint, bucket, and region.

### Provider

- Biteship credentials and verification material.
- Email provider configuration.
- Future payment provider configuration.

### Security

- Authentication/session configuration.
- Internal service trust keys or mTLS material.
- Encryption/rotation settings.
- Rate-limit profiles.

### Observability

- Log level and format.
- Metrics endpoint/exporter.
- Trace sampling.
- Alert destination references.

## 5. Naming Convention

Environment variables use uppercase prefixes:

- `TC_GATEWAY_*`
- `TC_COMMERCE_*`
- `TC_WORKER_*`
- `TC_POSTGRES_*`
- `TC_REDIS_*`
- `TC_SSE_*`
- `TC_BITESHIP_*`
- `TC_EMAIL_*`
- `TC_STORAGE_*`
- `TC_SECURITY_*`
- `TC_OBSERVABILITY_*`

Names are stable contracts. Renaming requires deprecation and migration guidance.

## 6. Required Configuration Registry

Each setting records:

- key
- owning service
- type
- required/optional
- secret classification
- default
- allowed range/format
- environment applicability
- reload behavior
- restart requirement
- owner
- deprecation status

A machine-readable schema must validate environment configuration in CI and at startup.

## 7. Secret Handling

- Never commit secrets.
- Separate credentials per service and environment.
- Least-privilege database, Redis, provider, and deployment identities.
- Mask secrets in UI and logs.
- Rotate after suspected compromise and on defined schedule.
- Do not expose secret values after storage.
- Lower environments use distinct non-production credentials.

## 8. Business Settings vs Runtime Configuration

Business-managed settings such as feature flags, maintenance mode, plan limits, store branding, and tenant warehouse settings live in approved application data stores and authorization workflows.

Runtime configuration such as database URLs, provider secrets, memory limits, and internal trust keys remains deployment-managed.

## 9. Compatibility and Startup Validation

Services validate:

- required variables present
- URL/port/type formats
- secure protocol requirements
- schema-version compatibility
- provider credentials where mandatory
- internal trust configuration
- mutually consistent limits

Services refuse startup when configuration is invalid or insecure.

## 10. Change Management

Every production change includes:

- change ticket or release reference
- owner and reviewer
- before/after values excluding secret contents
- risk and compatibility assessment
- restart/reload impact
- rollback procedure
- validation evidence

Emergency changes are retrospectively reviewed.

## 11. Environment Promotion

Configuration is promoted through templates and reviewed overlays, not copied manually from production. Environment-specific values remain explicit. Promotion verifies schema compatibility and secret availability.

## 12. Drift Detection

- Compare deployed configuration fingerprints against approved release manifest.
- Exclude raw secret values while hashing stable references/versions.
- Alert on unauthorized drift.
- Reconcile through deployment tooling rather than manual mutation where possible.

## 13. Configuration Backup and Recovery

- Version non-secret templates in infrastructure repository.
- Back up application-managed global settings with PostgreSQL.
- Store secret recovery/rotation procedures separately.
- Restore validation confirms environment can start with approved configuration references.

## 14. Validation and Tests

- Missing required variable.
- Invalid type/range.
- insecure URL/protocol.
- mismatched schema range.
- stale/rotated provider credential.
- secret redaction.
- environment isolation.
- configuration drift.
- rollback to previous approved configuration.

## 15. Acceptance Criteria

1. Every configuration key has owner, type, sensitivity, and validation.
2. Secrets remain outside source and images.
3. Startup fails safely on invalid configuration.
4. Environment promotion and drift control are defined.
5. Changes are auditable and reversible.

## Change Summary

Initial environment and configuration specification draft synchronized with finalized infrastructure, security, and quality documents.
