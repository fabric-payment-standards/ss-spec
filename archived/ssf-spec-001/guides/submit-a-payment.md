---
sidebar_position: 2
---

# Submit a Payment

Step-by-step guide for constructing, signing, and submitting a `TransferRequest` to the Payment Processor API.

**Prerequisites:** familiarity with EIP-712 typed data signing, access to an ERC-2612-compliant stablecoin wallet, and a checkout session issued by the Checkout Engine.

---

## Overview

A payment submission requires:

1. Gathering session data from the Checkout Engine
2. Fetching on-chain state (nonce, fees)
3. Constructing the permit parameters
4. Signing the Permit (ERC-2612)
5. Constructing the operation parameters
6. Signing the operation (Binding Signature)
7. Assembling and submitting the payload

---

## Step 1 — Receive the Checkout Session

The Checkout Engine issues a session containing:

| Field           | Description                                                       |
| --------------- | ----------------------------------------------------------------- |
| `orderRef`      | 16-byte order reference (UUID) identifying this payment session.  |
| `beneficiary`   | Merchant's receiving address.                                     |
| `token`         | ERC-2612 stablecoin contract address.                             |
| `principal`     | The amount the merchant expects to receive, in token units.       |
| `deadline`      | Unix timestamp (seconds) after which the permit will be invalid.  |
| `acquirerId`    | 16-byte Acquirer ID, or the Zero-UUID if no acquirer is involved. |

Store all of these; you will need them in subsequent steps.

---

## Step 2 — Fetch the ERC-2612 Nonce

Call the token contract to retrieve the current permit nonce for the payer's address:

```javascript
const nonce = await tokenContract.nonces(ownerAddress);
```

This value must be used exactly as returned. A stale nonce will cause the on-chain `permit()` call to revert.

---

## Step 3 — Compute the Total Amount with Fees

Call `calculateFees` on the Settlement Contract to determine the fee to add on top of the principal:

```javascript
const totalFee = await settlementContract.calculateFees(principal, acquirerId);
const permitValue = principal + totalFee;
```

`permitValue` is the value the payer must sign. Do not use `principal` alone — the Settlement Contract will pull the full `permitValue`.

---

## Step 4 — Construct `PermitParams`

```javascript
const permitParams = {
  owner:    ownerAddress,          // payer's wallet address
  spender:  settlementContractAddress,
  value:    permitValue,           // from Step 3
  nonce:    nonce,                 // from Step 2
  deadline: deadline,              // from the checkout session
};
```

---

## Step 5 — Sign the ERC-2612 Permit

Produce the Permit Signature using EIP-712 typed data signing against the **token contract's** domain separator.

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

Normalise `v`: if the returned value is `0` or `1`, add `27`.

---

## Step 6 — Construct the `ref` Field

Concatenate the 16-byte `orderRef` from the checkout session with the 16-byte `acquirerId`:

```javascript
// Both must be 16 bytes (32 hex chars without 0x prefix)
const ref = "0x" + orderRef.replace("0x", "") + acquirerId.replace("0x", "");
```

If either value is absent, replace it with 16 zero bytes:

```javascript
const ZERO_16 = "00000000000000000000000000000000"; // 32 hex chars
const ref = "0x"
  + (orderRef  ? orderRef.replace("0x", "")  : ZERO_16)
  + (acquirerId ? acquirerId.replace("0x", "") : ZERO_16);
```

---

## Step 7 — Construct `PayWithPermitParams`

```javascript
const payWithPermitParams = {
  token:        tokenContractAddress,
  beneficiary:  beneficiaryAddress,      // from checkout session
  ref:          ref,                     // from Step 6
  permitParams: permitParams,            // from Step 4
};
```

---

## Step 8 — Sign the Operation (Binding Signature)

Produce the Binding Signature using EIP-712 typed data signing against the **Settlement Contract's** domain separator.

```javascript
const settlementDomain = {
  name:              "StablecoinStack",
  version:           "1",
  chainId:           chainId,
  verifyingContract: settlementContractAddress,
};

const payWithPermitTypes = {
  PayWithPermitParams: [
    { name: "token",        type: "address" },
    { name: "beneficiary",  type: "address" },
    { name: "ref",          type: "bytes32" },
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
  payWithPermitTypes,
  payWithPermitParams
);

const payWithPermitSig = { hash, v, r, s };
```

---

## Step 9 — Generate the `payloadId`

Generate a UUID to uniquely identify this submission. Store it locally for idempotency tracking.

```javascript
const payloadId = crypto.randomUUID(); // or use a UUID library
```

---

## Step 10 — Assemble and Submit the Payload

```javascript
const transferRequest = {
  payWithPermitParams,
  payWithPermitSig,
  permitSig,
  payloadId,
};

const response = await fetch("https://processor.example.com/v1/transfer", {
  method:  "POST",
  headers: { "Content-Type": "application/json" },
  body:    JSON.stringify(transferRequest),
});
```

The request MUST be sent over TLS 1.2 or higher.

---

## Step 11 — Handle the Response

| HTTP Status | Meaning                                                                              |
| ----------- | ------------------------------------------------------------------------------------ |
| `200`       | Payload accepted and transaction broadcast. Monitor the session for on-chain confirmation. |
| `400`       | `STRUCTURAL_ERROR` or `SEMANTIC_ERROR` — check the error body for the violated rule. |
| `422`       | `CRYPTOGRAPHIC_ERROR` — signature verification failed.                               |
| `502`       | `BROADCAST_ERROR` — the on-chain transaction failed. Session is NOT marked as settled. |

On a `SEMANTIC_ERROR` with a nonce mismatch, re-fetch the nonce (Step 2) and retry from Step 4.

---

## Checklist

- [ ] `permitParams.value` equals `principal + calculateFees(principal, acquirerId)`
- [ ] `permitParams.nonce` is the current on-chain nonce
- [ ] `permitParams.deadline` is in the future with sufficient margin for on-chain inclusion
- [ ] `ref` is 32 bytes: `orderRef (16) || acquirerId (16)`, zero-padded where absent
- [ ] `permitSig` was signed against the **token contract** domain
- [ ] `payWithPermitSig` was signed against the **Settlement Contract** domain
- [ ] `v` values are `27` or `28`
- [ ] `payloadId` is stored locally before submission
- [ ] Request is sent over HTTPS

---

## Related Documents

- [Permit-Based Payments](../core-concepts/permit-based-payments.md) — conceptual background
- [Fee Model](../../ssf-spec-002/core-concepts/fee-model.md) — how fees are calculated
- [Payload Fields Reference](../reference/payload-fields.md) — field-level constraints
- [SSF-SPEC-001, Section 8.1](../specifications/ssf-spec-001.md#81-payment-transfer--step-by-step) — normative step-by-step
