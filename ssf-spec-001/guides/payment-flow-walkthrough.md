---
document: SSF-SPEC-001/guides/payment-flow-walkthrough.md
spec: SSF-SPEC-001
version: 1.0.0
status: Draft
date: 2026-03-04
author: "Adalton Reis <reis@stablecoinstack.org>"
organization: Stablecoin Stack Foundation
license: Apache License 2.0
---

# Payment Flow Walkthrough

A developer-oriented walkthrough of the complete end-to-end payment flow, tracing each action through the components of the Stablecoin Stack.

**Prerequisites:** read [System Architecture](../core-concepts/system-architecture.md) and [Permit-Based Payments](../core-concepts/permit-based-payments.md) first.

---

## The Flow at a Glance

```
Merchant Server
  │  1. POST /charges  (mTLS)
  ▼
core-checkout-engine  ──► generates session, ref, deadline
  │  2. returns Widget URL + Ephemeral Token
  ▼
Merchant presents QR / deep-link to Payer
  │
  ▼
Client Wallet
  │  3. redeems Ephemeral Token → session params
  │  4. fetches nonce + fees
  │  5. constructs + signs PermitParams       (Permit Signature)
  │  6. constructs + signs PayWithPermitParams (Binding Signature)
  │  7. POST /submit  +  WebSocket connection
  ▼
wallet-gateway → broadcast-service → broadcast-submitter
                                           │  8. transferWithPermit()
                                           ▼
                                   Settlement Contract
                                           │  9. permit() + transferFrom()
                                           │     fee distribution
                                           │     PermittedTransfer event
                                           ▼
                                   transfer-history
                                           │  10. confirmed event (N blocks)
                                           ▼
                                   core-checkout-engine  ──► marks charge settled
                                           │  11. webhook to Merchant Server
                                           ▼
                                   balance-and-history   ──► WebSocket to Wallet
```

---

## Step by Step

### 1. Merchant creates a charge

The Merchant Server sends a charge creation request to the core-checkout-engine over a mutually authenticated TLS session:

```
POST /api/v1/charges
{
  "amount": "100.00",
  "token": "0xTokenContractAddress",
  "beneficiary": "0xMerchantAddress",
  "currency": "USDC",
  "orderId": "order-abc-123"
}
```

The core-checkout-engine generates:
- a 16-byte **Order Reference** (UUID)
- a **Payload ID**
- a `deadline` timestamp (e.g. `now + 15 minutes`)

A session is created in the `awaiting-payment` state.

### 2. Merchant receives the Ephemeral Token

The core-checkout-engine returns:
```json
{
  "widgetUrl": "https://pay.processor.example/s/abc123",
  "ephemeralToken": "eyJ..."
}
```

The merchant presents the widget URL to the payer — as a QR code, deep-link, or redirect.

### 3. Wallet redeems the Ephemeral Token

The wallet opens the widget URL and exchanges the Ephemeral Token for session parameters. The token is invalidated upon first use.

```json
{
  "token": "0xTokenContractAddress",
  "beneficiary": "0xMerchantAddress",
  "principal": "100000000",
  "acquirerId": "0x1234567890abcdef1234567890abcdef",
  "orderReference": "0xdeadbeef...",
  "deadline": 1743000000
}
```

### 4. Wallet fetches nonce and fees

```javascript
// From wallet-gateway or directly on-chain
const nonce = await walletGateway.getNonce(ownerAddress, tokenAddress);
const totalFee = await walletGateway.calculateFees(principal, acquirerId);
const permitValue = principal + totalFee;
```

### 5. Wallet signs the Permit (Permit Signature)

```javascript
const permitParams = {
  owner: ownerAddress,
  spender: settlementContractAddress,
  value: permitValue,
  nonce: nonce,
  deadline: deadline,
};

// Sign against TOKEN contract domain separator
const permitSig = await signTypedData(tokenDomain, permitTypes, permitParams);
```

### 6. Wallet constructs the `ref` and signs the operation (Binding Signature)

```javascript
// Concatenate 16-byte orderReference + 16-byte acquirerId
const ref = "0x"
  + orderReference.replace("0x", "")
  + acquirerId.replace("0x", "");

const payWithPermitParams = {
  token: tokenAddress,
  beneficiary: beneficiaryAddress,
  ref: ref,
  permitParams: permitParams,
};

// Sign against SETTLEMENT CONTRACT domain separator
const payWithPermitSig = await signTypedData(settlementDomain, payWithPermitTypes, payWithPermitParams);
```

### 7. Wallet submits the payload

```javascript
const transferRequest = {
  payWithPermitParams,
  payWithPermitSig,
  permitSig,
  payloadId: crypto.randomUUID(),
};

// HTTPS POST + open WebSocket for status updates
await walletGateway.submit(transferRequest);
walletGateway.on("status", handleStatusUpdate);
```

### 8. Processor validates and broadcasts

The broadcast-service validates the payload (structural, semantic, cryptographic checks per Section 14 of the spec). On success, broadcast-submitter calls `transferWithPermit` on the Settlement Contract.

The wallet receives a WebSocket status update: `BROADCAST_SUBMITTED`. **This is not final settlement.**

### 9. Settlement Contract executes

```
permit()       → token contract grants allowance to Settlement Contract
transferFrom() → tokens move from payer to Settlement Contract
                 fees distributed to processor + acquirer (if applicable)
                 principal credited to beneficiary
usedHashes[digest] = true
emit PermittedTransfer(...)
emit CommissionGenerated(...)  // if acquirer
```

All operations are atomic. Either all succeed or all revert.

### 10. transfer-history confirms the event

After N block confirmations (configurable), transfer-history publishes the confirmed `PermittedTransfer` event to the core-checkout-engine. The `orderReference` in the event's `ref` field is matched against open sessions.

### 11. Final confirmation delivered

The core-checkout-engine marks the charge as `settled` and dispatches a webhook to the Merchant Server. The balance-and-history service delivers a `PAYMENT_CONFIRMED` notification to the wallet over WebSocket.

---

## Key Points to Remember

- **Two domains, two signatures.** The Permit Signature uses the token contract's domain. The Binding Signature uses the Settlement Contract's domain. Never mix them.
- **`permitValue` = principal + fees.** Never sign the permit for just the principal.
- **`ref` is always 32 bytes.** Always present, always zero-padded where parts are absent.
- **Broadcast ≠ settlement.** Wait for the WebSocket `PAYMENT_CONFIRMED` before treating a payment as final.
- **The Ephemeral Token is single-use.** Attempting to reuse it will fail.

---

## Related Documents

- [Submit a Payment](./submit-a-payment.md) — implementation guide with full code
- [Permit-Based Payments](../core-concepts/permit-based-payments.md) — conceptual background
- [Dual-Signature Pattern](../core-concepts/dual-signature-pattern.md) — why two signatures
- [Payment Flow](../core-concepts/payment-flow.md) — narrative description
- [Formal Specification, Section 16](../specifications/ssf-spec-001.md#16-end-to-end-payment-flow) — normative flow
