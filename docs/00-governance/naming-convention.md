# Naming Convention

**Document ID:** GOV-TC-002  
**Version:** 1.0.0  
**Status:** Approved

## 1. File Names

Use descriptive kebab-case for general documents:

- `document-governance.md`
- `tenant-isolation.md`
- `deployment-architecture.md`

Use controlled document names for versioned primary specifications:

- `PRD_Terra_Commerce_v1.0.0_Final.md`
- `FSD_Terra_Commerce_v1.0.0_Draft.md`
- `TDD_Gateway_v1.0.0_Draft.md`

Avoid spaces, ambiguous abbreviations, and suffixes such as `final-final`, `latest`, or `new`.

## 2. Document IDs

Use the following prefixes:

| Artifact | Prefix | Example |
|---|---|---|
| Governance | `GOV-TC` | `GOV-TC-001` |
| PRD | `PRD-TC` | `PRD-TC-001` |
| FSD | `FSD-TC` | `FSD-TC-001` |
| Architecture | `ARCH-TC` | `ARCH-TC-001` |
| Technical design | `TDD-TC` | `TDD-TC-001` |
| Data design | `DATA-TC` | `DATA-TC-001` |
| API contract | `API-TC` | `API-TC-001` |
| Event contract | `EVT-TC` | `EVT-TC-001` |
| Security | `SEC-TC` | `SEC-TC-001` |
| State machine | `SM-TC` | `SM-TC-001` |
| Test specification | `TEST-TC` | `TEST-TC-001` |
| Operations | `OPS-TC` | `OPS-TC-001` |
| ADR | `ADR` | `ADR-0001` |

## 3. Requirement IDs

- Product functional requirement: `PRD-FR-001`
- Product non-functional requirement: `PRD-NFR-001`
- Functional specification requirement: `FSD-FR-001`
- Business rule: `BR-001`
- Acceptance criterion: `AC-001`
- Use case: `UC-001`
- API operation: `API-OP-001`
- Domain event: `EVT-001`
- Permission: use dot notation such as `order.cancel`

IDs are immutable after approval. Deleted requirements remain reserved and are marked deprecated.

## 4. Repository Names

Recommended application repository names:

- `terra-commerce-gateway`
- `terra-commerce-core`
- `terra-commerce-superadmin-web`
- `terra-commerce-tenant-admin-web`
- `terra-commerce-storefront-web`
- `terra-commerce-infrastructure`

## 5. Branch Names

- `feature/<ticket-or-feature>`
- `fix/<ticket-or-issue>`
- `docs/<document-or-topic>`
- `chore/<maintenance-topic>`
- `release/vX.Y.Z`
- `hotfix/<issue>`

Examples:

- `docs/fsd-v1`
- `feature/order-picking`
- `fix/tenant-scope-validation`

## 6. Commit Messages

Use Conventional Commits:

- `docs(prd): finalize Terra Commerce PRD v1.0.0`
- `docs(governance): add dependency map`
- `feat(order): add picking workflow`
- `fix(auth): enforce tenant scope`
- `chore(submodules): update gateway reference`

## 7. API and Event Naming

- REST resources use plural kebab-case nouns.
- JSON fields use `snake_case` or `camelCase` consistently as defined by the API contract; do not mix styles.
- Domain events use past-tense dot notation: `order.created`, `inventory.reserved`, `shipment.delivered`.
- SSE channel names use scoped patterns: `tenant:{tenant_id}:orders`.

## 8. Database Naming

- Tables: plural `snake_case`.
- Columns: `snake_case`.
- Primary key: `id` unless a stronger project-wide convention is approved.
- Foreign keys: `<entity>_id`.
- Tenant-owned tables must include `tenant_id` directly or have a documented enforceable ownership path.
- Indexes: `idx_<table>_<columns>`.
- Unique constraints: `uq_<table>_<columns>`.
- Foreign keys: `fk_<table>_<referenced_table>`.
