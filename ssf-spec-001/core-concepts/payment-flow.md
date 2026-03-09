---
document: SSF-SPEC-001/core-concepts/payment-flow.md
spec: SSF-SPEC-001
version: 1.0.0
status: Draft
date: 2026-03-04
author: "Adalton Reis <reis@stablecoinstack.org>"
organization: Stablecoin Stack Foundation
license: Apache License 2.0
---

# Payment Flow

This document describes the complete end-to-end payment flow — from charge creation by the merchant to final settlement confirmation delivered to both merchant and payer. It is written as a narrative to build intuition. Normative requirements for each step are in the [formal specification](../specifications/ssf-spec-001.md).

---

## Overview

A payment moves through five phases:

```
Phase A — Session Creation
Phase B — Payload Construction and Signing
Phase C — Submission and Off-Chain Validation
Phase D — On-Chain Settlement
Phase E — Confirmation and Reconciliation
```

The payer's active involvement spans only Phase B and the submission in Phase C — two signature operations and one HTTPS request. Everything else is infrastructure.

---

## Phase A — Session Creation

The Merchant Server sends a charge creation request to the **core-checkout-engine** over a mutually authenticated TLS session. The request includes the payment amount, the accepted token, the merchant's beneficiary address, and any order metadata.

The core-checkout-engine generates a unique 16-byte **Order Reference**, a **Payload ID**, and a `deadline` timestamp. A new charge session is created in the awaiting-payment state.

The core-checkout-engine returns a Checkout Widget URL and an **Ephemeral Token** to the Merchant Server. The merchant presents this to the payer — typically as a QR code, deep-link, or redirect.

---

## Phase B — Payload Construction and Signing

The **Client Wallet** redeems the Ephemeral Token against the **checkout-public-widget** to retrieve the payment parameters: token address, beneficiary address, amount, Order Reference, Acquirer ID, and `deadline`. The Ephemeral Token is invalidated upon first use.

The wallet reads the payer's current ERC-2612 permit nonce from the token contract (or from the wallet-gateway, which provides this as a convenience endpoint).

The wallet calls `calculateFees` on the Settlement Contract (or via the wallet-gateway) to determine the total amount inclusive of fees. This total is the value the permit must be signed for.

The wallet constructs `PermitParams` and signs the ERC-2612 Permit typed-data using EIP-712 against the **token contract's domain separator**, producing the **Permit Signature**.

The wallet concatenates the 16-byte Order Reference with the 16-byte Acquirer ID to form the 32-byte `ref` field. Where either is absent, zeros are used.

The wallet constructs `PayWithPermitParams` and signs it using EIP-712 against the **Settlement Contract's domain separator**, producing the **Binding Signature**.

---

## Phase C — Submission and Off-Chain Validation

The wallet assembles the complete `TransferRequest` payload — parameters, Binding Signature, Permit Signature, and Payload ID — and submits it to the **wallet-gateway** over HTTPS. A WebSocket connection is maintained for real-time status updates.

The **broadcast-service** enqueues the submission. The Payment Processor validates the payload: structural checks (field encoding, lengths), semantic checks (deadline not expired, nonce matches on-chain state, Payload ID known and unprocessed), and cryptographic checks (recovered signer matches owner, hash fields match recomputed digests).

Any validation failure results in immediate rejection with a categorised error. No on-chain activity occurs on rejection.

---

## Phase D — On-Chain Settlement

The **broadcast-submitter** (Relayer) broadcasts the transaction by calling `transferWithPermit` on the Settlement Contract, paying gas from its own funded account.

The Settlement Contract verifies the Binding Signature and checks that its hash is not present in `usedHashes`. It calls `permit()` on the token contract using the Permit Signature, granting itself an allowance. It then calls `transferFrom` to collect the full amount from the payer.

The contract computes and distributes fees, credits the principal to the beneficiary, records the Binding Signature hash in `usedHashes`, and emits `PermittedTransfer` (and `CommissionGenerated` if an Acquirer was involved). All of this is atomic within a single transaction — it either completes entirely or reverts entirely.

The broadcast-service delivers a status update to the wallet over WebSocket confirming the transaction was broadcast without revert. **This is not final settlement** — it is confirmation that the transaction reached the network.

---

## Phase E — Confirmation and Reconciliation

The **transfer-history** service monitors the Settlement Contract for `PermittedTransfer` events. After a configurable number of block confirmations, it publishes the confirmed event to the core-checkout-engine.

The core-checkout-engine matches the event's `orderReference` field against its session registry using the 16-byte Order Reference embedded in the first half of the `ref` field, and marks the corresponding charge as settled.

The core-checkout-engine dispatches a settlement webhook to the **Merchant Server**, carrying the final status and on-chain reference.

The **balance-and-history** service detects the confirmed transfer and delivers a final settlement notification to the wallet over WebSocket.

---

## Key Distinctions

**Broadcast ≠ Settlement.** The broadcast-service confirms that the Relayer received no revert when submitting the transaction. Final settlement is confirmed only by transfer-history after sufficient block confirmations.

**Submission identity is irrelevant.** The Settlement Contract does not validate `msg.sender`. Any party in possession of both valid signatures could submit the transaction. The payer's protection comes from the signature binding, not from submission control.

**The Ephemeral Token has no signing authority.** Intercepting an Ephemeral Token allows a party to read session parameters — it does not enable payment. The payer's private key is still required for both signatures.

---

## Related Documents

- [System Architecture](./system-architecture.md) — component roles
- [Permit-Based Payments](./permit-based-payments.md) — how gasless transfers work
- [Dual-Signature Pattern](./dual-signature-pattern.md) — the two-signature model
- [Submit a Payment](../guides/submit-a-payment.md) — implementation guide
- [Formal Specification, Section 10](../specifications/ssf-spec-001.md#10-end-to-end-payment-flow) — normative flow
