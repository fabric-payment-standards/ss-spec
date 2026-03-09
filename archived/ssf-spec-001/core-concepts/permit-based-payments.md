---
sidebar_position: 2
---

# Permit-Based Payments

## The Problem: Gas Tokens

Normally, a user who wants to transfer an ERC-20 token must hold native ETH (or the equivalent gas token on the target network) to pay for the transaction. This creates friction: payers must manage two separate assets — the stablecoin they want to send, and a gas token they may not hold or want to hold.

## The Solution: ERC-2612 Permits

**ERC-2612** extends standard ERC-20 tokens with a `permit()` function. Instead of sending an approval transaction from their wallet, a token holder can sign a structured off-chain message that carries the same authorisation. This signed message — the **Permit** — can then be submitted by *any* account on their behalf.

In the Stablecoin Stack, the account that submits the transaction is the **Relayer** — a processor-controlled account that holds ETH and pays for gas. The payer never touches a gas token.

## How It Works: Step by Step

```
1. Payer signs a Permit off-chain
   → "I authorise the Settlement Contract to pull X tokens from my wallet
      until deadline D, using my current nonce N."

2. Payer signs the operation parameters off-chain  
   → "I authorise this specific payment: token T, beneficiary B, amount A, ref R."

3. Payer submits both signatures to the Payment Processor API (HTTPS)

4. Processor validates both signatures

5. Relayer broadcasts a single transaction to the Settlement Contract

6. Settlement Contract:
   a. Calls token.permit(owner, spender, value, deadline, v, r, s)
      → token contract verifies the Permit signature and grants allowance
   b. Calls token.transferFrom(owner, contract, amount)
      → tokens move from payer to contract
   c. Distributes principal to beneficiary, fees to recipients
```

The payer never sends a blockchain transaction. Their only action is two off-chain signatures and one HTTPS request.

## The Two-Signature Pattern

Every submission carries exactly two independent signatures:

| Signature         | Signs                                           | Verified by               |
| ----------------- | ----------------------------------------------- | ------------------------- |
| **Permit Signature**  | ERC-2612 `Permit` typed-data                | The ERC-2612 token contract on-chain |
| **Binding Signature** | The full operation parameters (token, amount, beneficiary, ref) | The Settlement Contract on-chain, and the processor off-chain |

Separating these two signatures is a deliberate security choice. If there were only one signature, the Payment Processor could potentially reuse a permit authorisation in an unintended context. With two signatures, neither the token contract nor the processor can act unilaterally outside the exact parameters the payer authorised.

## EIP-712: Typed Data Signing

Both signatures are produced using **EIP-712** — a standard for signing structured, human-readable data rather than raw byte strings. EIP-712 binds each signature to:

- a specific **chain** (via `chainId`)
- a specific **contract** (via `verifyingContract`)
- a specific **data type** (via the type hash)

This prevents a signature produced for one chain or contract from being replayed on another.

The EIP-712 digest follows the form:

```
digest = keccak256(
    0x1901
    || domainSeparator
    || hashStruct(message)
)
```

where `domainSeparator` encodes the `name`, `version`, `chainId`, and `verifyingContract` of the Settlement Contract.

## Nonces and Replay Protection

ERC-2612 maintains a per-owner nonce on each token contract. Each permit consumes one nonce, making it impossible to replay the same permit twice. Additionally, the Settlement Contract records the Binding Signature's digest in a `usedHashes` registry — meaning the operation-level authorisation is also strictly single-use, independent of the nonce mechanism.

## Related Documents

- [Payment Submission](../overview/payment-submission.md) — overview of the submission flow
- [Dual-Signature Pattern](../../ssf-spec-002/core-concepts/dual-signature-pattern.md) — deeper look at the two-signature model
- [SSF-SPEC-001](../specifications/ssf-spec-001.md) — formal specification of payloads and signatures
