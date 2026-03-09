---
sidebar_position: 3
---

# Buy an Acquiring Pack

Step-by-step guide for constructing, signing, and submitting a `BuyAcquiringPackRequest` to register a new Acquirer.

**Prerequisites:** familiarity with EIP-712 typed data signing and access to an ERC-2612-compliant stablecoin wallet.

---

## Overview

Registering as an Acquirer requires a one-time payment of `acquiringPrice` tokens to the Settlement Contract. Upon settlement, a unique Acquirer ID (16-byte UUID) is permanently assigned to the `acquiring` wallet address on-chain.

The payer and the wallet being registered do not need to be the same address — a third party may pay the registration fee on behalf of the intended acquirer.

---

## Step 1 — Fetch `acquiringPrice` from the Settlement Contract

```javascript
const acquiringPrice = await settlementContract.acquiringPrice();
```

This is the exact amount the permit must be signed for. Do not estimate or approximate.

---

## Step 2 — Choose the `feePercent`

Decide the acquiring fee percentage your acquirer will charge on referred payments. This value must not exceed `maxAcquiringFee` set by the Settlement Contract Administrator.

```javascript
const maxAcquiringFee = await settlementContract.maxAcquiringFee();
// Choose feePercent in [0, maxAcquiringFee]
const feePercent = chosenFeePercent;
```

---

## Step 3 — Fetch the ERC-2612 Nonce

Retrieve the current permit nonce for the **payer's** address on the chosen token contract:

```javascript
const nonce = await tokenContract.nonces(payerAddress);
```

---

## Step 4 — Construct `PermitParams`

```javascript
const permitParams = {
  owner:    payerAddress,
  spender:  settlementContractAddress,
  value:    acquiringPrice,     // must equal feeValue exactly
  nonce:    nonce,
  deadline: deadline,           // Unix timestamp; choose with sufficient margin
};
```

Note: `value` MUST equal `acquiringPrice`. This is validated on-chain — any mismatch reverts.

---

## Step 5 — Sign the ERC-2612 Permit

Sign against the **token contract's** domain separator:

```javascript
const permitDomain = {
  name:              await tokenContract.name(),
  version:           "1",
  chainId:           chainId,
  verifyingContract: tokenContractAddress,
};

const permitTypes = {
  Permit: [
    { name: "owner",    type: "address" },
    { name: "spender",  type: "address" },
    { name: "value",    type: "uint256" },
    { name: "nonce",    type: "uint256" },
    { name: "deadline", type: "uint256" },
  ],
};

const { v, r, s, hash } = await signer._signTypedData(
  permitDomain,
  permitTypes,
  permitParams
);

const permitSig = { hash, v, r, s };
```

Normalise `v`: if `0` or `1`, add `27`.

---

## Step 6 — Construct `BuyAcquiringPackPermitParams`

```javascript
const buyAcquiringPackParams = {
  token:        tokenContractAddress,
  feeValue:     acquiringPrice,        // must equal permitParams.value
  acquiring:    acquiringAddress,      // wallet to register as acquirer
  permitParams: permitParams,
};
```

---

## Step 7 — Sign the Operation (Binding Signature)

Sign against the **Settlement Contract's** domain separator:

```javascript
const settlementDomain = {
  name:              "StablecoinStack",
  version:           "1",
  chainId:           chainId,
  verifyingContract: settlementContractAddress,
};

const buyAcquiringPackTypes = {
  BuyAcquiringPackPermitParams: [
    { name: "token",        type: "address" },
    { name: "feeValue",     type: "uint256" },
    { name: "acquiring",    type: "address" },
    { name: "permitParams", type: "PermitParams" },
  ],
  PermitParams: [
    { name: "owner",    type: "address" },
    { name: "spender",  type: "address" },
    { name: "value",    type: "uint256" },
    { name: "nonce",    type: "uint256" },
    { name: "deadline", type: "uint256" },
  ],
};

const { v, r, s, hash } = await signer._signTypedData(
  settlementDomain,
  buyAcquiringPackTypes,
  buyAcquiringPackParams
);

const payWithPermitSig = { hash, v, r, s };
```

---

## Step 8 — Assemble and Submit the Payload

```javascript
const buyAcquiringPackRequest = {
  buyAcquiringPackParams,
  payWithPermitSig,
  permitSig,
};

const response = await fetch("https://processor.example.com/v1/acquiring", {
  method:  "POST",
  headers: { "Content-Type": "application/json" },
  body:    JSON.stringify(buyAcquiringPackRequest),
});
```

The request MUST be sent over TLS 1.2 or higher.

---

## Step 9 — Retrieve Your Acquirer ID

After successful settlement, the Settlement Contract emits an `AcquirerCreated` event. Listen for it to retrieve your assigned Acquirer ID:

```javascript
settlementContract.on("AcquirerCreated", (acquirerId, wallet, feePercent) => {
  if (wallet.toLowerCase() === acquiringAddress.toLowerCase()) {
    console.log("Acquirer ID:", acquirerId);
    // Store this acquirerId — you will embed it in payment refs
  }
});
```

Alternatively, query the event log after the transaction is confirmed.

---

## Step 10 — Handle the Response

| HTTP Status | Meaning                                                                                 |
| ----------- | --------------------------------------------------------------------------------------- |
| `200`       | Payload accepted and transaction broadcast. Listen for `AcquirerCreated` event.         |
| `400`       | `STRUCTURAL_ERROR` or `SEMANTIC_ERROR` — check the error body for the violated rule.    |
| `422`       | `CRYPTOGRAPHIC_ERROR` — signature verification failed.                                  |
| `502`       | `BROADCAST_ERROR` — on-chain transaction failed. The acquiring address is NOT registered. |

Common `SEMANTIC_ERROR` causes for this flow: `feePercent` exceeds `maxAcquiringFee`, `acquiring` is already registered, or `price` does not match the current `acquiringPrice` (re-fetch and retry).

---

## Using Your Acquirer ID in Payments

Once registered, embed your Acquirer ID in the `ref` field of every `TransferRequest` you originate. See [Submit a Payment — Step 6](./submit-a-payment.md#step-6--construct-the-ref-field) for the concatenation format.

Payments that include your Acquirer ID in `ref` will trigger a `CommissionGenerated` event on the Settlement Contract, crediting your `feePercent` share to your internal balance.

---

## Checklist

- [ ] `acquiringPrice` fetched from the contract immediately before signing (it may change)
- [ ] `permitParams.value` equals `acquiringPrice` exactly
- [ ] `buyAcquiringPackParams.feeValue` equals `permitParams.value`
- [ ] `feePercent` does not exceed `maxAcquiringFee`
- [ ] `acquiring` address is not already registered (check via `getAcquiringWallet`)
- [ ] `permitSig` was signed against the **token contract** domain
- [ ] `payWithPermitSig` was signed against the **Settlement Contract** domain
- [ ] `v` values are `27` or `28`
- [ ] Request is sent over HTTPS

---

## Related Documents

- [Fee Model](../../ssf-spec-002/core-concepts/fee-model.md) — acquiring fee mechanics
- [Dual-Signature Pattern](../../ssf-spec-002/core-concepts/dual-signature-pattern.md) — why two signatures are required
- [Payload Fields Reference](../reference/payload-fields.md) — field-level constraints
- [SSF-SPEC-001, Section 8.2](../specifications/ssf-spec-001.md#82-acquirer-service-purchase--step-by-step) — normative step-by-step
- [Contract Interface Reference](../../ssf-spec-002/reference/contract-interface.md#buyacquiringpack) — on-chain function spec
