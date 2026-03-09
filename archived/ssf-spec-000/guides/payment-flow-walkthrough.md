# Payment Flow Walkthrough

This guide walks through the complete lifecycle of a stablecoin payment step by step, from the perspective of each actor in the system. It is written for developers building or integrating with any component of the Stablecoin Stack.

For the conceptual narrative, see [The Payment Flow](../core-concepts/payment-flow.md). For the normative payload and signing requirements, see [SSF-SPEC-001](../specifications/SSF-SPEC-000-system-overview.md).

---

## Prerequisites

Before following this walkthrough, ensure you understand:

- the [System Architecture](../core-concepts/system-architecture.md) and the role of each component;
- the [Dual-Signature Pattern](../core-concepts/system-architecture.md#trust-boundaries) (Permit Signature vs. Binding Signature); and
- the distinction between [Submission Completion and Transaction Finality](../core-concepts/payment-flow.md#submission-completion-vs-transaction-finality).

---

## Phase A — Session Creation

**Actor: Merchant Server**

The Merchant Server initiates the payment flow by creating a charge session. It sends a POST request to the core-checkout-engine over a mutually authenticated TLS session.

**What the Merchant Server sends:**
- The payment amount, denominated in the smallest unit of the stablecoin (e.g., USDC uses 6 decimal places: 10.00 USDC = `10000000`).
- The EVM address of the accepted stablecoin token.
- The beneficiary address — the wallet to which the principal should be credited.
- Any order metadata (order ID, item descriptions, etc.).

**What the core-checkout-engine does:**
1. Validates the request (authenticated merchant, valid token, non-zero amount).
2. Generates a **32-byte Order Reference (`ref`)** — a unique identifier for this charge. This value must be globally unique and unpredictable. It will travel all the way to the on-chain event.
3. Generates a **Payload ID** — an off-chain session identifier.
4. Computes a **deadline** — the Unix timestamp after which the payment will no longer be accepted. The processor configures the session window; typical values are 5–15 minutes.
5. Persists the session in the awaiting-payment state.

**What the Merchant Server receives:**
- The Checkout Widget URL — the hosted payment page for this session.
- An Ephemeral Token — a short-lived, single-use credential the payer's wallet will use to retrieve session details.

**What the Merchant Server does with this:**
- Presents the Checkout Widget URL and/or the Ephemeral Token to the payer. Typical methods: QR code, mobile deep-link, browser redirect.

---

## Phase B — Payload Construction and Signing

**Actor: Client Wallet**

This phase is where the payer produces the two cryptographic signatures that constitute the payment commitment.

### Step 1 — Retrieve Session Details

The wallet redeems the Ephemeral Token against the checkout-public-widget. This is a simple HTTPS GET request.

**The wallet sends:**
- The Ephemeral Token (in the request header or as a query parameter, per the processor's API).

**The wallet receives:**
- `token` — the ERC-2612 stablecoin contract address.
- `beneficiary` — the merchant's receiving address.
- `amount` — the total amount the payer must permit (principal + fees). This is the value that goes into the permit.
- `ref` — the 32-byte order reference.
- `deadline` — the permit expiry timestamp.
- `acquirerId` — the `bytes16` Acquirer ID to include (or the Zero-UUID if none).

The Ephemeral Token is invalidated server-side upon this redemption. The wallet MUST check the response before proceeding.

### Step 2 — Read the On-Chain Nonce

The wallet queries the token contract to read the payer's current ERC-2612 permit nonce.

```
nonce = token.nonces(payer_address)
```

This value must be current at the time of signing. If the nonce has changed between reading and submitting (e.g., because another permit was consumed on-chain), the Permit Signature will be rejected by the token contract.

### Step 3 — Construct and Sign the Permit Signature

The wallet constructs `PermitParams`:

| Field | Value |
|-------|-------|
| `owner` | The payer's EVM address |
| `spender` | The Settlement Contract address |
| `value` | The total amount (principal + fees), as received from the widget |
| `nonce` | The on-chain nonce read in Step 2 |
| `deadline` | The deadline received from the widget |

The wallet signs the ERC-2612 Permit typed-data using EIP-712 over these parameters, using the token contract's EIP-712 domain separator. This produces the **Permit Signature** (`v1`, `r1`, `s1`).

> **Important:** The `spender` MUST be the Settlement Contract address, not the processor's wallet. Using the wrong address will result in an on-chain revert.

### Step 4 — Construct and Sign the Binding Signature

The wallet constructs `PayWithPermitParams`:

| Field | Value |
|-------|-------|
| `token` | The stablecoin contract address |
| `beneficiary` | The merchant's receiving address |
| `ref` | The 32-byte order reference |
| `permitParams` | The `PermitParams` object from Step 3 |

The wallet signs this structure using EIP-712 over the Settlement Contract's domain separator. This produces the **Binding Signature** (`v2`, `r2`, `s2`).

> **Important:** The Binding Signature covers all payment parameters. Any modification — changing the amount, beneficiary, token, or ref — will invalidate this signature. The wallet MUST present these parameters to the user for review before signing.

### Step 5 — Assemble the Payload

The wallet assembles the complete `TransferRequest`:

```
TransferRequest {
  payWithPermitParams: {
    token,
    beneficiary,
    ref,
    permitParams: { owner, spender, value, nonce, deadline }
  },
  payWithPermitSig: { hash, v, r, s },   // Binding Signature
  permitSig:        { hash, v, r, s },   // Permit Signature
  payloadId                               // received from checkout session
}
```

The `hash` field in each signature object is the EIP-712 digest over which the signature was computed. The processor will recompute this digest and verify it matches.

---

## Phase C — Submission and Off-Chain Validation

**Actor: Client Wallet → broadcast-gateway**

### Step 6 — Submit the Payload

The wallet sends the `TransferRequest` to the broadcast-gateway as an HTTP POST request over TLS 1.2 or higher. Simultaneously, it establishes a WebSocket connection to receive real-time status updates.

The broadcast-gateway acknowledges receipt and assigns the submission to the broadcast-service queue.

### Step 7 — Off-Chain Validation

The Payment Processor validates the payload before any on-chain activity occurs. Validation failures are returned immediately.

**Structural checks:**
- All required fields present and non-empty.
- All hex strings correctly prefixed (`0x`) and of the correct length.
- `v` values are `27` or `28` (or `0`/`1`, normalised to `27`/`28`).
- Numeric fields within `uint256` range.

**Semantic checks:**
- `deadline` is in the future (with configurable clock-skew tolerance).
- `nonce` matches the current on-chain nonce for the `owner` address.
- `payloadId` corresponds to a known, unprocessed session.
- `ref` matches the stored value for that session.

**Cryptographic checks:**
- The Binding Signature recovers to the expected signer (`owner`).
- The recomputed digest matches the `hash` field in `payWithPermitSig`.
- The processor SHOULD also verify `permitSig` locally against the token contract's domain separator.

---

## Phase D — On-Chain Settlement

**Actor: broadcast-submitter (Relayer)**

### Step 8 — Broadcast the Transaction

The broadcast-submitter calls `transferWithPermit` on the Settlement Contract, passing all parameters and both sets of signature components (`v1`/`r1`/`s1` and `v2`/`r2`/`s2`). Gas is paid from the Relayer's funded account.

### Step 9 — On-Chain Execution

The Settlement Contract executes the following atomically. If any step reverts, all steps revert.

1. Check `usedHashes[bindingDigest]` — revert if already present.
2. Verify the Binding Signature — revert if invalid.
3. Call `token.permit(owner, address(this), value, deadline, v1, r1, s1)`.
4. Call `token.transferFrom(owner, address(this), value)`.
5. Compute and distribute fees (base fee to processor, acquiring fee to acquirer if applicable).
6. Transfer principal to `recipient`.
7. Record `usedHashes[bindingDigest] = true`.
8. Emit `PermittedTransfer(domainSeparator, token, payer, recipient, value, fee, ref)`.
9. If acquiring fee > 0, emit `CommissionGenerated(acquirerId, amount)`.

### Step 10 — Submission Completion Notification

The broadcast-service receives the transaction hash and updates the submission status. The wallet receives a status update over WebSocket: the transaction has been broadcast. **This is not final settlement.** Advise users accordingly.

---

## Phase E — Confirmation and Reconciliation

**Actor: transfer-history, core-checkout-engine, Merchant Server**

### Step 11 — Event Confirmation

The transfer-history service observes the `PermittedTransfer` event on-chain. It waits for the configured number of block confirmations (the specific threshold is an operator configuration decision, balancing speed against reorganisation risk).

### Step 12 — Charge Reconciliation

Upon reaching the confirmation threshold, transfer-history delivers the confirmed event to the core-checkout-engine. The engine:

1. Extracts the `orderReference` (`ref`) from the event.
2. Looks up the corresponding session.
3. Verifies the settled amount and token match the session's expectations.
4. Marks the charge as **settled**.

### Step 13 — Merchant Notification

The core-checkout-engine dispatches a settlement webhook to the Merchant Server, carrying:
- the charge ID and session ref;
- the final settled status;
- the on-chain transaction hash and block number; and
- the settled amount and token.

The Merchant Server can now fulfil the order.

### Step 14 — Wallet Notification

The balance-and-history service detects the confirmed transfer and delivers a final settlement notification to the wallet over WebSocket. The user sees their payment confirmed.

---

## Error Handling Reference

| Error Category | When it occurs | What to do |
|---------------|----------------|-----------|
| `STRUCTURAL_ERROR` | Off-chain, Phase C | Fix the payload construction. Check hex lengths, field presence, and `v` value range. |
| `SEMANTIC_ERROR` | Off-chain, Phase C | Check deadline (is it in the future?), nonce (is it current?), payloadId (is it known?). |
| `CRYPTOGRAPHIC_ERROR` | Off-chain, Phase C | Re-sign. The digest or signer did not match. Ensure the correct domain separator is used for each signature. |
| `BROADCAST_ERROR` | On-chain, Phase D | The transaction reverted on-chain. The session is not settled. Investigate the revert reason. Do not mark the session as complete. |

For the normative definition of each error category, see SSF-SPEC-001 Section 9.
