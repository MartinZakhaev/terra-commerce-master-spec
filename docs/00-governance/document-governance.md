# Document Governance Policy

**Document ID:** GOV-TC-001  
**Version:** 1.0.0  
**Status:** Approved  
**Applies To:** `terra-commerce-master-spec` and all linked application repositories

## 1. Purpose

This policy defines how Terra Commerce specifications are created, reviewed, approved, versioned, changed, and kept synchronized with application implementation.

## 2. Source of Truth

`MartinZakhaev/terra-commerce-master-spec` is the authoritative source for product, functional, architecture, data, API, event, security, infrastructure, quality, and operational decisions.

Application repositories may contain implementation notes, but they must not redefine or contradict approved master-spec documents.

## 3. Document Hierarchy

The hierarchy is:

1. PRD.
2. FSD and business rules.
3. System architecture and ADRs.
4. Technical design.
5. Database and data dictionary.
6. API, event, SSE, webhook, and integration contracts.
7. State machines.
8. Authorization matrix.
9. Test strategy and acceptance traceability.
10. Operational runbooks.
11. Application implementation.

Lower-level artifacts must remain consistent with higher-level approved artifacts.

## 4. Required Metadata

Every authoritative document must include:

- Document ID.
- Title.
- Version.
- Status.
- Owner.
- Last updated date.
- Related documents.
- Change summary.

## 5. Ownership

- Product Owner owns PRD scope, priorities, and acceptance.
- Product and engineering jointly own the FSD.
- Technical Lead owns architecture, technical design, and ADRs.
- Backend owners own data and API contracts.
- Security owner owns authorization, threat model, and audit policy.
- QA owner owns test strategy and traceability.
- Operations owner owns deployment and runbooks.

One person may hold multiple roles during MVP, but ownership responsibilities remain distinct.

## 6. Review Rules

A document may move to Approved only when:

- Required sections are complete.
- Upstream dependencies are approved or explicitly referenced as draft dependencies.
- Terminology is consistent with the glossary.
- Cross-document impact is checked.
- Open decisions are recorded.
- Acceptance criteria are testable.

## 7. Change Rules

A material change to an approved document requires:

1. Change request or clearly described pull request.
2. Impact assessment.
3. Update of all affected dependent documents.
4. Version increment.
5. Review and approval.
6. Update of application submodule references when implementation changes.

## 8. Traceability

Each functional requirement should have a stable requirement ID. Downstream documents, test cases, issues, and implementation PRs should reference those IDs.

Recommended format:

- Product requirement: `PRD-FR-###`
- Functional requirement: `FSD-FR-###`
- Business rule: `BR-###`
- Non-functional requirement: `NFR-###`
- Acceptance criterion: `AC-###`

## 9. Prohibited Practices

- Defining undocumented endpoints directly in code.
- Adding status values without updating state machines.
- Adding permissions without updating the authorization matrix.
- Changing persistent data structures without updating database design.
- Implementing cross-tenant access rules only in UI code.
- Keeping the only copy of an important decision in chat, issue comments, or commit messages.

## 10. Governance Review Cadence

Governance should be reviewed:

- Before each major release.
- After a major architecture change.
- After a production incident caused by documentation inconsistency.
- When a new application repository is added.
