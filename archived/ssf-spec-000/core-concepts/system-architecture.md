# System Architecture

This document describes the structure of the Stablecoin Stack: who the principal actors are, what each component does, how components relate to one another, and where trust is placed and withheld. Reading this document gives you the mental model needed to understand any individual specification in the series.

---

## Principal Actors

Four parties interact in every payment flow. Understanding their roles and the boundaries between them is the first step to understanding the system.

| Actor | Role |
|-------|------|
| **Merchant** | Creates charges and receives settlement confirmation. Integrates with the Checkout Engine over mTLS. Does not interact with the blockchain directly. |
| **Payer** | Holds the stablecoin, signs the payment commitment, and submits the payload via a compliant wallet. Never submits transactions directly to the network. |
| **Payment Processor** | Operates the off-chain infrastructure: Checkout Engine, Broadcast Layer, and Relayer. Acts as the trusted intermediary between merchants and the on-chain Settlement Contract. |
| **Acquirer** | A registered third party who distributes the payment service to payers and earns a share of the processing fee. May operate as a public Service Provider discoverable by wallets. |

---

## Component Overview

A fully conformant Stablecoin Stack deployment consists of the following components, grouped by functional category.

### On-Chain

| Component | Role |
|-----------|------|
| **Settlement Contract** | The trust anchor of the system. Verifies permit and binding signatures, executes token transfers, distributes fees, and registers acquirers. Specified in SSF-SPEC-002. |

### Checkout Engine

| Component | Role |
|-----------|------|
| **core-checkout-engine** | Manages the payment session lifecycle: session creation, charge management, and settlement reconciliation. Exposes a merchant API protected by mTLS. |
| **checkout-public-widget** | A payment page hosted by the processor. Delivers session payment details to the wallet via an Ephemeral Token. The information bridge between the Checkout Engine and the wallet. |
| **ca-server** | Internal Certificate Authority. Issues and manages TLS certificates for mTLS sessions between merchants and the core-checkout-engine. |
| **login-server** | Issues JWT bearer tokens to merchants within the mTLS context, enabling programmatic API access. |
| **credentials-manager** | Manages certificate and user lifecycle. Interfaces with the ca-server and Keycloak. Used by administrator staff and automated provisioning workflows. |
| **merchant-dashboard** | The merchant-facing web UI. Account management, real-time order monitoring, fee reporting, and statistics. |

### Broadcast Layer

| Component | Role |
|-----------|------|
| **broadcast-gateway** | External entry point for wallet connections. Accepts payload submissions and maintains WebSocket connections for real-time status delivery. |
| **broadcast-service** | Manages the internal submission lifecycle: queuing, state tracking, and real-time status publication. |
| **broadcast-submitter** | Holds the Relayer's funded account. The only off-chain component that issues Ethereum transactions. |
| **balance-and-history** | Maintains real-time wallet balance and transaction history. Delivers on-chain transfer notifications to wallets over WebSocket. |

### Shared Infrastructure

| Component | Role |
|-----------|------|
| **transfer-history** | Monitors the Settlement Contract and all supported token contracts for on-chain events. Authoritative source of confirmed transaction data. Publishes confirmation events to the Checkout Engine. |
| **basic-data-server** | Publicly accessible read-only service. Provides supported token lists, registered Service Provider (Acquirer) data, and processor configuration. |

### Client

| Component | Role |
|-----------|------|
| **Client Wallet** | Constructs and signs compliant payment payloads. Submits them to the broadcast-gateway. Displays real-time payment status to the user. |

---

## Trust Boundaries

The security of the system depends on a clear understanding of which components are trusted for what, and which are not trusted at all.

### The Payer's Key — Fully Self-Sovereign

The payer's private key is the root of authority for every payment. Nothing in the stack can move the payer's funds without a valid signature produced by that key. No component is trusted to act on the payer's behalf without it. The wallet software that manages the key is trusted by the payer and by no one else.

