---
document: SSF-SPEC-001/governance/versioning-policy.md
spec: SSF-SPEC-001
version: 1.0.0
status: Draft
date: 2026-03-04
author: "Adalton Reis <reis@stablecoinstack.org>"
organization: Stablecoin Stack Foundation
license: Apache License 2.0
---

# Versioning Policy

The Stablecoin Stack specification (SSF-SPEC-001) follows [Semantic Versioning 2.0.0](https://semver.org).

---

## Version Segments

| Segment | Increment when… |
| ------- | --------------- |
| **MAJOR** | A backwards-incompatible change is made to any interface, data encoding, security model, or conformance requirement. Implementers who have conformed to a previous MAJOR version MUST re-verify conformance. |
| **MINOR** | New components, flows, or requirements are added that do not break existing conformant implementations. Companion specifications covering new microservice interfaces are MINOR increments to this document. |
| **PATCH** | Editorial corrections, clarifications, typographical fixes, and other non-normative changes are made. Conformance is unaffected. |

---

## Companion Specifications

All companion specifications (SSF-SPEC-002 and above) MUST declare the version of SSF-SPEC-001 to which they conform using the `conformsTo` field in their identity table.

When a companion specification is published, SSF-SPEC-001 receives a MINOR version increment and the companion specification is added to the conformance registry in Section 19.3 of the formal specification.

---

## Deprecation Policy

Before a MAJOR version increment is published:

1. The change MUST be proposed as a documented issue in the specification repository, with a rationale.
2. A review period of no less than 30 days MUST be provided for community feedback.
3. A migration guide MUST be published alongside the new MAJOR version.

MINOR and PATCH changes may be published without a formal review period, at the discretion of the Foundation.

---

## Current Version

SSF-SPEC-001 is currently at version **1.0.0**. For a history of changes see [changelog.md](./changelog.md).
