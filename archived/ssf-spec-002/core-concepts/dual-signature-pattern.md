---
sidebar_position: 3
---

# Dual-Signature Pattern

## Overview

Every operation submitted to the Stablecoin Stack — whether a payment transfer or an acquirer registration — requires exactly two independent ECDSA signatures. This is the **dual-signature pattern**.

The two signatures serve different purposes, are verified by different components, and protect against different classes of attack.

## The Two Signatures

### Permit Signature (`permitSig` / `v1, r1, s1`)

| Property        | Value                                                                          |
| --------------- | ------------------------------------------------------------------------------ |
| **What it signs** | The ERC-2612 `Permit` typed-data: `owner`, `spender`, `value`, `nonce`, `deadline` |
| **Produced by** | The token holder (`owner`)                                                     |
| **Verified by** | The ERC-2612 token contract, inside its `permit()` function                   |
| **Effect**      | Grants the Settlement Contract an allowance to pull tokens from the owner's wallet |
| **Replay protection** | The ERC-2612 nonce on the token contract — consumed on first use         |

This signature is about **token authorisation**. It answers the question: *"Is the token holder allowing their tokens to be moved?"*

### Binding Signature (`payWithPermitSig` / `v2, r2, s2`)

| Property        | Value                                                                          |
| --------------- | ------------------------------------------------------------------------------ |
| **What it signs** | The full operation parameters: token address, beneficiary/acquiring address, amount, `ref`, deadline |
| **Produced by** | The payer (same account as `owner`)                                            |
| **Verified by** | The Payment Processor (off-chain) and the Settlement Contract (on-chain)       |
| **Effect**      | Authorises the processor to submit this specific operation                     |
| **Replay protection** | The `usedHashes` registry in the Settlement Contract — the digest is consumed on first use |

This signature is about **operation authorisation**. It answers the question: *"Is the payer authorising this specific payment with these exact parameters?"*

## Why Two Signatures Are Required

Consider what would be possible with only a Permit Signature:

- The Payment Processor could reuse a valid permit to send tokens to a different beneficiary, change the amount, or alter the reference.
- An intercepted permit could be replayed with modified parameters before the original transaction is broadcast.

The Binding Signature eliminates this. Because it covers the full operation parameters, any modification — even a single byte change to the beneficiary address — would produce a digest mismatch and be rejected.

Consider what would be possible with only a Binding Signature:

- There would be no standard mechanism for the Settlement Contract to pull tokens from the payer. The payer would need to issue a separate on-chain approval, reintroducing the gas requirement.

The Permit Signature eliminates this. It enables the gasless `permit()` + `transferFrom()` pattern.

Each signature is therefore **necessary** and **insufficient alone**.

## Verification Flow

```
Client Wallet
  │
  ├─ signs Permit typed-data    → permitSig  (v1, r1, s1)
  └─ signs operation params     → payWithPermitSig  (v2, r2, s2)
         │
         ▼
Payment Processor API
  ├─ recovers signer from payWithPermitSig
  ├─ verifies signer == permitParams.owner        [off-chain]
  ├─ recomputes digest; verifies hash field matches [off-chain]
  └─ optionally pre-verifies permitSig using token's domain separator
         │
         ▼
Settlement Contract
  ├─ checks digest(payWithPermitSig) not in usedHashes
  ├─ records digest in usedHashes                  [replay protection]
  ├─ calls token.permit(owner, spender, value, deadline, v1, r1, s1)
  │    └─ token contract verifies permitSig internally
  └─ calls token.transferFrom(owner, contract, amount)
```

## Relationship to `usedHashes`

The Settlement Contract maintains a `usedHashes` mapping. Every time a `transferWithPermit` or `buyAcquiringPack` call succeeds, the EIP-712 digest of the Binding Signature is recorded. Any future transaction presenting the same digest reverts immediately.

This provides operation-level replay protection that is independent of and complementary to the ERC-2612 nonce:

| Mechanism          | Protects against                                              | Maintained by             |
| ------------------ | ------------------------------------------------------------- | ------------------------- |
| ERC-2612 nonce     | Permit Signature replay across different operations           | The ERC-2612 token contract |
| `usedHashes`       | Binding Signature replay even if a new permit were issued     | The Settlement Contract   |

## Related Documents

- [Permit-Based Payments](../../ssf-spec-001/core-concepts/permit-based-payments.md) — the broader gasless payment model
- [SSF-SPEC-001, Section 6.1](../../ssf-spec-001/specifications/ssf-spec-001.md#61-two-signature-pattern) — normative definition
- [SSF-SPEC-002, Section 8](../specifications/ssf-spec-002.md#8-dual-signature-pattern) — on-chain implementation
