# Document Status Lifecycle

**Document ID:** GOV-TC-006  
**Version:** 1.0.0  
**Status:** Approved

## 1. Statuses

### Draft

The document is under active creation and may contain unresolved decisions. It must not be treated as an implementation contract unless explicitly authorized.

### In Review

The document is structurally complete and ready for stakeholder review. Changes are expected to address review findings.

### Approved

The document is accepted as authoritative. Downstream artifacts and implementation must conform to it.

### Superseded

A newer approved version replaces the document. The document remains available for historical traceability.

### Deprecated

The document or defined behavior remains temporarily valid but is scheduled for removal or replacement.

### Archived

The document is retained only for history and must not guide current implementation.

## 2. Allowed Transitions

```text
Draft → In Review → Approved → Superseded → Archived
                    ↓
                Deprecated → Archived
```

A document may return from In Review to Draft when significant rework is needed.

## 3. Approval Conditions

A document may be Approved when:

- Required metadata is present.
- Scope and ownership are clear.
- Dependencies are identified.
- Terminology is consistent.
- Open decisions are either resolved or explicitly deferred.
- Acceptance criteria or verification expectations are defined.
- Review comments are resolved.
- Version number follows the versioning strategy.

## 4. Final File Naming

Primary versioned documents use `Final` in the filename only after approval, for example:

`PRD_Terra_Commerce_v1.0.0_Final.md`

The document metadata status should use `Approved` or `Final` consistently according to the document type. Governance treats both as authoritative approved states.

## 5. Superseding Documents

A superseding document must state:

- Which version it replaces.
- Effective date or release.
- Migration implications.
- Any behavior that remains temporarily supported.

## 6. Historical Integrity

Approved historical versions should not be silently rewritten. Corrections require a new patch version unless they only repair repository metadata without changing the document content.

## 7. Implementation Use

Application teams must use the latest Approved document version referenced by the current master-spec release or target branch. Draft documents may guide exploratory work but cannot override an Approved document.