---
document: SSF-SPEC-001/core-concepts/system-architecture.md
spec: SSF-SPEC-001
version: 1.0.0
status: Draft
date: 2026-03-04
author: "Adalton Reis <reis@stablecoinstack.org>"
organization: Stablecoin Stack Foundation
license: Apache License 2.0
---

# System Architecture

---

## Principal Actors

Four principal actors drive every payment flow in the Stablecoin Stack:

| Actor | Role |
| ----- | ---- |
| **Merchant** | Creates charges and receives settlement confirmation. Integrates with the Checkout Engine over mTLS. Does not interact directly with the blockchain. |
| **Payer** | Holds the stablecoin, signs the payment commitment, and submits the payload via a compliant wallet. Never submits transactions directly to the network. |
| **Payment Processor** | Operates the off-chain infrastructure (Checkout Engine, Broadcast Layer, Relayer). Acts as the trusted intermediary between Merchants and the on-chain Settlement Contract. |
| **Acquirer** | A registered third party who distributes the payment service to payers and earns a share of the processing fee. May operate as a public Service Provider discoverable by wallets. |

---

## Component Overview

A fully conformant Stablecoin Stack deployment consists of the following components:

| Component | Subsystem | Description |
| --------- | --------- | ----------- |
| **Settlement Contract** | On-chain | Verifies signatures, executes transfers, distributes fees, registers acquirers. |
| **core-checkout-engine** | Checkout Engine | Merchant-facing API for session creation, charge management, and reconciliation. |
| **checkout-public-widget** | Checkout Engine | Payment page that issues Ephemeral Tokens and delivers session parameters to wallets. |
| **ca-server** | Checkout Engine | Internal Certificate Authority for mTLS session management. |
| **login-server** | Checkout Engine | Issues JWT bearer tokens within authenticated mTLS sessions. |
| **credentials-manager** | Checkout Engine | Certificate lifecycle and user administration. |
| **merchant-dashboard** | Checkout Engine | Merchant web interface for account management, order monitoring, and fee reporting. |
| **wallet-gateway** | Broadcast Layer | External entry point for wallet connections. Accepts payload submissions and maintains WebSocket connections for real-time status updates. Also provides nonce retrieval and fee calculation to wallets. |
| **broadcast-service** | Broadcast Layer | Manages submission lifecycle and state transitions. Publishes state updates to wallets via wallet-gateway. |
| **broadcast-submitter** | Broadcast Layer | Holds the Relayer's funded account. The only component that issues on-chain transactions. |
| **balance-and-history** | Broadcast Layer | Maintains real-time wallet balance views and delivers transfer notifications over WebSocket. |
| **transfer-history** | Event Indexing | Monitors the Settlement Contract for on-chain events. Publishes confirmed settlements to the Checkout Engine for reconciliation. |
| **basic-data-server** | Shared Infrastructure | Publicly accessible read-only service providing supported token lists, Service Provider registry, and processor configuration. |
| **Client Wallet** | Client | Signs and submits payment payloads. The only component that holds the payer's private key. |

---

## Trust Boundaries

Understanding the trust model is essential for reasoning about the security properties of the system.

**Fully trusted by the payer:** the payer's own private key and the wallet software that manages it. No other component is trusted to act on the payer's behalf without a valid signature.

**Trusted by the merchant within a verified mTLS session:** the core-checkout-engine, which receives charge creation requests and delivers settlement confirmation.

**Trusted to relay faithfully, but not trusted with funds:** the Payment Processor and Relayer. They can submit transactions, but cannot construct valid signatures without the payer's key. A malicious Processor cannot forge a payment.

**Trusted as a shared public resource:** the basic-data-server. It holds no private state and serves only public reference data. Its integrity is reputational, not cryptographic.

**Trust-minimised — verified on-chain:** the Settlement Contract. Its behaviour is determined by its deployed bytecode, verified independently by the blockchain network. No off-chain component can alter its execution.

---

## Design Principles

Every architectural decision in the Stablecoin Stack follows from five explicit principles:

**User-centric by design.** Technical complexity must not be a barrier to participation. The end user encounters a flow as simple as any mobile payment. All complexity — signature construction, nonce management, fee calculation, gas payment — is absorbed by the infrastructure.

**Payer sovereignty.** The authority to authorise a payment belongs exclusively to the payer, expressed through cryptographic signature. No component can move funds without a valid signed authorisation. The system is payer-agnostic: the identity of the account that submits a transaction is irrelevant to its validity.

**Gas abstraction as a baseline.** Gas is always paid by the Relayer, which recovers costs through the processing fee. The user holds only the stablecoin they wish to spend.

**Complementary rails, not replacement.** The Stablecoin Stack is a set of additional payment rails, particularly suited to cross-border transfers, privacy-preserving payments, contexts without a shared banking relationship, and scenarios where programmable settlement logic adds value.

**Open by default.** All specifications, reference implementations, and supporting materials are published under open licences. There are no proprietary extensions, no closed conformance test suites, and no tollgates on the information required to build a compliant implementation.

---

## Related Documents

- [Why Cryptographic Payments](./why-cryptographic-payments.md) — motivation and design philosophy
- [Payment Flow](./payment-flow.md) — end-to-end flow narrative
- [Formal Specification](../specifications/ssf-spec-001.md) — normative component requirements
- [Component Index](../reference/component-index.md) — authoritative component reference
