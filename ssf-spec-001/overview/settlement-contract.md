---
document: SSF-SPEC-001/overview/settlement-contract.md
spec: SSF-SPEC-001
version: 1.0.0
status: Draft
date: 2026-03-04
author: "Adalton Reis <reis@stablecoinstack.org>"
organization: Stablecoin Stack Foundation
license: Apache License 2.0
---

# The Settlement Contract

The Settlement Contract is the on-chain trust anchor of the Stablecoin Stack. It is a Solidity smart contract deployed on Ethereum-compatible networks that serves as the authoritative execution layer for all payment operations.

Its correctness can be verified independently of any operator. All payment value flows through it. No off-chain component can alter its execution.

---

## What It Does

| Responsibility | Description |
| -------------- | ----------- |
| **Signature verification** | Validates the Permit Signature via the ERC-2612 token contract and the Binding Signature against its own logic. |
| **Token transfer** | Pulls the full amount from the payer via `permit()` + `transferFrom()` and credits the principal to the beneficiary. |
| **Fee distribution** | Deducts the base fee and, if applicable, the acquirer's commission, routing each to its recipient. |
| **Acquirer registration** | Accepts a one-time fee and permanently registers a wallet as an Acquirer with a unique Acquirer ID. |
| **Replay protection** | Records each processed Binding Signature hash in `usedHashes`, preventing any operation from being executed twice. |

---

## Design Guarantees

**Payer-agnostic.** Any account — including the Relayer — can broadcast the transaction. Validity depends entirely on the signatures, not on who submits.

**Non-custodial administration.** The Administrator can adjust fees and pricing but has no mechanism to move participant funds.

**Atomic settlement.** Either every step of a payment succeeds, or the entire transaction reverts. No partial execution is possible.

---

## Where It Fits in the Stack

```
broadcast-submitter (Relayer)
        │
        │  transferWithPermit() or buyAcquiringPack()
        ▼
Settlement Contract          ← this component
        │
        ├── ERC-2612 Token Contract  (permit() verification)
        └── Fee recipients / Acquirer internal balances
```

---

## Related Documents

- [Fee Model](../core-concepts/fee-model.md) — how base fees and acquiring fees are calculated
- [Dual-Signature Pattern](../core-concepts/dual-signature-pattern.md) — the two-signature verification model
- [Contract Interface Reference](../reference/contract-interface.md) — all functions, parameters, and events
- [Integrate the Settlement Contract](../guides/integrate-settlement-contract.md) — guide for processor operators
