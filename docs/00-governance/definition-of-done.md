# Definition of Done

**Document ID:** GOV-TC-007  
**Version:** 1.0.0  
**Status:** Approved

## 1. Purpose

This document defines the minimum completion criteria for specifications, implementation, integrations, releases, and operational readiness.

## 2. Specification Definition of Done

A specification is done when:

- Required metadata and version are present.
- Scope, actors, preconditions, behavior, errors, and acceptance criteria are clear.
- Requirement IDs are stable and traceable.
- Upstream and downstream dependencies are linked.
- Terms and status values are consistent.
- Tenant ownership and isolation are defined.
- Security and authorization implications are documented.
- Data, API, event, state, and operational impacts are addressed where applicable.
- Open decisions are resolved or explicitly deferred.
- Required reviewers approve the document.

## 3. Feature Definition of Done

A feature is done when:

- Approved requirements and acceptance criteria exist.
- Implementation matches approved architecture and contracts.
- Tenant scope is enforced on the server side.
- Authorization is enforced by permission and resource ownership.
- Happy path, validation, failure, and retry behavior are implemented.
- State transitions follow the approved state machine.
- Persistent changes are covered by migration and rollback strategy.
- Idempotency is implemented where duplicate requests or events are possible.
- Audit logging is implemented for sensitive operations.
- Domain and SSE events are published where required.
- Logs, metrics, and error reporting are sufficient for operation.
- Unit, integration, contract, tenant-isolation, and relevant end-to-end tests pass.
- Documentation and contracts are updated.
- No critical or high unaccepted security findings remain.

## 4. API Definition of Done

An API operation is done when:

- It exists in the approved OpenAPI contract.
- Authentication and permission requirements are declared.
- Tenant resolution and ownership checks are defined.
- Request, response, error, pagination, and idempotency behavior are documented.
- State-transition rules are enforced.
- Contract tests and integration tests pass.
- Observability includes correlation ID and appropriate tenant context.

## 5. Event Definition of Done

An event is done when:

- Event name, producer, consumers, trigger, and schema are documented.
- `tenant_id`, event ID, timestamp, and `schema_version` are included where applicable.
- Delivery, ordering, duplication, retry, and failure behavior are defined.
- Sensitive fields are excluded or audience filtered.
- Producer and consumer contract tests pass.
- Replay or deduplication behavior is tested where required.

## 6. Database Change Definition of Done

A database change is done when:

- Database design and data dictionary are updated.
- Tenant ownership is explicit.
- Constraints and indexes are reviewed.
- Forward and rollback migrations are defined where feasible.
- Existing data compatibility is assessed.
- Backup and recovery impact is reviewed.
- Migration tests pass.

## 7. Release Definition of Done

A release is done when:

- All included features meet their definition of done.
- Required acceptance, regression, security, and performance tests pass.
- Known risks and accepted exceptions are documented.
- Database migrations and rollback instructions are verified.
- Monitoring and alerts are configured.
- Operational runbooks are updated.
- Release notes are complete.
- Master-spec documents reference the approved versions.
- All application submodules are pinned to exact release commits.
- Backup and recovery readiness is confirmed.

## 8. Exceptions

Any exception must identify the unmet criterion, risk, owner, approval, mitigation, and deadline. Exceptions must not be implicit.