### The Settlement Contract — Trust-Minimised and On-Chain

The Settlement Contract's behaviour is determined by its deployed code and enforced by the blockchain network. No off-chain component can alter its execution. Auditors, operators, and users can verify its logic independently. It is the only component in the system whose correctness does not require trusting an operator.

### The Payment Processor — Trusted to Relay, Not to Hold

The Payment Processor operates the off-chain infrastructure and acts as the Relayer. It is trusted to faithfully submit transactions and to not censor payments. It is **not** trusted with funds: it cannot construct valid signatures without the payer's key, and therefore cannot forge or redirect payments. A malicious or compromised Processor can delay or drop submissions, but cannot steal funds.

### The Checkout Engine — Trusted by Merchants, Within a Verified Session

The core-checkout-engine receives sensitive charge data from merchant servers. Merchants trust it to generate accurate sessions and deliver correct settlement confirmation. This trust is bounded by the mTLS session: neither party can be impersonated without a valid certificate.

### The Basic Data Service — Trusted as a Public Resource

The basic-data-server holds no private state and serves only public reference data. Its integrity is a reputational concern: incorrect data (e.g., a fabricated Service Provider listing) could mislead users, but cannot cause fund loss. The Foundation intends to operate a canonical instance as a shared public good.

### The Merchant — Trusted for Session Parameters

Merchants supply the charge parameters (amount, token, beneficiary) that populate the checkout session. The integrity of these parameters depends on the merchant's honesty and system security. The Settlement Contract enforces that the signed beneficiary receives the funds; it does not validate that the beneficiary is the merchant's legitimate account.

---

## The Acquiring Model

Acquirers are registered third-party participants who distribute the payment service and earn a share of the processing fee on referred payments. The model is inspired by the acquirer role in traditional card networks, but implemented transparently and on-chain.

### Registration

An Acquirer registers by calling `buyAcquiringPack` on the Settlement Contract, paying the registration fee in an accepted stablecoin and selecting a fee percentage (subject to the `maxAcquiringFee` ceiling). Upon success, a `bytes16` UUID **Acquirer ID** is assigned on-chain.

The payer of the registration fee and the account being registered do not need to be the same address, enabling delegated or sponsored registration.

### Fee Distribution

Every payment that includes a non-zero Acquirer ID triggers an automatic fee distribution to the acquirer's on-chain balance. The fee is calculated as a percentage of the principal and is distributed atomically with the payment. Payments without an acquirer use the **Zero-UUID** (`0x00000000000000000000000000000000`) as a convention that suppresses the acquiring fee.

### Service Provider Discovery

Acquirers who wish to be discoverable by wallet users may opt in to the **basic-data-server** public registry, where they appear as **Service Providers**. Wallets present this list to users, enabling provider selection based on fee rates, reputation, or personal preference. This gives users meaningful choice without requiring them to understand the underlying acquiring infrastructure.

---

## Data Flow Summary

The following describes how data moves through the system during a payment, without prescribing timing or transport details. The full step-by-step narrative is in [Payment Flow](./payment-flow.md).

```
Merchant Server
    │  charge parameters (mTLS)
    ▼
core-checkout-engine ──── generates ──── ref · payloadId · deadline
    │  checkout URL + ephemeral token
    ▼
Payer (via checkout-public-widget)
    │  session details
    ▼
Client Wallet
    │  reads nonce from token contract
    │  produces permitSig + payWithPermitSig
    │  TransferRequest payload
    ▼
broadcast-gateway → broadcast-service → broadcast-submitter
                                              │  transferWithPermit(...)
                                              ▼
                                      Settlement Contract
                                              │  PermittedTransfer event
                                              ▼
                                      transfer-history
                                              │  confirmed event (after N blocks)
                                              ▼
                                      core-checkout-engine
                                              │  webhook
                                              ▼
                                      Merchant Server
```

Each arrow represents a trust-bounded interaction. No component passes signing authority to any other. The Settlement Contract is the only place where funds move.
