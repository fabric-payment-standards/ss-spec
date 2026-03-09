# Change Log — SSF-SPEC-002

This file records all changes to the SSF-SPEC-002 specification group (The Settlement Contract — Canonical). Changes to companion specifications (SSF-SPEC-000, SSF-SPEC-001, etc.) are recorded in their respective change logs.

---

## [1.0.0] — 2026-03-04

**Status:** Draft
**Authors:** Adalton Reis \<reis@stablecoinstack.org\>
**Summary:** Initial release of SSF-SPEC-002 and its supporting documentation under the Layered Docs structure. Terminology aligned with SSF-SPEC-001: `acquirring` → `acquiring`, `distributor` → `acquirer`, `referral` identifier → `acquirerId`, `breakdowTransferAmount` → `breakdownTransferAmount`, `AcquirringCreated` → `AcquirerCreated`, `CommissonGenerated` → `CommissionGenerated`.

**Files introduced:**

- `overview/ssf-spec-002/settlement-contract.md` — What the Settlement Contract does, its role in the stack, and its three core design guarantees.
- `core-concepts/ssf-spec-002/fee-model.md` — Base fee, acquiring fee, Zero-UUID convention, and the two fee calculation helpers.
- `core-concepts/ssf-spec-002/dual-signature-pattern.md` — How the Permit Signature and Binding Signature work together, verification flow, and the `usedHashes` replay protection mechanism.
- `specifications/ssf-spec-002/ssf-spec-002.md` — Normative specification: design principles, state variables, events, fee model, functions, dual-signature pattern, and security considerations.
- `reference/ssf-spec-002/contract-interface.md` — Flat reference for all state variables, functions, parameters, return values, and events.
- `guides/ssf-spec-002/integrate-settlement-contract.md` — Guide for processor operators: reading parameters, computing fees, verifying acquirers, querying balances, listening for events, and pre-validating permit signatures.
- `governance/ssf-spec-002/changelog.md` — This file.

---

## License

Copyright © 2026 Stablecoin Stack Foundation. All rights reserved.

All specifications and documentation in this repository are published under the Apache License, Version 2.0. Unless required by applicable law or agreed to in writing, material distributed under the License is distributed on an "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
