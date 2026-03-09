# The Payment Flow

This document describes the complete lifecycle of a stablecoin payment through the Stablecoin Stack, from the moment a merchant creates a charge to the moment the session is reconciled and the merchant is notified. It is written as a narrative to build conceptual understanding. For a developer-oriented step-by-step walkthrough, see [Payment Flow Walkthrough](../guides/payment-flow-walkthrough.md).

---

## Overview

A payment in the Stablecoin Stack passes through five distinct phases. Each phase has a clear owner and a clear handoff to the next.

| Phase | Name | Primary Actors |
|-------|------|----------------|
| A | Session Creation | Merchant, Checkout Engine |
| B | Payload Construction and Signing | Client Wallet, Checkout Widget |
| C | Submission and Off-Chain Validation | Wallet, Broadcast Layer |
| D | On-Chain Settlement | Settlement Contract, Relayer |
| E | Confirmation and Reconciliation | transfer-history, Checkout Engine, Merchant |

---

## Phase A — Session Creation

The flow begins on the merchant's side. When a customer initiates a purchase, the Merchant Server sends a **charge creation request** to the core-checkout-engine over a mutually authenticated TLS session. This request contains the payment amount, the accepted token, the merchant's beneficiary address, and any order metadata.

The core-checkout-engine responds by creating a payment session. It generates three critical values:

- an **Order Reference** (`ref`) — a 32-byte value that will travel from here all the way to the on-chain event, serving as the immutable link between the off-chain session and the on-chain transaction;
- a **Payload ID** — an off-chain session identifier used to correlate the submission with the correct charge; and
- a **deadline** — a Unix timestamp after which the payment will no longer be accepted.

The session is persisted in the awaiting-payment state. The core-checkout-engine returns to the Merchant Server a **Checkout Widget URL** and an **Ephemeral Token**. The merchant presents these to the customer — typically as a QR code, a mobile deep-link, or a browser redirect.

At this point, no cryptographic commitment has been made. No funds have moved. The session is simply an expectation.

---

## Phase B — Payload Construction and Signing

This phase is where the payer's intent becomes a cryptographic commitment.

The Client Wallet redeems the Ephemeral Token against the **checkout-public-widget** to retrieve the session payment parameters: the token address, the beneficiary address, the amount, the `ref`, and the `deadline`. The Ephemeral Token is single-use and expires shortly; its purpose is to carry the payment details to the wallet securely without persisting them unnecessarily.

With these parameters in hand, the wallet performs two signing operations, in order.

**First: the Permit Signature.** The wallet reads the payer's current ERC-2612 permit nonce from the token contract on-chain. It then constructs `PermitParams` — specifying the owner (payer), spender (Settlement Contract), value, nonce, and deadline — and signs this structure using EIP-712 to produce the **Permit Signature**. This signature authorises the Settlement Contract to call `permit()` on the token contract, granting itself an allowance to pull the specified amount from the payer. It is validated by the token contract, not by the Settlement Contract.

**Second: the Binding Signature.** The wallet constructs `PayWithPermitParams` — specifying the token, beneficiary, `ref`, and the `PermitParams` from the previous step — and signs this structure using EIP-712 to produce the **Binding Signature**. This signature authorises the processor to submit this specific operation. It binds the entire set of payment parameters together: altering any one of them — the amount, the recipient, the token, the deadline, the order reference — invalidates the signature.

Neither signature alone is sufficient to execute a payment. Both are required, and both are verified independently. This dual-signature pattern is central to the security model.

The payer has now produced a complete, self-contained payment commitment. Their private key is not needed again.

---

## Phase C — Submission and Off-Chain Validation

The wallet assembles the two signatures and the payment parameters into a `TransferRequest` payload and submits it to the **broadcast-gateway** over HTTPS. A WebSocket connection is established simultaneously, over which the wallet will receive real-time status updates for the duration of the submission process.

