# Component Index

This document is the authoritative reference for all components that constitute a conformant Stablecoin Stack deployment. Each entry states the component's role, its category, the specification that governs its interface, and the key requirements an implementer must satisfy.

For narrative descriptions of how components interact, see [System Architecture](../core-concepts/system-architecture.md).

---

## On-Chain Components

### Settlement Contract

| Field | Value |
|-------|-------|
| **Category** | On-chain |
| **Governing Specification** | SSF-SPEC-002 |
| **Conformance** | Required for a fully conformant stack |

The Settlement Contract is the on-chain trust anchor of the Stablecoin Stack. It is the only component whose correctness can be verified without trusting any operator. All payment value flows through it.

**Key requirements:**
- MUST verify the Binding Signature before executing any operation.
- MUST check and record all Binding Signature digests in `usedHashes` before any token transfer occurs.
- MUST call `permit()` on the token contract using the Permit Signature before calling `transferFrom`.
- MUST distribute fees atomically with the token transfer.
- MUST enforce `block.timestamp ≤ deadline` on every operation.
- MUST NOT provide the Administrator with any mechanism to transfer participant balances.

---

## Checkout Engine Components

### core-checkout-engine

| Field | Value |
|-------|-------|
| **Category** | Checkout Engine |
| **Governing Specification** | SSF-SPEC-000 (this document series), SSF-SPEC-001 |
| **Conformance** | Required for a fully conformant stack |

The central service for merchant payment session management.

**Key requirements:**
- MUST expose a charge creation API secured by mTLS per RFC 8705.
- MUST generate a cryptographically unique 32-byte `ref` for each session.
- MUST enforce single-use `payloadId` validation in coordination with the Broadcast Layer.
- MUST reconcile charges against confirmed `PermittedTransfer` events received from transfer-history, using the `ref` as the correlation key.
- MUST deliver settlement webhooks to Merchant Servers upon reconciliation.

### checkout-public-widget

| Field | Value |
|-------|-------|
| **Category** | Checkout Engine |
| **Governing Specification** | SSF-SPEC-000 |
| **Conformance** | Required for a fully conformant stack |

The hosted payment page that delivers session details to the wallet.

**Key requirements:**
- MUST issue Ephemeral Tokens with a configurable expiry, RECOMMENDED to be no longer than 5 minutes.
- MUST invalidate each Ephemeral Token upon first successful redemption.
- MUST NOT include any signing authority or private key material in an Ephemeral Token.
- MUST display the merchant-supplied order details accurately.

### ca-server

| Field | Value |
|-------|-------|
| **Category** | Checkout Engine — Infrastructure |
| **Governing Specification** | SSF-SPEC-000 |
| **Conformance** | Required for a fully conformant stack |

Internal Certificate Authority for mTLS certificate lifecycle management.

**Key requirements:**
- MUST issue certificates only to verified Merchant Servers and to the core-checkout-engine.
- MUST support certificate revocation.

### login-server

| Field | Value |
|-------|-------|
| **Category** | Checkout Engine — Infrastructure |
| **Governing Specification** | SSF-SPEC-000 |
| **Conformance** | Required for a fully conformant stack |

Issues JWT bearer tokens for API access within the mTLS context.

### credentials-manager

| Field | Value |
|-------|-------|
| **Category** | Checkout Engine — Infrastructure |
| **Governing Specification** | SSF-SPEC-000 |
| **Conformance** | Required for a fully conformant stack |

Administrative service for certificate and user lifecycle management.

### merchant-dashboard

| Field | Value |
|-------|-------|
| **Category** | Checkout Engine — Merchant UI |
| **Governing Specification** | SSF-SPEC-000 |
| **Conformance** | Required for a fully conformant stack |

Merchant-facing web interface for account management, real-time order monitoring, and reporting.

---

## Broadcast Layer Components

### broadcast-gateway

