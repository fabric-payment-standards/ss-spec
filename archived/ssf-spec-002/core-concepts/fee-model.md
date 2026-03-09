---
sidebar_position: 4
---

# Fee Model

## Overview

Every payment processed by the Settlement Contract is subject to up to two fee components:

| Fee Component   | Type                          | When Applied                                          |
| --------------- | ----------------------------- | ----------------------------------------------------- |
| **Base Fee**    | Absolute amount, in token units | Every transfer, unconditionally                     |
| **Acquiring Fee** | Percentage of the principal  | Only when the payment includes a non-zero Acquirer ID |

Understanding the fee model is important for building a checkout integration: the amount the payer signs in their permit must be the **total with fees**, not the principal alone.

## Key Concepts

### Principal Amount

The amount the merchant (beneficiary) will receive after all fees are deducted. This is what a checkout session typically presents as the "payment amount."

### Total With Fees

The total token amount the payer must hold and authorise. This is the value that goes into the `value` field of `PermitParams`.

```
Total With Fees = Principal Amount + Base Fee + Acquiring Fee (if applicable)
```

### Base Fee (`baseFeeAmount`)

A flat, absolute fee charged on every transfer regardless of amount. It is denominated in the smallest unit of the token being transferred (e.g., for a 6-decimal USDC token, a fee of `1_000_000` equals 1 USDC). The Administrator sets this value.

### Acquiring Fee

A percentage-based fee charged by a registered Acquirer on payments they referred. It is expressed in processor-defined units (typically basis points). Each Acquirer sets their own fee at registration, subject to the `maxAcquiringFee` ceiling enforced by the contract.

## The Zero-UUID Convention

When a payment has no associated Acquirer, the caller MUST pass the **Zero-UUID** as the `acquirerId`:

```
0x00000000000000000000000000000000
```

The Zero-UUID is a reserved identifier. No wallet can ever be registered under it. Passing it suppresses the Acquiring Fee component entirely without causing a revert.

Do not omit the `acquirerId` field or substitute a random value — pass Zero-UUID explicitly.

## Fee Calculation Helpers

The Settlement Contract exposes two pure view functions to assist integrations.

### `calculateFees(amount, acquirerId)`

Use this when you know the **principal** and want to compute the fees to add.

```
Input:  principal = 100 USDC, acquirerId = 0x1234...
Output: totalFee  = 2.5 USDC
→ Payer must sign a permit for 102.5 USDC
```

### `breakdownTransferAmount(totalWithFees, acquirerId)`

Use this when you know the **total** (e.g., from an existing permit value) and want to decompose it.

```
Input:  totalWithFees = 102.5 USDC, acquirerId = 0x1234...
Output: principalAmount = 100 USDC, totalFees = 2.5 USDC
```

These two functions are inverses of each other. Call them before constructing a permit to ensure the signed amount is correct.

## Flow: Computing the Correct Permit Value

```
1. Checkout Engine issues: principal = P, acquirerId = A

2. Client calls: calculateFees(P, A) → fee = F

3. Client computes: permitValue = P + F

4. Client signs: PermitParams { value: permitValue, ... }

5. On-chain: Settlement Contract receives permitValue,
             deducts F, credits P to beneficiary
```

Getting this wrong in either direction causes a revert: too little and the transfer cannot cover all amounts; too much and the payer overpays.

## Acquirer Registration Fee (`acquiringPrice`)

Separate from the per-payment fee model, registering as an Acquirer requires a one-time fee — `acquiringPrice` — paid in a supported stablecoin. This is a flat price set by the Administrator and fetched from the contract before constructing a `BuyAcquiringPackRequest`. The permit for this flow must be signed for exactly `acquiringPrice`.

## Related Documents

- [Settlement Contract](../overview/settlement-contract.md) — overview
- [SSF-SPEC-002, Section 6](../specifications/ssf-spec-002.md#6-fee-model) — normative fee model definition
- [Contract Interface Reference](../reference/contract-interface.md#calculatefees) — function signatures
- [Submit a Payment](../../ssf-spec-001/guides/submit-a-payment.md) — how to apply this in practice