The broadcast-service enqueues the submission and the Payment Processor begins off-chain validation. This validation is not a courtesy check — it is a defence against reverting on-chain, which would waste gas and degrade the user experience. The processor validates:

- **Structural validity** — all required fields present, hex strings correctly formatted and of the right length;
- **Semantic validity** — the deadline has not passed, the nonce matches the current on-chain value, the payloadId corresponds to a known unprocessed session, and the `ref` matches the session's stored value; and
- **Cryptographic validity** — the Binding Signature recovers to the expected signer (the payer), and the recomputed digest matches the `hash` field in the signature object.

Any validation failure results in immediate rejection with a categorised error. No on-chain activity occurs.

Upon successful validation, the submission enters the broadcast queue.

---

## Phase D — On-Chain Settlement

The **broadcast-submitter** — the component that holds the Relayer's funded account — calls `transferWithPermit` on the Settlement Contract, paying gas from its own balance. The payer's account is not involved in this transaction at the network level.

Inside the Settlement Contract, everything happens atomically:

1. The Binding Signature digest is checked against `usedHashes`. If it has been seen before, the transaction reverts immediately.
2. The Binding Signature is verified. If it does not recover to the authorised signer, the transaction reverts.
3. `permit()` is called on the token contract with the Permit Signature, granting the Settlement Contract an allowance.
4. `transferFrom` collects the full amount (principal plus fees) from the payer's wallet.
5. The fee is computed and distributed: the base fee goes to the processor's fee recipient; if a non-zero Acquirer ID was included, the acquiring fee is credited to the acquirer's on-chain balance.
6. The principal amount is transferred to the beneficiary.
7. The Binding Signature digest is recorded in `usedHashes`, permanently preventing replay.
8. The `PermittedTransfer` event is emitted, carrying the `ref`, the token, the payer, the beneficiary, the value, and the fee. If an acquiring commission was generated, `CommissionGenerated` is also emitted.

If any step fails, the entire transaction reverts. There is no partial settlement.

The broadcast-service receives the transaction hash and updates the submission status: the transaction has been broadcast without revert. This update is delivered to the wallet over WebSocket. **This is not final settlement.** It means only that the network accepted the transaction. Finality requires confirmation.

---

## Phase E — Confirmation and Reconciliation

The **transfer-history** service monitors the Settlement Contract for `PermittedTransfer` events. When it observes the event, it waits for a configurable number of block confirmations to elapse — the point at which the transaction is considered sufficiently deep in the chain to be treated as irreversible.

Once the confirmation threshold is reached, transfer-history publishes the confirmed event to the **core-checkout-engine**. The engine matches the event's `orderReference` field (the `ref`) against its session registry and marks the corresponding charge as **settled**.

Two notifications flow from this reconciliation:

- The core-checkout-engine dispatches a **settlement webhook** to the Merchant Server, carrying the final status and the on-chain reference. The merchant's system can now fulfil the order.
- The **balance-and-history** service detects the confirmed transfer and delivers a **final settlement notification** to the wallet over WebSocket. The user sees their payment confirmed.

The `ref` is the thread that connects all five phases. It is generated by the Checkout Engine in Phase A, embedded in the signed payload in Phase B, validated by the processor in Phase C, emitted by the Settlement Contract in Phase D, and used for reconciliation in Phase E. Its integrity is protected by the Binding Signature: a `ref` cannot be changed after signing without invalidating the signature.

---

## Submission Completion vs. Transaction Finality

These two concepts are distinct and MUST NOT be conflated.

**Submission completion** means the Relayer submitted the transaction and did not receive a revert. The transaction exists in the mempool or has been included in a block. It does not mean the transaction is permanent.

**Transaction finality** means the transaction has been included in a block that is sufficiently deep in the chain that reorganisation is considered negligible. This is confirmed by transfer-history after observing the required number of block confirmations.

Settlement reconciliation and merchant notification occur only upon finality, not upon submission completion.
