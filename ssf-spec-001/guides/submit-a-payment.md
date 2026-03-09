---
document: SSF-SPEC-001/guides/submit-a-payment.md
spec: SSF-SPEC-001
version: 1.0.0
status: Draft
date: 2026-03-04
author: "Adalton Reis <reis@stablecoinstack.org>"
organization: Stablecoin Stack Foundation
license: Apache License 2.0
---

# Submit a Payment

Step-by-step guide for wallet implementers. Covers the complete construction and submission of a `TransferRequest` payload.

**Prerequisites:** read [Permit-Based Payments](../core-concepts/permit-based-payments.md) and [Dual-Signature Pattern](../core-concepts/dual-signature-pattern.md) first.

---

## Before You Start

You will need:
- the payer's private key (held locally — never transmitted)
- the wallet-gateway base URL for your processor
- the Settlement Contract address (from the basic-data-server)
- the session parameters (retrieved via Ephemeral Token)

---

## Step 1 — Retrieve Session Parameters

Exchange the Ephemeral Token for payment parameters:

```javascript
const session = await fetch(`${widgetUrl}/api/session`, {
  headers: { Authorization: `Bearer ${ephemeralToken}` },
}).then(r => r.json());

// session contains:
// {
//   token: "0x...",           ERC-2612 stablecoin address
//   beneficiary: "0x...",     merchant receiving address
//   principal: "100000000",   amount in smallest token units
//   orderReference: "0x...",  16-byte Order Reference (32-char hex)
//   acquirerId: "0x...",      16-byte Acquirer ID (32-char hex), or Zero-UUID
//   deadline: 1743000000,     Unix timestamp (seconds)
// }
```

The Ephemeral Token is now invalid. Do not retry this call.

---

## Step 2 — Fetch Nonce and Fees

```javascript
// Option A: via wallet-gateway (recommended — no direct on-chain call needed)
const { nonce } = await walletGateway.getNonce(ownerAddress, session.token);
const { totalFee } = await walletGateway.calculateFees(
  session.principal,
  session.acquirerId
);

// Option B: direct on-chain calls
const nonce = await tokenContract.nonces(ownerAddress);
const totalFee = await settlementContract.calculateFees(
  session.principal,
  session.acquirerId
);

const permitValue = BigInt(session.principal) + BigInt(totalFee);
```

`permitValue` is the total the payer must hold. This is the amount the permit will authorise.

---

## Step 3 — Build `PermitParams`

```javascript
const permitParams = {
  owner:    ownerAddress,                   // payer's address
  spender:  settlementContractAddress,      // Settlement Contract
  value:    permitValue.toString(),         // total with fees
  nonce:    nonce.toString(),
  deadline: session.deadline,
};
```

---

## Step 4 — Sign the Permit (Permit Signature)

Sign against the **token contract's** EIP-712 domain separator.

```javascript
// EIP-712 domain for the TOKEN contract
const tokenDomain = {
  name:              await tokenContract.name(),
  version:           "1",
  chainId:           chainId,
  verifyingContract: session.token,
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

// Compute the EIP-712 digest to include in the ERC20RelayerSig object
const permitHash = ethers.TypedDataEncoder.hash(tokenDomain, permitTypes, permitParams);

const permitSigObj = {
  hash: permitHash,
  v: v1 < 27 ? v1 + 27 : v1,   // normalise 0/1 → 27/28
  r: r1,
  s: s1,
};
```

---

## Step 5 — Build `ref`

The `ref` field is always 32 bytes: the 16-byte Order Reference followed by the 16-byte Acquirer ID.

```javascript
// Both orderReference and acquirerId are already 16-byte hex values
// Strip the 0x prefix, concatenate, re-add prefix
const orderRef = session.orderReference.replace("0x", "");  // 32 hex chars
const acquirerId = session.acquirerId.replace("0x", "");    // 32 hex chars

// Validate lengths
if (orderRef.length !== 32) throw new Error("orderReference must be 16 bytes");
if (acquirerId.length !== 32) throw new Error("acquirerId must be 16 bytes");

const ref = "0x" + orderRef + acquirerId;   // 66-char hex string (32 bytes)
```

If no Order Reference: use `"0".repeat(32)`. If no Acquirer: use the Zero-UUID `"0".repeat(32)`.

