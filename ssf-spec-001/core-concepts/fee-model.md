---
document: SSF-SPEC-001/core-concepts/fee-model.md
spec: SSF-SPEC-001
version: 1.0.0
status: Draft
date: 2026-03-04
author: "Adalton Reis <reis@stablecoinstack.org>"
organization: Stablecoin Stack Foundation
license: Apache License 2.0
---

# Fee Model

Every payment processed by the Settlement Contract is subject to up to two fee components. Understanding the fee model is essential for checkout integrations: the amount the payer signs in their permit must be the **total with fees**, not the principal alone.

---

## Fee Components

| Fee | Type | When Applied | Recipient |
| --- | ---- | ------------ | --------- |
| **Base Fee** (`baseFeeAmount`) | Absolute, in token units | Every transfer, unconditionally | Payment Processor fee recipient |
| **Acquiring Fee** | Percentage of the principal | Only when payment includes a non-zero Acquirer ID | Registered Acquirer's internal balance |

---

## Key Concepts

**Principal Amount** — the amount the merchant (beneficiary) will receive after all fees are deducted. This is what a checkout session typically presents as the "payment amount."

**Total With Fees** — the total token amount the payer must hold and authorise. This is the value that goes into `PermitParams.value`.

```
Total With Fees = Principal Amount + Base Fee + Acquiring Fee (if applicable)
```

**Base Fee** — a flat absolute fee in the smallest unit of the token being transferred. Set by the Administrator and intended to cover at minimum the gas cost of on-chain execution.

**Acquiring Fee** — a percentage-based fee set by each Acquirer at registration, subject to the `maxAcquiringFee` ceiling enforced by the contract. Expressed in processor-defined units.

---

## The Zero-UUID Convention

When a payment has no associated Acquirer, the caller MUST pass the **Zero-UUID** as the `acquirerId`:

```
0x00000000000000000000000000000000
```

This is a reserved identifier. No wallet can ever be registered under it. Passing it suppresses the Acquiring Fee entirely without causing a revert. Do not omit the field or substitute an arbitrary value — pass Zero-UUID explicitly.

---

## Fee Calculation Helpers

The Settlement Contract provides two pure view functions.

**`calculateFees(amount, acquirerId)`** — use when you know the principal and need to compute what to add.

```
principal = 100 USDC, acquirerId = 0x1234...
→ totalFee = 2.50 USDC
→ permit must be signed for 102.50 USDC
```

**`breakdownTransferAmount(totalWithFees, acquirerId)`** — use when you know the total and need to decompose it.

```
totalWithFees = 102.50 USDC, acquirerId = 0x1234...
→ principalAmount = 100.00 USDC
→ totalFees       = 2.50 USDC
```

These two functions are inverses of each other. Always call `calculateFees` before constructing a permit to ensure the signed amount is correct.

---

## The Acquiring Model

The Stablecoin Stack incorporates a first-class acquiring model that allows third-party participants — **Acquirers** — to distribute the payment service and earn a share of the processing fee. This model is analogous to the acquirer role in traditional card payment networks, but implemented transparently and on-chain.

When a payer includes an Acquirer ID in a payment, the Settlement Contract automatically distributes the acquiring fee to the registered acquirer's internal balance. This distribution is atomic with the payment itself and is reflected in the `CommissionGenerated` event.

An Acquirer who wishes to be discoverable by wallet users may publish their information to the **basic-data-server**, where they appear as a **Service Provider**. This allows wallets to present users with a curated list of available providers, enabling selection based on fee rates, reputation, or other criteria.

---

## Acquirer Registration Fee

Separate from the per-payment fee model, registering as an Acquirer requires a one-time payment of `acquiringPrice` tokens to the Settlement Contract. This is fetched from the contract before constructing a `BuyAcquiringPackRequest`. The permit for this flow must be signed for exactly `acquiringPrice`.

---

## Related Documents

- [Buy an Acquiring Pack](../guides/buy-acquiring-pack.md) — registration guide
- [Submit a Payment](../guides/submit-a-payment.md) — how to apply fee calculation in practice
- [Formal Specification, Section 9](../specifications/ssf-spec-001.md#9-the-acquiring-model) — normative acquiring model
- [Contract Interface Reference](../reference/contract-interface.md#calculatefees) — function signatures
