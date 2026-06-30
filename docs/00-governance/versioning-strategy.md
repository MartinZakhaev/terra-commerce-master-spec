# Versioning Strategy

**Document ID:** GOV-TC-003  
**Version:** 1.0.0  
**Status:** Approved

## 1. Version Format

Authoritative documents use Semantic Versioning:

`MAJOR.MINOR.PATCH`

Example: `1.2.3`.

## 2. Increment Rules

### MAJOR

Increment when a change:

- Changes approved product scope materially.
- Breaks an approved API or event contract.
- Replaces a core architecture decision.
- Changes tenant-isolation or security assumptions.
- Requires coordinated migration across multiple applications.

### MINOR

Increment when a change:

- Adds backward-compatible requirements or capabilities.
- Adds new endpoints or events without breaking existing consumers.
- Adds a new module, role, report, or workflow.
- Clarifies and expands an approved specification materially.

### PATCH

Increment when a change:

- Corrects wording, formatting, links, examples, or typos.
- Clarifies behavior without changing approved intent.
- Fixes an internal inconsistency with no implementation impact.

## 3. Document Status and Version

Draft documents may evolve under the intended version until approved. Once approved, any content change requires a version increment.

Examples:

- `FSD_Terra_Commerce_v1.0.0_Draft.md`
- `FSD_Terra_Commerce_v1.0.0_Final.md`
- `FSD_Terra_Commerce_v1.1.0_Draft.md`

Do not overwrite a historical final version when traceability is required. A new final version should be added and the previous version retained or moved to an archive according to repository policy.

## 4. Contract Versioning

OpenAPI, event, SSE, and webhook contracts must expose an explicit contract version.

Breaking contract changes require:

- A major version increment.
- A migration or compatibility plan.
- Consumer impact review.
- Coordinated release notes.

Event payloads must include `schema_version`.

## 5. Application Versioning

Each application repository may use its own semantic version. The master repository release manifest pins exact commits and records application versions.

A master-spec release is reproducible only when all submodule references point to exact commits.

## 6. Release Tags

Recommended tags:

- Master specification: `spec-v1.0.0`
- Application release: `v1.0.0`

## 7. Changelog

Every approved MINOR or MAJOR document release must record:

- Added requirements.
- Changed requirements.
- Deprecated requirements.
- Removed requirements.
- Migration or implementation impact.

## 8. Compatibility Principle

Backward compatibility is preferred for MVP. Breaking changes require explicit Product Owner and Technical Lead approval and must not be introduced only through code changes.