| Field | Value |
|-------|-------|
| **Category** | Broadcast Layer |
| **Governing Specification** | SSF-SPEC-000, SSF-SPEC-001 |
| **Conformance** | Required for a fully conformant stack |

External entry point for wallet connections.

**Key requirements:**
- MUST accept `TransferRequest` and `BuyAcquiringPackRequest` payloads as defined in SSF-SPEC-001.
- MUST maintain persistent WebSocket connections to deliver real-time status updates.
- MUST apply all validation rules defined in SSF-SPEC-001 Section 7 before forwarding to the broadcast-service.

### broadcast-service

| Field | Value |
|-------|-------|
| **Category** | Broadcast Layer |
| **Governing Specification** | SSF-SPEC-000 |
| **Conformance** | Required for a fully conformant stack |

Submission lifecycle management and status reporting.

**Key requirements:**
- MUST track each submission through a defined state machine.
- MUST publish state transitions to connected wallets in real time.
- MUST clearly distinguish *submission completion* from *transaction finality* in all status communications.

### broadcast-submitter

| Field | Value |
|-------|-------|
| **Category** | Broadcast Layer |
| **Governing Specification** | SSF-SPEC-000 |
| **Conformance** | Required for a fully conformant stack |

The Relayer. Holds the funded account that pays gas.

**Key requirements:**
- MUST submit only validated payloads.
- MUST NOT hold or have access to payer private keys.
- MUST record the transaction hash and propagate it to the broadcast-service upon submission.

### balance-and-history

| Field | Value |
|-------|-------|
| **Category** | Broadcast Layer |
| **Governing Specification** | SSF-SPEC-000 |
| **Conformance** | Required for a fully conformant stack |

Real-time wallet balance and history service.

**Key requirements:**
- MUST deliver on-chain transfer notifications to registered wallets via WebSocket.
- MUST NOT deliver a final settlement notification until transfer-history has confirmed the transaction.

---

## Shared Infrastructure Components

### transfer-history

| Field | Value |
|-------|-------|
| **Category** | Event Indexing |
| **Governing Specification** | SSF-SPEC-000 |
| **Conformance** | Required for a fully conformant stack |

On-chain event monitor and confirmation oracle.

**Key requirements:**
- MUST monitor the deployed Settlement Contract for `PermittedTransfer` and `AcquirerCreated` events.
- MUST wait for a configurable number of block confirmations before treating an event as final.
- MUST deliver confirmed events to the core-checkout-engine for reconciliation.
- MUST NOT alter or filter event data.

### basic-data-server

| Field | Value |
|-------|-------|
| **Category** | Shared Infrastructure |
| **Governing Specification** | SSF-SPEC-000 |
| **Conformance** | Required for a fully conformant stack |

Public reference data service.

**Key requirements:**
- MUST expose the list of supported ERC-2612 tokens accepted by the processor.
- MUST expose the registry of Service Providers (Acquirers who have opted in to public discovery).
- MUST expose the processor's public wallet addresses and Settlement Contract address.
- MUST be publicly accessible without authentication.
- MUST NOT expose any private or sensitive data.

---

## Client Components

### Client Wallet

| Field | Value |
|-------|-------|
| **Category** | Client |
| **Governing Specification** | SSF-SPEC-001 |
| **Conformance** | Required for a fully conformant client |

Any application that constructs and submits compliant payment payloads.

**Key requirements:**
- MUST construct Permit Signatures and Binding Signatures using EIP-712 with the correct domain separator.
- MUST read the payer's ERC-2612 nonce from the token contract before each signing operation.
- MUST submit payloads exclusively over TLS 1.2 or higher.
- MUST NOT transmit the payer's private key or seed phrase to any remote service.
- MUST NOT persist signed payloads in environments accessible to untrusted parties.
- MUST maintain a WebSocket connection to the broadcast-gateway for real-time status updates.
- MUST clearly distinguish submission completion from finality in status presented to the user.
