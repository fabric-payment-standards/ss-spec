---
document: SSF-SPEC-001/reference/component-index.md
spec: SSF-SPEC-001
version: 1.0.0
status: Draft
date: 2026-03-04
author: "Adalton Reis <reis@stablecoinstack.org>"
organization: Stablecoin Stack Foundation
license: Apache License 2.0
---

# Component Index

Authoritative reference for all components of the Stablecoin Stack. For normative requirements see [SSF-SPEC-001, Section 5](../specifications/ssf-spec-001.md#5-component-specifications).

---

## On-Chain

| Component | Subsystem | Spec Section | Key Requirement |
| --------- | --------- | ------------ | --------------- |
| **Settlement Contract** | On-chain | §§10–13 | MUST verify both signatures, execute transfers atomically, distribute fees, register acquirers, and maintain `usedHashes`. MUST be non-custodial with respect to the Administrator. |

---

## Checkout Engine

| Component | Spec Section | Key Requirements |
| --------- | ------------ | ---------------- |
| **core-checkout-engine** | §5.2 | MUST expose merchant API over mTLS. MUST reconcile charges on confirmed `PermittedTransfer` events using the Order Reference in `ref`. |
| **checkout-public-widget** | §5.2 | MUST issue single-use Ephemeral Tokens. MUST deliver session parameters to wallets. Token MUST expire within 5 minutes (RECOMMENDED). |
| **ca-server** | §5.2 | MUST issue and manage mTLS certificates for merchant ↔ core-checkout-engine sessions. |
| **login-server** | §5.2 | MUST issue JWT bearer tokens within authenticated mTLS sessions. |
| **credentials-manager** | §5.2 | MUST manage certificate lifecycle and user administration. |
| **merchant-dashboard** | §5.7 | SHOULD provide real-time order monitoring, fee reporting, and account management. MUST NOT interact with the blockchain directly. |

---

## Broadcast Layer

| Component | Spec Section | Key Requirements |
| --------- | ------------ | ---------------- |
| **wallet-gateway** | §5.3 | MUST accept `TransferRequest` and `BuyAcquiringPackRequest` payloads. MUST maintain persistent WebSocket connections for status updates. SHOULD provide nonce retrieval and fee calculation endpoints. |
| **broadcast-service** | §5.3 | MUST enqueue validated payloads. MUST manage submission state transitions. MUST NOT treat broadcast-without-revert as final settlement. |
| **broadcast-submitter** | §5.3 | MUST be the sole component that issues on-chain transactions. Holds the Relayer's funded account. |
| **balance-and-history** | §5.3 | SHOULD maintain real-time wallet balance views. SHOULD deliver transfer notifications over WebSocket. |

---

## Event Indexing

| Component | Spec Section | Key Requirements |
| --------- | ------------ | ---------------- |
| **transfer-history** | §5.4 | MUST monitor the Settlement Contract for `PermittedTransfer` events. MUST wait for a configurable confirmation depth before publishing to the Checkout Engine. MUST NOT have write authority over other components. |

---

## Shared Infrastructure

| Component | Spec Section | Key Requirements |
| --------- | ------------ | ---------------- |
| **basic-data-server** | §5.5 | MUST be publicly accessible and read-only. MUST serve supported token list, Service Provider registry, and processor configuration. |

---

## Client

| Component | Spec Section | Key Requirements |
| --------- | ------------ | ---------------- |
| **Client Wallet** | §5.6 | MUST construct and sign `PermitParams` and `PayWithPermitParams` using EIP-712. MUST assemble and submit `TransferRequest` and `BuyAcquiringPackRequest`. MUST NOT transmit the payer's private key or seed phrase to any remote service. All signing MUST be performed locally. |

---

## Future Specifications

The following components require dedicated interface specifications, planned as future companion specs to SSF-SPEC-001:

| Component | Planned Spec |
| --------- | ------------ |
| core-checkout-engine (merchant API) | SSF-SPEC-002 |
| wallet-gateway (WebSocket protocol) | SSF-SPEC-003 |
| wallet-gateway (interface) | SSF-SPEC-004 |
| broadcast-service (interface) | SSF-SPEC-005 |
| broadcast-submitter (interface) | SSF-SPEC-006 |
| transfer-history (interface) | SSF-SPEC-007 |
| basic-data-server (interface) | SSF-SPEC-008 |
| Client Wallet (certification) | SSF-SPEC-009 |
