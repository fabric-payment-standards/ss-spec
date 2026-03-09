---
sidebar_position: 3
---

# The Settlement Contract

## What It Is

The Settlement Contract is the on-chain component of the Stablecoin Stack. It is a Solidity smart contract deployed on Ethereum-compatible networks that serves as the authoritative execution layer for all payment operations.

When the Payment Processor broadcasts a transaction, the Settlement Contract takes over: it verifies the cryptographic authorisations, moves the tokens, distributes the fees, and — in the case of acquirer registration — assigns a permanent on-chain identity.

## What It Does

| Responsibility              | Description                                                                                       |
| --------------------------- | ------------------------------------------------------------------------------------------------- |
| **Signature verification**  | Validates the Permit Signature via the ERC-2612 token contract and the Binding Signature itself.  |
| **Token transfer**          | Pulls the full amount from the payer via `permit` + `transferFrom` and credits the beneficiary.   |
| **Fee distribution**        | Deducts the base fee and, if applicable, the acquirer's commission, and routes each to its recipient. |
| **Acquirer registration**   | Accepts a one-time fee and permanently registers a wallet as an acquirer with a unique Acquirer ID. |
| **Replay protection**       | Records each processed Binding Signature hash to prevent any operation from being executed twice. |

## Design Guarantees

**Payer-agnostic.** The contract does not require the payer to submit their own transaction. Any account — including the Relayer — can broadcast the transaction as long as the signatures are valid.

**Non-custodial administration.** The Administrator can adjust fees and pricing but has no mechanism to move user funds.

**Atomic settlement.** Either every step of a payment succeeds, or the entire transaction reverts. There is no partial execution.

## Where It Fits in the Stack

```
Client Wallet
     │
     │  (signed payload over HTTPS)
     ▼
Payment Processor API        ← SSF-SPEC-001
     │
     │  (broadcast transaction)
     ▼
Settlement Contract          ← SSF-SPEC-002  (this document's subject)
     │
     ├── ERC-2612 Token Contract  (permit verification)
     └── Fee Recipients / Acquirer Balances
```

## Related Documents

- [Fee Model](../core-concepts/fee-model.md) — how base fees and acquiring fees are calculated
- [Dual-Signature Pattern](../core-concepts/dual-signature-pattern.md) — why two signatures are required and how they interact
- [SSF-SPEC-002](../specifications/ssf-spec-002.md) — the full formal specification
- [Contract Interface Reference](../reference/contract-interface.md) — all functions, parameters, and events
- [Integrate the Settlement Contract](../guides/integrate-settlement-contract.md) — guide for processor operators
