---
document: SSF-SPEC-001/guides/buy-acquiring-pack.md
spec: SSF-SPEC-001
version: 1.0.0
status: Draft
date: 2026-03-04
author: "Adalton Reis <reis@stablecoinstack.org>"
organization: Stablecoin Stack Foundation
license: Apache License 2.0
---

# Buy an Acquiring Pack

Step-by-step guide for registering as an Acquirer via the `BuyAcquiringPackRequest` submission flow.

**Prerequisites:** read [Fee Model](../core-concepts/fee-model.md) and [Dual-Signature Pattern](../core-concepts/dual-signature-pattern.md) first.

---

## What You're Doing

Registering as an Acquirer requires:
1. Paying a one-time registration fee (`acquiringPrice`) to the Settlement Contract.
2. Choosing an acquiring fee percentage (subject to `maxAcquiringFee`).
3. Providing the wallet address to be registered on-chain.

The payer of the fee and the wallet being registered do not need to be the same address — sponsored registration is supported.

On success, the Settlement Contract assigns a new `bytes16` **Acquirer ID** and emits `AcquirerCreated`.

---

## Step 1 — Fetch Registration Parameters

```javascript
// From the Settlement Contract directly or via wallet-gateway
const acquiringPrice  = await settlementContract.acquiringPrice();
const maxAcquiringFee = await settlementContract.maxAcquiringFee();
const allowedTokens   = await walletGateway.getAcquiringAllowedTokens();
// Choose a token from allowedTokens
const token = allowedTokens[0];

// The fee percentage you want to charge payers (≤ maxAcquiringFee)
const desiredFeePercent = 100;   // processor-defined units — verify meaning with your processor
```

---

## Step 2 — Fetch Nonce

```javascript
const nonce = await tokenContract.nonces(payerAddress);
```

---

## Step 3 — Build `PermitParams`

The permit value MUST equal `acquiringPrice` exactly.

```javascript
const permitParams = {
  owner:    payerAddress,
  spender:  settlementContractAddress,
  value:    acquiringPrice.toString(),   // must equal acquiringPrice exactly
  nonce:    nonce.toString(),
  deadline: Math.floor(Date.now() / 1000) + 900,   // 15 minutes from now
};
```

---

## Step 4 — Sign the Permit (Permit Signature)

Sign against the **token contract's** EIP-712 domain separator.

```javascript
const tokenDomain = {
  name:              await tokenContract.name(),
  version:           "1",
  chainId:           chainId,
  verifyingContract: token,
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

const permitSig = await signer.signTypedData(tokenDomain, permitTypes, permitParams);
const { v: v1, r: r1, s: s1 } = ethers.Signature.from(permitSig);
const permitHash = ethers.TypedDataEncoder.hash(tokenDomain, permitTypes, permitParams);

const permitSigObj = {
  hash: permitHash,
  v:    v1 < 27 ? v1 + 27 : v1,
  r:    r1,
  s:    s1,
};
```

---

## Step 5 — Build `BuyAcquiringPackPermitParams`

```javascript
const buyAcquiringPackParams = {
  token:        token,
  feeValue:     acquiringPrice.toString(),   // must equal permitParams.value
  acquiring:    acquiringWalletAddress,      // wallet to register (may differ from payer)
  permitParams: permitParams,
};
```

---

## Step 6 — Sign the Operation (Binding Signature)

Sign against the **Settlement Contract's** EIP-712 domain separator.

```javascript
const settlementDomain = {
  name:              "SettlementContract",
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

const payWithPermitSig = await signer.signTypedData(
  settlementDomain,
  buyAcquiringPackTypes,
  buyAcquiringPackParams
);
const { v: v2, r: r2, s: s2 } = ethers.Signature.from(payWithPermitSig);
const bindingHash = ethers.TypedDataEncoder.hash(
  settlementDomain,
  buyAcquiringPackTypes,
  buyAcquiringPackParams
);

const payWithPermitSigObj = {
  hash: bindingHash,
  v:    v2 < 27 ? v2 + 27 : v2,
  r:    r2,
  s:    s2,
};
```

---

## Step 7 — Assemble and Submit the `BuyAcquiringPackRequest`

```javascript
const buyAcquiringPackRequest = {
  buyAcquiringPackParams: buyAcquiringPackParams,
  payWithPermitSig:       payWithPermitSigObj,
  permitSig:              permitSigObj,
};

const response = await fetch(`${walletGatewayUrl}/api/submit-acquiring`, {
  method: "POST",
  headers: { "Content-Type": "application/json" },
  body: JSON.stringify(buyAcquiringPackRequest),
});

if (!response.ok) {
  const error = await response.json();
  throw new Error(`Rejected: ${error.category} — ${error.message}`);
}
```

---

## Step 8 — Retrieve Your Acquirer ID

After on-chain confirmation, your Acquirer ID is emitted in the `AcquirerCreated` event. You can also look it up:

```javascript
// Listen for the AcquirerCreated event
settlementContract.on("AcquirerCreated", (acquirerId, wallet, feePercent) => {
  if (wallet.toLowerCase() === acquiringWalletAddress.toLowerCase()) {
    console.log("Registered! Acquirer ID:", acquirerId);
  }
});

// Or look up the reverse mapping (if exposed by your processor)
const acquirerId = await walletGateway.getAcquirerId(acquiringWalletAddress);
```

Store your Acquirer ID. You will distribute it to payers so they can include it in their `ref` field.

---

## Becoming a Service Provider

If you want to be publicly discoverable by wallets, submit your information to the basic-data-server to appear as a **Service Provider**. This is optional and opt-in.

---

## Common Mistakes

| Mistake | Result | Fix |
| ------- | ------ | --- |
| `feeValue` ≠ `permitParams.value` | `SEMANTIC_ERROR` | Both must equal current `acquiringPrice` exactly |
| `feePercent` exceeds `maxAcquiringFee` | On-chain revert | Fetch `maxAcquiringFee` first; set at or below |
| Trying to register an already-registered wallet | On-chain revert | Each wallet can only be registered once |
| Using a token not in `acquiringAllowedTokens` | On-chain revert | Fetch the allowed list first |
| Wrong `v` value | Processor rejection | Normalise: if `v < 27`, add `27` |

---

## Related Documents

- [Fee Model](../core-concepts/fee-model.md) — acquiring fee mechanics
- [Payload Fields Reference](../reference/payload-fields.md) — `BuyAcquiringPackRequest` field spec
- [Contract Interface Reference](../reference/contract-interface.md#buyacquiringpack) — `buyAcquiringPack` function
- [Formal Specification, Section 15.2](../specifications/ssf-spec-001.md#152-acquirer-registration--step-by-step) — normative registration flow
