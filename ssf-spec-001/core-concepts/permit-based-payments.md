---
document: SSF-SPEC-001/core-concepts/permit-based-payments.md
spec: SSF-SPEC-001
version: 1.0.0
status: Draft
date: 2026-03-04
author: "Adalton Reis <reis@stablecoinstack.org>"
organization: Stablecoin Stack Foundation
license: Apache License 2.0
---

# Permit-Based Payments

---

## The Problem: Gas Tokens

Normally, a user who wants to transfer an ERC-20 token must hold native ETH (or the equivalent gas token) to pay for the transaction. This creates friction: payers must manage two separate assets — the stablecoin they want to send and a gas token they may not hold or want to hold.

## The Solution: ERC-2612 Permits

**ERC-2612** extends standard ERC-20 tokens with a `permit()` function. Instead of sending an on-chain approval transaction, a token holder signs a structured off-chain message carrying the same authorisation. This signed message — the **Permit** — can then be submitted by any account on their behalf.

In the Stablecoin Stack, the submitting account is the **Relayer** — a processor-controlled account that holds ETH and pays for gas. The payer never touches a gas token.

## How It Works

```
1. Payer signs a Permit off-chain
   "I authorise the Settlement Contract to pull X tokens from my wallet
    until deadline D, using my current nonce N."

2. Payer signs the operation parameters off-chain
   "I authorise this specific payment: token T, beneficiary B, amount A, ref R."

3. Payer submits both signatures to the Payment Processor (HTTPS)

4. Processor validates both signatures off-chain

5. Relayer broadcasts a single transaction to the Settlement Contract

6. Settlement Contract:
   a. calls token.permit(owner, spender, value, deadline, v, r, s)
      → token contract grants allowance
   b. calls token.transferFrom(owner, contract, amount)
      → tokens move from payer to contract
   c. distributes principal to beneficiary, fees to recipients
```

The payer never sends a blockchain transaction. Their only actions are two off-chain signatures and one HTTPS request.

## EIP-712: Typed Data Signing

Both signatures are produced using **EIP-712** — a standard for signing structured, human-readable data rather than raw byte strings. EIP-712 binds each signature to:

- a specific **chain** (via `chainId`)
- a specific **contract** (via `verifyingContract`)
- a specific **data type** (via the type hash)

This prevents a signature produced for one chain or contract from being replayed on another. The digest is constructed as:

```
digest = keccak256(
    0x1901
    || domainSeparator
    || hashStruct(message)
)
```

The **Permit Signature** is signed against the token contract's domain separator. The **Binding Signature** is signed against the Settlement Contract's domain separator. The two domains are always distinct.

## Nonces and Replay Protection

ERC-2612 maintains a per-owner nonce on each token contract. Each permit consumes one nonce, making it impossible to replay the same Permit Signature. The Settlement Contract additionally records the Binding Signature's EIP-712 digest in a `usedHashes` registry — making the operation-level authorisation strictly single-use, independent of the nonce mechanism. Both controls are active simultaneously.

---

## Related Documents

- [Dual-Signature Pattern](./dual-signature-pattern.md) — the two-signature model in detail
- [Payment Flow](./payment-flow.md) — where this fits in the end-to-end flow
- [Formal Specification, Section 4](../specifications/ssf-spec-001.md#4-cryptographic-conventions) — normative cryptographic conventions