---

## Step 6 — Build `PayWithPermitParams`

```javascript
const payWithPermitParams = {
  token:        session.token,
  beneficiary:  session.beneficiary,
  ref:          ref,
  permitParams: permitParams,
};
```

---

## Step 7 — Sign the Operation (Binding Signature)

Sign against the **Settlement Contract's** EIP-712 domain separator.

```javascript
// EIP-712 domain for the SETTLEMENT CONTRACT
const settlementDomain = {
  name:              "SettlementContract",   // verify exact name from contract
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

const payWithPermitSig = await signer.signTypedData(
  settlementDomain,
  payWithPermitTypes,
  payWithPermitParams
);
const { v: v2, r: r2, s: s2 } = ethers.Signature.from(payWithPermitSig);

const bindingHash = ethers.TypedDataEncoder.hash(
  settlementDomain,
  payWithPermitTypes,
  payWithPermitParams
);

const payWithPermitSigObj = {
  hash: bindingHash,
  v: v2 < 27 ? v2 + 27 : v2,
  r: r2,
  s: s2,
};
```

---

## Step 8 — Assemble and Submit the `TransferRequest`

```javascript
const payloadId = crypto.randomUUID();

const transferRequest = {
  payWithPermitParams:  payWithPermitParams,
  payWithPermitSig:     payWithPermitSigObj,
  permitSig:            permitSigObj,
  payloadId:            payloadId,
};

// Open WebSocket for real-time status updates
const ws = new WebSocket(`${walletGatewayWsUrl}?session=${payloadId}`);
ws.onmessage = (event) => handleStatusUpdate(JSON.parse(event.data));

// Submit the payload
const response = await fetch(`${walletGatewayUrl}/api/submit`, {
  method: "POST",
  headers: { "Content-Type": "application/json" },
  body: JSON.stringify(transferRequest),
});

if (!response.ok) {
  const error = await response.json();
  // error.category: STRUCTURAL_ERROR | SEMANTIC_ERROR | CRYPTOGRAPHIC_ERROR
  throw new Error(`Submission rejected: ${error.category} — ${error.message}`);
}
```

---

## Step 9 — Handle Status Updates

```javascript
function handleStatusUpdate(update) {
  switch (update.status) {
    case "QUEUED":
      // Payload received and queued for validation
      showToUser("Processing your payment…");
      break;

    case "BROADCAST_SUBMITTED":
      // Relayer submitted the transaction — NOT final settlement
      showToUser("Transaction submitted, confirming on-chain…");
      break;

    case "PAYMENT_CONFIRMED":
      // transfer-history confirmed sufficient block depth — FINAL
      showToUser("Payment confirmed!");
      navigateToReceipt(update.txHash);
      break;

    case "FAILED":
      showToUser(`Payment failed: ${update.reason}`);
      break;
  }
}
```

`BROADCAST_SUBMITTED` means the Relayer did not receive a revert. It does not mean the payment has settled. Wait for `PAYMENT_CONFIRMED` before treating the payment as final.

---

## Common Mistakes

| Mistake | Result | Fix |
| ------- | ------ | --- |
| Signing permit for principal only (without fees) | On-chain revert — `transferFrom` amount exceeds allowance | Always use `calculateFees` and add fees to `permitValue` |
| Mixing up domain separators | Invalid signature | Token domain for Permit Signature; Settlement Contract domain for Binding Signature |
| Wrong `v` value (`0` or `1`) | Processor rejection | Normalise: if `v < 27`, add `27` |
| Omitting `ref` or wrong length | `STRUCTURAL_ERROR` | Always `0x` + 64 hex chars; zero-pad missing parts |
| Treating `BROADCAST_SUBMITTED` as final | Premature confirmation shown to user | Wait for `PAYMENT_CONFIRMED` |

---

## Related Documents

- [Payload Fields Reference](../reference/payload-fields.md) — complete field-level spec
- [Fee Model](../core-concepts/fee-model.md) — fee calculation detail
- [Dual-Signature Pattern](../core-concepts/dual-signature-pattern.md) — why two signatures, two domains
- [Formal Specification, Section 15.1](../specifications/ssf-spec-001.md#151-payment-transfer--step-by-step) — normative submission flow
