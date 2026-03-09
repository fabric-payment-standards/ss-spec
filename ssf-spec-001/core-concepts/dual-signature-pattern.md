---
document: SSF-SPEC-001/core-concepts/dual-signature-pattern.md
spec: SSF-SPEC-001
version: 1.0.0
status: Draft
date: 2026-03-04
author: "Adalton Reis <reis@stablecoinstack.org>"
organization: Stablecoin Stack Foundation
license: Apache License 2.0
---

# Dual-Signature Pattern

Every operation in the Stablecoin Stack requires exactly two independent ECDSA signatures. They serve different purposes, are verified by different components, and protect against different classes of attack. Neither is sufficient alone.

---

## The Two Signatures

### Permit Signature (`permitSig` / `v1, r1, s1`)

| Property | Value |
| -------- | ----- |
| **Signs** | ERC-2612 `Permit` typed-data: `owner`, `spender`, `value`, `nonce`, `deadline` |
| **Produced by** | The token holder (`owner`) |
| **Verified by** | The ERC-2612 token contract inside its `permit()` function |
| **Effect** | Grants the Settlement Contract an allowance to pull tokens from the owner's wallet |
| **Replay protection** | ERC-2612 nonce — consumed on first use |
| **Domain** | Token contract's EIP-712 domain separator |

This signature answers: *"Is the token holder allowing their tokens to be moved?"*

### Binding Signature (`payWithPermitSig` / `v2, r2, s2`)

| Property | Value |
| -------- | ----- |
| **Signs** | Full operation parameters: token, beneficiary/acquiring address, amount, `ref`, deadline |
| **Produced by** | The payer (same account as `owner`) |
| **Verified by** | The Payment Processor (off-chain) and the Settlement Contract (on-chain) |
| **Effect** | Authorises the processor to submit this specific operation |
| **Replay protection** | `usedHashes` registry in the Settlement Contract — digest consumed on first use |
| **Domain** | Settlement Contract's EIP-712 domain separator |

This signature answers: *"Is the payer authorising this specific payment with these exact parameters?"*

---

## Why Both Are Required

**Without the Binding Signature:** the Permit Signature alone grants an allowance but does not determine where the funds go. The Payment Processor could alter the beneficiary, amount, or reference without detection.

**Without the Permit Signature:** the Binding Signature alone authorises the operation but provides no gasless mechanism to pull tokens. The payer would need an on-chain approval transaction, reintroducing the gas requirement.

Together: the Permit Signature enables gasless token movement; the Binding Signature binds the processor to the exact parameters the payer authorised.

---

## Verification Flow

```
Client Wallet
  ├── signs ERC-2612 Permit typed-data  →  permitSig  (v1, r1, s1)
  └── signs operation parameters        →  payWithPermitSig  (v2, r2, s2)
            │
            ▼
Payment Processor (off-chain)
  ├── recovers signer from payWithPermitSig
  ├── verifies signer == permitParams.owner
  ├── recomputes digest; verifies hash field matches
  └── optionally pre-verifies permitSig using token's domain separator
            │
            ▼
Settlement Contract (on-chain)
  ├── checks digest(payWithPermitSig) not in usedHashes
  ├── records digest in usedHashes                    ← replay protection
  ├── calls token.permit(owner, spender, value, deadline, v1, r1, s1)
  │       └── token contract verifies permitSig
  └── calls token.transferFrom(owner, contract, amount)
```

---

## The `usedHashes` Registry

The Settlement Contract maintains a `usedHashes` mapping. Every time `transferWithPermit` or `buyAcquiringPack` succeeds, the EIP-712 digest of the Binding Signature is recorded. Any future transaction presenting the same digest reverts.

This provides operation-level replay protection that is independent of and complementary to the ERC-2612 nonce:

| Mechanism | Protects against | Maintained by |
| --------- | ---------------- | ------------- |
| ERC-2612 nonce | Permit Signature replay | ERC-2612 token contract |
| `usedHashes` | Binding Signature replay even if a new permit were issued | Settlement Contract |

---

## Related Documents

- [Permit-Based Payments](./permit-based-payments.md) — the broader gasless payment model
- [Formal Specification, Section 6.1](../specifications/ssf-spec-001.md#61-two-signature-pattern) — normative definition
- [Formal Specification, Section 8](../specifications/ssf-spec-001.md#8-dual-signature-pattern) — on-chain implementation
- [Security Model](../specifications/ssf-spec-001.md#11-security-model) — threat model and mitigations
