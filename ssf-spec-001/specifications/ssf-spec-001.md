---
document: SSF-SPEC-001/specifications/ssf-spec-001.md
spec: SSF-SPEC-001
version: 1.0.0
status: Draft
date: 2026-03-04
author: "Adalton Reis <reis@stablecoinstack.org>"
organization: Stablecoin Stack Foundation
license: Apache License 2.0
---

# SSF-SPEC-001 — The Stablecoin Stack

| Field | Value |
| ----- | ----- |
| **Document ID** | SSF-SPEC-001 |
| **Title** | The Stablecoin Stack |
| **Version** | 1.0.0 |
| **Status** | Draft |
| **Date** | 2026-03-04 |
| **Supersedes** | SSF-SPEC-000, SSF-SPEC-001 (pre-unification), SSF-SPEC-002 (pre-unification) |
| **Author(s)** | Adalton Reis \<reis@stablecoinstack.org\> |
| **Reviewers** | — |
| **Organization** | Stablecoin Stack Foundation |
| **Contact** | contact@stablecoinstack.org |
| **Public Address** | `0x0000000000000000000000000000000000000000` |
| **License** | Apache License 2.0 |

---

## Table of Contents

1. [Abstract](#1-abstract)
2. [Introduction](#2-introduction)
   - 2.1 [Purpose](#21-purpose)
   - 2.2 [Scope](#22-scope)
   - 2.3 [Normative References](#23-normative-references)
   - 2.4 [Conventions and Terminology](#24-conventions-and-terminology)
3. [Vision and Design Philosophy](#3-vision-and-design-philosophy)
   - 3.1 [User-Centric by Design](#31-user-centric-by-design)
   - 3.2 [Payer Sovereignty](#32-payer-sovereignty)
   - 3.3 [Gas Abstraction as a Baseline](#33-gas-abstraction-as-a-baseline)
   - 3.4 [Complementary Rails, Not Replacement](#34-complementary-rails-not-replacement)
   - 3.5 [Open by Default](#35-open-by-default)
4. [System Architecture](#4-system-architecture)
   - 4.1 [Principal Actors](#41-principal-actors)
   - 4.2 [Component Overview](#42-component-overview)
   - 4.3 [Trust Boundaries](#43-trust-boundaries)
5. [Component Specifications](#5-component-specifications)
   - 5.1 [Settlement Contract](#51-settlement-contract)
   - 5.2 [Checkout Engine](#52-checkout-engine)
   - 5.3 [Broadcast Layer](#53-broadcast-layer)
   - 5.4 [Event Indexing and History](#54-event-indexing-and-history)
   - 5.5 [Basic Data Service](#55-basic-data-service)
   - 5.6 [Client Wallet](#56-client-wallet)
   - 5.7 [Merchant Dashboard](#57-merchant-dashboard)
6. [Cryptographic Conventions](#6-cryptographic-conventions)
   - 6.1 [Signature Algorithm](#61-signature-algorithm)
   - 6.2 [Typed Data Signing (EIP-712)](#62-typed-data-signing-eip-712)
   - 6.3 [Signature Representation](#63-signature-representation)
7. [Data Structures](#7-data-structures)
   - 7.1 [Signature Object — ERC20RelayerSig](#71-signature-object--erc20relayersig)
   - 7.2 [ERC-2612 Permit Parameters — PermitParams](#72-erc-2612-permit-parameters--permitparams)
   - 7.3 [Payment Transfer Parameters — PayWithPermitParams](#73-payment-transfer-parameters--paywithpermitparams)
   - 7.4 [Acquirer Registration Parameters — BuyAcquiringPackPermitParams](#74-acquirer-registration-parameters--buyacquiringpackpermitparams)
8. [Submission Payloads](#8-submission-payloads)
   - 8.1 [Two-Signature Pattern](#81-two-signature-pattern)
   - 8.2 [Payment Transfer Payload — TransferRequest](#82-payment-transfer-payload--transferrequest)
   - 8.3 [Acquirer Registration Payload — BuyAcquiringPackRequest](#83-acquirer-registration-payload--buyacquiringpackrequest)
9. [The Acquiring Model](#9-the-acquiring-model)
   - 9.1 [Overview](#91-overview)
   - 9.2 [Acquirer Registration](#92-acquirer-registration)
   - 9.3 [Fee Distribution](#93-fee-distribution)
   - 9.4 [Service Provider Discovery](#94-service-provider-discovery)
10. [Settlement Contract — State Variables](#10-settlement-contract--state-variables)
11. [Settlement Contract — Events](#11-settlement-contract--events)
12. [Settlement Contract — Fee Model](#12-settlement-contract--fee-model)
    - 12.1 [Overview](#121-overview)
    - 12.2 [calculateFees](#122-calculatefees)
    - 12.3 [breakdownTransferAmount](#123-breakdowntransferamount)
13. [Settlement Contract — Functions](#13-settlement-contract--functions)
    - 13.1 [transferWithPermit](#131-transferwithpermit)
    - 13.2 [buyAcquiringPack](#132-buyacquiringpack)
    - 13.3 [getBalances (single token)](#133-getbalances-single-token)
    - 13.4 [getBalances (multi-token overload)](#134-getbalances-multi-token-overload)
    - 13.5 [getAcquiringWallet](#135-getacquiringwallet)
14. [Validation Rules](#14-validation-rules)
    - 14.1 [Structural Validation](#141-structural-validation)
    - 14.2 [Semantic Validation](#142-semantic-validation)
    - 14.3 [Cryptographic Validation](#143-cryptographic-validation)
15. [Submission Flow](#15-submission-flow)
    - 15.1 [Payment Transfer — Step-by-Step](#151-payment-transfer--step-by-step)
    - 15.2 [Acquirer Registration — Step-by-Step](#152-acquirer-registration--step-by-step)
16. [End-to-End Payment Flow](#16-end-to-end-payment-flow)
    - 16.1 [Phase A — Session Creation](#161-phase-a--session-creation)
    - 16.2 [Phase B — Payload Construction and Signing](#162-phase-b--payload-construction-and-signing)
    - 16.3 [Phase C — Submission and Off-Chain Validation](#163-phase-c--submission-and-off-chain-validation)
    - 16.4 [Phase D — On-Chain Settlement](#164-phase-d--on-chain-settlement)
    - 16.5 [Phase E — Confirmation and Reconciliation](#165-phase-e--confirmation-and-reconciliation)
17. [Error Handling](#17-error-handling)
18. [Security Model](#18-security-model)
    - 18.1 [Threat Model](#181-threat-model)
    - 18.2 [Dual-Signature Binding](#182-dual-signature-binding)
    - 18.3 [Replay Protection](#183-replay-protection)
    - 18.4 [Payer-Agnostic Execution and its Implications](#184-payer-agnostic-execution-and-its-implications)
    - 18.5 [Ephemeral Token Security](#185-ephemeral-token-security)
    - 18.6 [mTLS for Merchant Communication](#186-mtls-for-merchant-communication)
    - 18.7 [Administrative Privilege Boundaries](#187-administrative-privilege-boundaries)
19. [Conformance](#19-conformance)
    - 19.1 [Fully Compliant Stack](#191-fully-compliant-stack)
    - 19.2 [Partial Conformance](#192-partial-conformance)
    - 19.3 [Future Specifications](#193-future-specifications)
20. [Future Work](#20-future-work)
21. [Versioning](#21-versioning)
22. [Change Log](#22-change-log)
23. [License](#23-license)

---

## 1. Abstract

This document is the foundational specification of the **Stablecoin Stack** — an open architecture for processing stablecoin payments on Ethereum-compatible networks. It establishes the design philosophy, system architecture, component responsibilities, cryptographic conventions, data structures, submission protocols, on-chain contract interface, end-to-end payment flow, security model, and conformance requirements for the full stack.

The Stablecoin Stack is built on a central proposition: that cryptographic signatures are a superior foundation for payment authorisation compared to identity-based delegation models, and that the tooling to make this practical for everyday users is both achievable and urgently needed. All architectural decisions described herein follow directly from this proposition.

This is the single authoritative specification for the Stablecoin Stack. It supersedes the pre-unification documents SSF-SPEC-000, the pre-unification SSF-SPEC-001, and SSF-SPEC-002. Future companion specifications covering individual component interfaces will reference this document.

---

## 2. Introduction

### 2.1 Purpose

This document provides the normative definitions required to implement, deploy, audit, or assess conformance of any component of the Stablecoin Stack. It is intended for:

- developers integrating a wallet or checkout front-end with the payment processor API;
- smart-contract engineers implementing or auditing a conformant Settlement Contract;
- payment processor operators who need to understand the on-chain guarantees upon which their off-chain service depends;
- implementers building any conformant component of the Stablecoin Stack; and
- auditors, regulators, and reviewers assessing the correctness, security, and compliance of an existing implementation.

### 2.2 Scope

This specification covers:

- the design principles governing all architectural decisions;
- the system architecture, principal actors, component responsibilities, and trust boundaries;
- the cryptographic conventions for all signatures;
- all data structures accepted by the payment processor;
- the two primary submission flows: payment transfer and acquirer registration;
- the full on-chain Settlement Contract interface: state variables, events, fee model, and functions;
- the validation rules a compliant processor MUST enforce before broadcasting;
- the end-to-end payment flow from session creation through settlement confirmation;
- the security model and threat mitigations; and
- conformance requirements for full and partial implementations.

This specification does not cover:

- the Checkout Engine API exposed to merchant servers (planned: future specification);
- the Broadcast Layer WebSocket protocol (planned: future specification);
- individual microservice interface specifications (planned: future specifications);
- key management and custody procedures;
- token issuance and redemption mechanics;
- regulatory compliance guidance; and
- offline transaction signing (identified as future work).

### 2.3 Normative References

| Reference | Description |
| --------- | ----------- |
| ERC-20 | Token Standard — Ethereum Improvement Proposal 20 |
| ERC-2612 | Permit Extension for EIP-20 Signed Approvals |
| EIP-712 | Typed Structured Data Hashing and Signing |
| ECDSA | Elliptic Curve Digital Signature Algorithm (FIPS 186-4) |
| SEC 1 v2.0 | Elliptic Curve Cryptography — SECG Standard |
| RFC 2119 | Key words for use in RFCs to Indicate Requirement Levels |
| RFC 4648 | The Base16, Base32, and Base64 Data Encodings |
| RFC 5246 | The Transport Layer Security (TLS) Protocol Version 1.2 |
| RFC 8446 | The Transport Layer Security (TLS) Protocol Version 1.3 |
| RFC 8705 | OAuth 2.0 Mutual-TLS Client Authentication |
| Solidity 0.8.x | Smart Contract Programming Language — solidity-lang.org |
| OpenZeppelin Contracts | Audited Solidity contract library — openzeppelin.com |

### 2.4 Conventions and Terminology

The key words **MUST**, **MUST NOT**, **REQUIRED**, **SHALL**, **SHOULD**, **RECOMMENDED**, **MAY**, and **OPTIONAL** in this document are to be interpreted as described in RFC 2119.

| Term | Definition |
| ---- | ---------- |
| Stablecoin Stack | The full set of components, protocols, and interfaces specified by this document. |
| Payment Processor | An operator that deploys and runs a conformant instance of the Stablecoin Stack. Responsible for the Broadcast Layer, Checkout Engine, and Relayer. |
| Relayer | The account that submits signed transactions on-chain on behalf of payers, absorbing gas costs. Operated by the Payment Processor. |
| Merchant | A business or individual that accepts stablecoin payments by integrating with the Checkout Engine. |
| Payer | The individual who holds the stablecoin and signs the payment commitment. |
| Acquirer | A registered third-party participant entitled to a share of the processing fee on payments they referred. |
| Service Provider | An Acquirer who has opted in to public discovery via the Basic Data Service. Visible to wallets as a selectable payment provider. |
| Settlement Contract | The on-chain Solidity smart contract that verifies signatures, executes token transfers, and distributes fees. |
| Permit Signature | An ERC-2612 off-chain signature authorising the Settlement Contract to call `permit()` on a token contract, granting a token allowance. Corresponds to `permitSig` in payload structures. |
| Binding Signature | An EIP-712 signature over the full operation parameters, authorising the processor to submit a specific operation. Corresponds to `payWithPermitSig` in payload structures. |
| Charge | A processor-issued record representing an expected payment of a specific amount, in a specific token, to a specific merchant, within a specific time window. |
| Session | The lifecycle context of a single charge, from creation through to settlement or expiry. |
| Order Reference | A 16-byte reference created by the merchant for tracking purposes. Embedded in the on-chain `PermittedTransfer` event and used for session reconciliation. |
| Payload ID | A client-generated identifier correlating a submission with a specific broadcasting intent. Both the client and the processor MUST use this ID to ensure request idempotency; the processor MUST validate that this ID is known, currently unprocessed, and unique to the broadcast session. |
| Acquirer ID | A 16-byte (`bytes16`) UUID uniquely identifying a registered Acquirer within the Settlement Contract. |
| Zero-UUID | The Acquirer ID consisting entirely of zero bytes (`0x00000000000000000000000000000000`). Represents the absence of an Acquirer on a given payment. |
| Ephemeral Token | A short-lived, single-use token issued by the checkout-public-widget that authorises a wallet to retrieve session payment details. |
| Principal Amount | The token amount intended to reach the beneficiary before fee deduction. |
| Total With Fees | The total token amount that must be held by the payer, equal to the principal amount plus all applicable fees. |
| Base Fee | The minimum absolute fee charged by the processor on every transfer, in token units. |
| Acquiring Fee | The additional percentage-based fee charged on behalf of a registered Acquirer. |
| Administrator | The privileged account that controls operational parameters of the Settlement Contract. MUST NOT have the ability to move user funds. |
| Hex string | A UTF-8 string with a `0x` prefix followed by an even number of lowercase hexadecimal digits. |
| EVM Address | A 20-byte account identifier, represented as a 42-character hex string (`0x` + 40 lowercase hex digits). |
| acquiringPrice | The amount that must be transferred to the Settlement Contract to register a new Acquirer, in the smallest unit of the respective token. |

---

## 3. Vision and Design Philosophy

Every architectural decision in the Stablecoin Stack derives from the following explicit principles.

### 3.1 User-Centric by Design

Technical complexity must not be a barrier to participation. The end user — the person making a payment — SHOULD encounter a flow as simple as any mobile payment: open a wallet, confirm a transaction, done. All complexity — signature construction, nonce management, fee calculation, gas payment — MUST be absorbed by the infrastructure and hidden behind well-designed interfaces.

This principle applies equally to merchants. A merchant integrating the Stablecoin Stack SHOULD NOT need to understand ERC-2612 or EIP-712. They interact with an API that accepts standard e-commerce concepts: charges, sessions, webhooks, and reconciliation reports.

### 3.2 Payer Sovereignty

The authority to authorise a payment belongs exclusively to the payer, expressed through cryptographic signature. No component of the stack — not the processor, not the relayer, not the checkout engine — can move funds on a payer's behalf without a valid signed authorisation. The signature is the payment commitment; everything else is infrastructure for delivering and executing it.

The system is payer-agnostic: the identity of the account that submits a transaction to the network is irrelevant to its validity. Provided the signatures are correct, the payment executes.

### 3.3 Gas Abstraction as a Baseline

Requiring users to hold native gas tokens creates friction that is, in the context of a payment system, entirely artificial. The Stablecoin Stack resolves this at the architectural level: gas is always paid by the Relayer, which recovers costs through the processing fee. The user holds only the stablecoin they wish to spend.

### 3.4 Complementary Rails, Not Replacement

The Stablecoin Stack does not position itself as a replacement for existing payment infrastructure. It is a set of complementary rails, particularly well-suited to cross-border transfers, privacy-preserving payments, contexts where the parties lack a shared banking relationship, and scenarios where programmable settlement logic adds value.

### 3.5 Open by Default

All specifications, reference implementations, and supporting materials produced by the Foundation are published under open licences. There are no proprietary extensions, no closed conformance test suites, and no tollgates on the information required to build a compliant implementation.

---

## 4. System Architecture

### 4.1 Principal Actors

| Actor | Role |
| ----- | ---- |
| **Merchant** | Creates charges and receives settlement confirmation. Integrates with the Checkout Engine over mTLS. Does not interact directly with the blockchain. |
| **Payer** | Holds the stablecoin, signs the payment commitment, and submits the payload via a compliant wallet. Never submits transactions directly to the network. |
| **Payment Processor** | Operates the off-chain infrastructure (Checkout Engine, Broadcast Layer, Relayer). Acts as the trusted intermediary between Merchants and the on-chain Settlement Contract. |
| **Acquirer** | A registered third party who distributes the payment service to payers and earns a share of the processing fee. May operate as a public Service Provider. |

### 4.2 Component Overview

A fully conformant Stablecoin Stack deployment MUST include the following components:

| Component | Subsystem | Key Requirements |
| --------- | --------- | ---------------- |
| Settlement Contract | On-chain | MUST satisfy all requirements of Sections 10–13 of this specification. |
| core-checkout-engine | Checkout Engine | MUST expose a charge creation API to merchants over mTLS. MUST reconcile sessions on confirmed `PermittedTransfer` events. |
| checkout-public-widget | Checkout Engine | MUST issue single-use Ephemeral Tokens. MUST deliver session parameters to wallets. |
| ca-server | Checkout Engine | MUST issue and manage certificates for mTLS sessions between merchant servers and core-checkout-engine. |
| login-server | Checkout Engine | MUST issue JWT bearer tokens within authenticated mTLS sessions. |
| credentials-manager | Checkout Engine | MUST manage certificate lifecycle and user administration. |
| merchant-dashboard | Checkout Engine | SHOULD provide real-time order monitoring, fee reporting, and account management. |
| wallet-gateway | Broadcast Layer | MUST accept `TransferRequest` and `BuyAcquiringPackRequest` payloads. MUST maintain WebSocket connections for real-time status updates. SHOULD provide nonce retrieval and fee calculation to wallets. |
| broadcast-service | Broadcast Layer | MUST enqueue validated payloads, manage submission state transitions, and publish state updates to wallets via wallet-gateway. |
| broadcast-submitter | Broadcast Layer | MUST hold the Relayer's funded account and be the sole component that issues on-chain transactions. |
| balance-and-history | Broadcast Layer | SHOULD maintain real-time wallet balance views and deliver transfer notifications over WebSocket. |
| transfer-history | Event Indexing | MUST monitor the Settlement Contract for `PermittedTransfer` events and publish confirmed events to the Checkout Engine after a configurable number of block confirmations. |
| basic-data-server | Shared Infrastructure | MUST expose a publicly accessible read-only API providing supported token lists, Service Provider registry, and processor configuration. |
| Client Wallet | Client | MUST satisfy all wallet requirements stated in Section 5.6. |

### 4.3 Trust Boundaries

**Fully trusted by the payer:** the payer's own private key and the wallet software that manages it. No other component is trusted to act on the payer's behalf without a valid signature.

**Trusted by the merchant within a verified mTLS session:** the core-checkout-engine, which receives charge creation requests and delivers settlement confirmation.

**Trusted to relay faithfully, but not trusted with funds:** the Payment Processor and Relayer. They can submit transactions but cannot construct valid signatures without the payer's key. A malicious Processor cannot forge a payment.

**Trusted as a shared public resource:** the basic-data-server. It holds no private state and serves only public reference data.

**Trust-minimised — verified on-chain:** the Settlement Contract. Its behaviour is determined by its deployed bytecode, verified independently by the blockchain network. No off-chain component can alter its execution.

---

## 5. Component Specifications

### 5.1 Settlement Contract

The Settlement Contract is the trust anchor of the entire system. Its responsibilities are:

- verifying the Permit Signature against the ERC-2612 token contract;
- verifying the Binding Signature against the processor's authorisation;
- preventing replay attacks by recording consumed Binding Signature hashes in `usedHashes`;
- transferring the full payment amount from the payer via `permit()` and `transferFrom()`;
- computing and distributing fees to the processor and, if applicable, the acquirer;
- crediting the principal amount to the merchant beneficiary; and
- registering new acquirers upon receipt of a valid `buyAcquiringPack` call.

The Settlement Contract MUST be deployed by the Payment Processor and its address MUST be published in the Basic Data Service. The full on-chain interface is specified in Sections 10–13.

### 5.2 Checkout Engine

The Checkout Engine is the merchant-facing subsystem. It manages the lifecycle of payment sessions from charge creation through final reconciliation.

**core-checkout-engine** — The central service. Exposes an API to merchant servers for session creation, charge management, and reconciliation queries. Receives confirmed settlement events from transfer-history and marks corresponding charges as settled. Communication with merchants MUST be secured by mutual TLS (mTLS).

**checkout-public-widget** — A payment page hosted by the Payment Processor. Displays merchant-supplied order information and issues an Ephemeral Token that the wallet uses to fetch payment parameters. The widget does not accept payment directly; it serves only as a secure information delivery mechanism between the Checkout Engine and the wallet.

**ca-server** — An internal Certificate Authority that issues and manages the client and server certificates used in mTLS sessions between merchant servers and the core-checkout-engine.

**login-server** — Provides merchants with a mechanism to obtain JWT bearer tokens within the mTLS context, simplifying programmatic API access.

**credentials-manager** — Administrative service for certificate lifecycle management and user administration. Interfaces with the ca-server and Keycloak identity service.

**merchant-dashboard** — The merchant-facing web interface. Provides account management, real-time order monitoring, fee reporting, and statistical views.

### 5.3 Broadcast Layer

The Broadcast Layer is responsible for receipt, validation, queuing, submission, and real-time status reporting of signed payment payloads.

**wallet-gateway** — The external entry point for wallet connections. Accepts payload submissions and maintains persistent WebSocket connections for real-time status updates. Also provides nonce retrieval and fee calculation as convenience endpoints, reducing the need for wallets to perform direct on-chain calls.

**broadcast-service** — Manages the internal lifecycle of a submission. Enqueues validated payloads and tracks each through a defined set of states. State transitions are published in real time to connected wallets via wallet-gateway. **Submission completion** — meaning the Relayer received no revert when broadcasting — does NOT constitute final settlement. Finality is confirmed by transfer-history after sufficient block confirmations.

**broadcast-submitter** — The component that holds the Relayer's funded account and submits signed payloads on-chain. This is the only component in the off-chain stack that issues Ethereum transactions.

**balance-and-history** — Maintains a real-time view of known wallet balances and delivers on-chain transfer notifications to wallets over WebSocket.

### 5.4 Event Indexing and History

**transfer-history** — Monitors the Settlement Contract and all supported ERC-2612 token contracts for on-chain events. When a `PermittedTransfer` event is observed with sufficient block confirmation depth, transfer-history publishes the confirmed event to the Checkout Engine, which uses the `ref` field to reconcile the corresponding charge. This service has no write authority over any other component; it is a read-and-publish relay.

### 5.5 Basic Data Service

**basic-data-server** — A publicly accessible, read-only service providing:

- the list of ERC-2612-compliant stablecoins supported by the processor;
- the registry of registered Acquirers who have opted in to public discovery (Service Providers); and
- the public wallet addresses and configuration of the Payment Processor.

This service is intentionally designed to be operable as a shared public resource. The Foundation intends to operate a canonical instance as a free public good for the ecosystem.

### 5.6 Client Wallet

A conformant Client Wallet MUST be able to:

- retrieve payment session details from the checkout-public-widget using an Ephemeral Token;
- read the payer's ERC-2612 nonce from the token contract or wallet-gateway;
- construct and sign a valid `PermitParams` structure using EIP-712;
- construct and sign a valid `PayWithPermitParams` structure using EIP-712;
- assemble a complete `TransferRequest` payload as specified in Section 8.2;
- submit the payload to the wallet-gateway and maintain a WebSocket connection for status updates; and
- present the payment status to the user in a clear and timely manner.

The wallet MUST NOT transmit the payer's private key or seed phrase to any remote service at any time. All signing MUST be performed locally on the user's device.

### 5.7 Merchant Dashboard

The merchant-dashboard provides merchants with a self-service interface to manage their integration with the Checkout Engine. It does not interact with the blockchain directly. Its backend delegates all payment session and reconciliation logic to the core-checkout-engine. Merchants access it using credentials managed by the credentials-manager.

---

## 6. Cryptographic Conventions

### 6.1 Signature Algorithm

All signatures in this specification are produced using the **Elliptic Curve Digital Signature Algorithm (ECDSA)** over the **secp256k1** curve, as specified in FIPS 186-4 and SEC 1 v2.0. This is the same scheme used natively by the Ethereum protocol.

### 6.2 Typed Data Signing (EIP-712)

Every signature MUST be computed over a structured hash produced in accordance with **EIP-712**. This scheme binds each signature to a specific domain (chain identifier, contract address, and version) and to a well-typed data structure, preventing cross-domain and cross-type replay attacks.

A compliant signer MUST construct the digest as:

```
digest = keccak256(
    0x1901                    ||
    domainSeparator           ||
    hashStruct(message)
)
```

where `domainSeparator` encodes the `name`, `version`, `chainId`, and `verifyingContract` of the relevant contract.

The **Permit Signature** MUST use the token contract's domain separator. The **Binding Signature** MUST use the Settlement Contract's domain separator.

### 6.3 Signature Representation

| Field | Type / Format | Required | Description |
| ----- | ------------- | -------- | ----------- |
| `hash` | `bytes32` — 66-char hex string | REQUIRED | The EIP-712 digest that was signed. `0x` + 64 lowercase hex characters. |
| `v` | `uint8` — integer | REQUIRED | Recovery identifier. MUST be `27` or `28`. Some signers produce `0` or `1`; processors MUST normalise by adding 27 if the value is less than 27. |
| `r` | `bytes32` — 66-char hex string | REQUIRED | The `r` component of the ECDSA signature. `0x` + 64 lowercase hex characters. |
| `s` | `bytes32` — 66-char hex string | REQUIRED | The `s` component of the ECDSA signature. `0x` + 64 lowercase hex characters. |

All hex strings throughout this specification MUST use the `0x` prefix and lowercase hexadecimal digits. Leading zero bytes MUST be preserved; the encoded string MUST always be the full expected length.

---

## 7. Data Structures

### 7.1 Signature Object — `ERC20RelayerSig`

A generic container for an ECDSA signature. Reused as `permitSig` and `payWithPermitSig` across all payload types.

| Field | Type / Format | Required | Description |
| ----- | ------------- | -------- | ----------- |
| `hash` | `bytes32` — 66-char hex string | REQUIRED | The EIP-712 message digest. Must be exactly 32 bytes (`0x` + 64 hex chars). |
| `v` | `uint8` — integer (`27` or `28`) | REQUIRED | ECDSA recovery parameter. Processors MUST reject values outside `{0, 1, 27, 28}` and normalise `0` and `1` to `27` and `28` respectively. |
| `r` | `bytes32` — 66-char hex string | REQUIRED | First 32-byte component. Must be exactly 32 bytes. |
| `s` | `bytes32` — 66-char hex string | REQUIRED | Second 32-byte component. Must be exactly 32 bytes. |

### 7.2 ERC-2612 Permit Parameters — `PermitParams`

Encapsulates the parameters of an ERC-2612 permit authorisation. Incorporated verbatim into the on-chain `permit()` call.

| Field | Type / Format | Required | Description |
| ----- | ------------- | -------- | ----------- |
| `owner` | `address` — 42-char hex string | REQUIRED | The Ethereum address of the token holder granting the approval. |
| `spender` | `address` — 42-char hex string | REQUIRED | The address authorised to transfer tokens. Typically the Settlement Contract address. |
| `value` | `uint256` — decimal or hex string | REQUIRED | The total token amount approved, in the token's smallest unit. MUST equal the principal plus all applicable fees. |
| `nonce` | `uint256` — decimal or hex string | REQUIRED | The current ERC-2612 permit nonce of the owner on the token contract. The processor MUST verify this matches the on-chain nonce at submission time. |
| `deadline` | `uint256` — Unix timestamp (seconds) | REQUIRED | The permit expiry. The Settlement Contract will revert if `block.timestamp` exceeds this value. |

### 7.3 Payment Transfer Parameters — `PayWithPermitParams`

Defines the business-level parameters of a stablecoin payment.

| Field | Type / Format | Required | Description |
| ----- | ------------- | -------- | ----------- |
| `token` | `address` — 42-char hex string | REQUIRED | The contract address of the ERC-2612-compliant stablecoin to be transferred. |
| `beneficiary` | `address` — 42-char hex string | REQUIRED | The merchant's receiving address. Funds are credited here after fee deduction. |
| `ref` | `bytes32` — 66-char hex string | REQUIRED | A 32-byte field formed by the concatenation of the 16-byte Order Reference and the 16-byte Acquirer ID. Missing parts MUST be filled with zero bytes. This field MUST always be present. |
| `permitParams` | `PermitParams` — nested object | REQUIRED | The ERC-2612 permit parameters. The `owner` field MUST match the address recovered from the Permit Signature. |

### 7.4 Acquirer Registration Parameters — `BuyAcquiringPackPermitParams`

Defines the parameters for purchasing an acquirer registration entitlement.

| Field | Type / Format | Required | Description |
| ----- | ------------- | -------- | ----------- |
| `token` | `address` — 42-char hex string | REQUIRED | The ERC-2612-compliant token used to pay the registration fee. Must be in `acquiringAllowedTokens`. |
| `feeValue` | `uint256` — decimal or hex string | REQUIRED | The registration fee amount. MUST equal `permitParams.value` and the current `acquiringPrice`. |
| `acquiring` | `address` — 42-char hex string | REQUIRED | The Ethereum address to be registered as an Acquirer. Must not already be registered. |
| `permitParams` | `PermitParams` — nested object | REQUIRED | The ERC-2612 permit parameters. The `value` field MUST equal `feeValue`. |

---

## 8. Submission Payloads

### 8.1 Two-Signature Pattern

Every submission payload carries exactly two signature objects:

| Signature field | What it signs | Verified by |
| --------------- | ------------- | ----------- |
| `permitSig` | ERC-2612 `Permit` typed-data: authorises the Settlement Contract to call `permit()` on the token contract. Produced by the token holder. | The ERC-2612 token contract on-chain. |
| `payWithPermitSig` | The full operation parameters. Authorises the processor to submit this specific operation. Produced by the payer. | The Payment Processor off-chain, and the Settlement Contract on-chain. |

Separating these two signatures ensures that neither the Payment Processor nor the token contract can unilaterally reuse an authorisation in an unintended context.

### 8.2 Payment Transfer Payload — `TransferRequest`

Submitted to initiate a stablecoin payment from a payer to a merchant beneficiary.

| Field | Type / Format | Required | Description |
| ----- | ------------- | -------- | ----------- |
| `payWithPermitParams` | `PayWithPermitParams` — nested object | REQUIRED | Payment parameters as defined in Section 7.3. |
| `payWithPermitSig` | `ERC20RelayerSig` — nested object | REQUIRED | Binding Signature over `payWithPermitParams`. The processor MUST recover the signer and verify it matches `permitParams.owner`. |
| `permitSig` | `ERC20RelayerSig` — nested object | REQUIRED | Permit Signature over the token's `Permit` typed-data. Forwarded verbatim to the Settlement Contract. |
| `payloadId` | `string` — UUID or processor-issued opaque ID | REQUIRED | Client-generated identifier for idempotency. The processor MUST validate that this ID is known, currently unprocessed, and unique to the broadcast session. |

### 8.3 Acquirer Registration Payload — `BuyAcquiringPackRequest`

Submitted to register a new Acquirer. Upon settlement, the `acquiring` address is registered on-chain with a unique Acquirer ID.

| Field | Type / Format | Required | Description |
| ----- | ------------- | -------- | ----------- |
| `buyAcquiringPackParams` | `BuyAcquiringPackPermitParams` — nested object | REQUIRED | Registration parameters as defined in Section 7.4. |
| `payWithPermitSig` | `ERC20RelayerSig` — nested object | REQUIRED | Binding Signature over `buyAcquiringPackParams`. The processor MUST recover the signer and verify it matches `permitParams.owner`. |
| `permitSig` | `ERC20RelayerSig` — nested object | REQUIRED | Permit Signature. The signed `value` MUST equal `buyAcquiringPackParams.feeValue`. Forwarded verbatim to the Settlement Contract. |

---

## 9. The Acquiring Model

### 9.1 Overview

The Stablecoin Stack incorporates a first-class acquiring model allowing third-party participants — **Acquirers** — to distribute the payment service and earn a share of the processing fee. Acquirers are identified by a `bytes16` UUID Acquirer ID. When a payer includes an Acquirer ID in a payment, the Settlement Contract automatically distributes the acquiring fee to the registered acquirer's balance. This distribution is atomic with the payment and is reflected in the `CommissionGenerated` event.

### 9.2 Acquirer Registration

Registration is performed by submitting a `BuyAcquiringPackRequest` payload to the Payment Processor, which broadcasts the corresponding `buyAcquiringPack` call on the Settlement Contract. Registration requires:

- payment of the registration fee (`acquiringPrice`) in an accepted stablecoin;
- selection of a fee percentage, subject to the `maxAcquiringFee` ceiling; and
- the wallet address to be registered as the acquirer.

The payer and the account being registered do not need to be the same address, enabling delegated or sponsored registration. Upon successful registration, a new Acquirer ID is assigned on-chain and the `AcquirerCreated` event is emitted.

### 9.3 Fee Distribution

Every payment processed by the Settlement Contract is subject to two fee components:

| Fee | Type | Recipient |
| --- | ---- | --------- |
| Base Fee (`baseFeeAmount`) | Absolute, in token units | Payment Processor fee recipient |
| Acquiring Fee | Percentage of principal | Registered Acquirer (if Acquirer ID is non-zero) |

The acquiring fee is applied only when the payment includes a non-zero Acquirer ID. Payments without an Acquirer MUST use the **Zero-UUID** to suppress the acquiring fee component without reverting.

### 9.4 Service Provider Discovery

An Acquirer who wishes to be discoverable by wallet users MAY publish their information to the basic-data-server, where they appear as a **Service Provider**. This opt-in mechanism allows wallets to present users with a curated list of available providers, enabling selection based on fee rates, reputation, or other criteria.

---

## 10. Settlement Contract — State Variables

The following state variables constitute the minimal persistent state that a conformant Settlement Contract MUST maintain.

```solidity
/// @notice Minimum absolute fee (in token units) charged on every transfer.
uint256 public baseFeeAmount;

/// @notice Maximum acquiring fee percentage an acquirer may configure.
uint256 public maxAcquiringFee;

/// @notice Internal token balances per participant.
/// Mapping: token address => participant address => amount.
mapping(address => mapping(address => uint256)) public balances;

/// @notice Registry of consumed Binding Signature hashes.
/// A hash present here MUST cause the transaction to revert.
mapping(bytes32 => bool) public usedHashes;

/// @notice Registry mapping Acquirer IDs to wallet addresses.
mapping(bytes16 => address) public acquirerWallets;

/// @notice Fee percentage charged by each registered acquirer.
mapping(address => uint256) public acquiringFeePercent;

/// @notice Set of ERC-2612-compliant tokens accepted for acquirer registration.
mapping(IERC20Permit => bool) public acquiringAllowedTokens;

/// @notice Price (in token units) required to register as a new acquirer.
uint256 public acquiringPrice;
```

**Notes:**

- `baseFeeAmount` provides a floor ensuring every transaction covers at minimum the cost of on-chain execution. Administrators SHOULD set this to reflect prevailing gas costs.
- `maxAcquiringFee` is a ceiling applied at acquirer registration. Acquirers MAY set their fee anywhere between zero and this ceiling.
- `balances` uses a two-level mapping (`token → participant → amount`) to support multiple token types.
- `usedHashes` stores the EIP-712 digest of each processed Binding Signature. Once recorded, any subsequent submission presenting the same hash MUST revert.
- `acquirerWallets` uses `bytes16` (UUID) keys. The canonical term for this key is **Acquirer ID**.

---

## 11. Settlement Contract — Events

A conformant Settlement Contract MUST emit the following events.

```solidity
/// @notice Emitted when a participant withdraws their accumulated balance.
event Withdrawal(
    address indexed owner,
    address indexed beneficiary,
    uint256 amount
);

/// @notice Emitted upon successful execution of a permitted token transfer.
/// @param orderReference The 32-byte field: 16-byte Order Reference || 16-byte Acquirer ID.
event PermittedTransfer(
    bytes32 indexed domainSeparator,
    address indexed token,
    address indexed payer,
    address recipient,
    uint256 value,
    uint256 fee,
    bytes32 orderReference
);

/// @notice Emitted when a new acquirer is successfully registered.
event AcquirerCreated(
    bytes16 indexed acquirerId,
    address indexed wallet,
    uint256 feePercent
);

/// @notice Emitted when an acquirer updates their fee percentage.
event AcquiringFeeUpdated(
    address indexed acquiring,
    uint256 feePercent
);

/// @notice Emitted when an acquiring fee is generated on a payment.
event CommissionGenerated(
    bytes16 indexed acquirerId,
    uint256 amount
);
```

**Notes:**

- `PermittedTransfer` is the primary event consumed by transfer-history for session reconciliation. The `orderReference` field MUST match the `ref` value submitted in the `TransferRequest` payload.
- `AcquirerCreated` uses `feePercent` — not `feeValue` — to clarify that this parameter represents a rate.
- The Zero-UUID MUST never be registerable as an acquirer wallet. The contract MUST enforce this.

---

## 12. Settlement Contract — Fee Model

### 12.1 Overview

Every payment is subject to two potentially applicable fee components:

| Fee Component | Type | Applicability |
| ------------- | ---- | ------------- |
| Base Fee (`baseFeeAmount`) | Absolute, in token units | Applied to every transfer unconditionally. |
| Acquiring Fee | Percentage of the principal, as set by the acquirer | Applied only when the payment includes a non-zero Acquirer ID. |

When no acquirer is involved, the caller MUST pass the **Zero-UUID** (`0x00000000000000000000000000000000`) as the `acquirerId`. The Zero-UUID suppresses the Acquiring Fee component entirely without reverting.

### 12.2 `calculateFees`

Computes the total fee to be added on top of a given principal amount.

```solidity
function calculateFees(
    uint256 amount,
    bytes16 acquirerId
) public view returns (uint256 totalFee)
```

| Parameter | Type | Description |
| --------- | ---- | ----------- |
| `amount` | `uint256` | The principal amount intended to reach the beneficiary, in token units. |
| `acquirerId` | `bytes16` | The Acquirer ID. Pass Zero-UUID if no acquirer is involved. |

**Returns:** `totalFee` — the absolute fee amount that must be added to `amount` to yield the correct `PermitParams.value`.

### 12.3 `breakdownTransferAmount`

Decomposes a total amount (principal plus fees) into its constituent parts.

```solidity
function breakdownTransferAmount(
    uint256 totalWithFees,
    bytes16 acquirerId
) public view returns (uint256 principalAmount, uint256 totalFees)
```

| Parameter | Type | Description |
| --------- | ---- | ----------- |
| `totalWithFees` | `uint256` | The total token amount inclusive of fees. The value in the permit's `value` field. |
| `acquirerId` | `bytes16` | The Acquirer ID. Pass Zero-UUID if no acquirer is involved. |

**Returns:** `principalAmount` — amount credited to beneficiary; `totalFees` — fees distributed to processor and acquirer.

---

## 13. Settlement Contract — Functions

### 13.1 `transferWithPermit`

Executes a complete stablecoin payment: verifies both signatures, collects the full amount from the payer, distributes fees, and credits the principal to the beneficiary.

```solidity
function transferWithPermit(
    IERC20Permit token,
    address tokenOwner,
    uint256 amount,
    uint256 deadline,
    uint8 v1, bytes32 r1, bytes32 s1,
    uint8 v2, bytes32 r2, bytes32 s2,
    address recipient,
    bytes32 ref
) external
```

| Parameter | Type | Description |
| --------- | ---- | ----------- |
| `token` | `IERC20Permit` | The ERC-2612-compliant stablecoin to transfer from. |
| `tokenOwner` | `address` | The token holder whose permit authorises the transfer. |
| `amount` | `uint256` | Total amount inclusive of all fees. MUST equal `value` in the signed permit. |
| `deadline` | `uint256` | Permit expiry. Forwarded to `token.permit()`. |
| `v1, r1, s1` | signature | **Permit Signature** — validated by the ERC-2612 token contract. |
| `v2, r2, s2` | signature | **Binding Signature** — validated by the Settlement Contract. |
| `recipient` | `address` | Beneficiary address. Receives principal after fee deduction. |
| `ref` | `bytes32` | 32-byte field: 16-byte Order Reference \|\| 16-byte Acquirer ID. |

**The function MUST revert if:** Binding Signature digest is in `usedHashes` · Binding Signature signer not authorised · `deadline ≤ block.timestamp` · `token.permit()` reverts · `tokenOwner` balance less than `amount`.

**Upon successful execution:** Binding Signature digest recorded in `usedHashes` · `amount` transferred from `tokenOwner` · fees distributed · principal credited to `recipient` · `PermittedTransfer` emitted · `CommissionGenerated` emitted if acquirer fee applies.

### 13.2 `buyAcquiringPack`

Registers a new Acquirer by accepting a fee payment. The payer and the account being registered do not need to be the same address.

```solidity
function buyAcquiringPack(
    IERC20Permit token,
    address payer,
    address acquiring,
    uint256 feePercent,
    uint256 price,
    uint256 deadline,
    uint8 v1, bytes32 r1, bytes32 s1,
    uint8 v2, bytes32 r2, bytes32 s2
) public
```

| Parameter | Type | Description |
| --------- | ---- | ----------- |
| `token` | `IERC20Permit` | Token used to pay the registration fee. Must be in `acquiringAllowedTokens`. |
| `payer` | `address` | Token holder paying the fee. May differ from `acquiring`. |
| `acquiring` | `address` | Wallet to be registered as Acquirer. Must not already be registered. |
| `feePercent` | `uint256` | Acquiring fee percentage. MUST NOT exceed `maxAcquiringFee`. |
| `price` | `uint256` | Registration fee in token units. MUST equal current `acquiringPrice`. |
| `deadline` | `uint256` | Permit expiry. |
| `v1, r1, s1` | signature | **Permit Signature** — validated by the ERC-2612 token contract. |
| `v2, r2, s2` | signature | **Binding Signature** — validated by the Settlement Contract. |

**The function MUST revert if:** `token` not in `acquiringAllowedTokens` · `feePercent` exceeds `maxAcquiringFee` · `price` does not equal `acquiringPrice` · `acquiring` already registered · Binding Signature digest in `usedHashes` · Binding Signature signer not authorised · `token.permit()` reverts · `payer` balance less than `price`.

**Upon successful execution:** Binding Signature digest recorded in `usedHashes` · `price` transferred from `payer` to fee recipient · new `bytes16` Acquirer ID assigned and recorded in `acquirerWallets` · `acquiring` recorded in `acquiringFeePercent` · `AcquirerCreated` emitted.

### 13.3 `getBalances` (single token)

```solidity
function getBalances(
    address token,
    address[] calldata users
) external view returns (uint256[] memory balances)
```

Returns internal balances for a list of participants in a single token. Response array is parallel to `users`.

### 13.4 `getBalances` (multi-token overload)

```solidity
function getBalances(
    address[] calldata tokens,
    address[] calldata users
) external view returns (uint256[][] memory balances)
```

Returns internal balances for multiple participants across multiple tokens. `balances[i][j]` is the internal balance of `users[j]` for `tokens[i]`.

### 13.5 `getAcquiringWallet`

```solidity
function getAcquiringWallet(
    bytes16 acquirerId
) public view returns (address)
```

Resolves the wallet address registered under a given Acquirer ID. Returns the zero address if not registered.

---

## 14. Validation Rules

A compliant Payment Processor MUST enforce all of the following checks before broadcasting a transaction. Any failure MUST result in immediate rejection with an appropriate error code. Partial processing is not permitted.

### 14.1 Structural Validation

- All REQUIRED fields MUST be present and non-empty.
- All hex strings MUST have a `0x` prefix and consist exclusively of hexadecimal digits (`0`–`9`, `a`–`f`). Uppercase hex MUST be normalised to lowercase before processing.
- Address fields MUST be exactly 42 characters (`0x` + 40 hex digits).
- `bytes32` fields MUST be exactly 66 characters (`0x` + 64 hex digits).
- The `v` field MUST be an integer. Values of `0` or `1` MUST be normalised to `27` or `28` respectively. All other values outside `{0, 1, 27, 28}` MUST be rejected.
- Numeric fields (`value`, `nonce`, `deadline`, `feeValue`) MUST represent non-negative integers within the `uint256` range (0 to 2²⁵⁶ − 1).

### 14.2 Semantic Validation

- The `deadline` field in `permitParams` MUST be strictly greater than the processor clock at time of receipt. Processors SHOULD apply a configurable tolerance (e.g., 30 seconds) to account for clock skew.
- The `nonce` field in `permitParams` MUST match the current on-chain permit nonce of the `owner` address on the specified token contract.
- For `TransferRequest`, the `payloadId` MUST correspond to a known, unprocessed checkout session held by the processor.
- For `TransferRequest`, the `ref` field MUST be present. It is formed by the 16-byte Order Reference concatenated with the 16-byte Acquirer ID. Missing parts MUST be filled with zeros.
- For `BuyAcquiringPackRequest`, `feeValue` MUST equal `permitParams.value`.

### 14.3 Cryptographic Validation

- The signer recovered from `payWithPermitSig` MUST equal `permitParams.owner` for both payload types.
- The processor SHOULD verify `permitSig` locally using the token contract's EIP-712 domain separator before broadcasting, to avoid on-chain reversal.
- The `hash` field in each `ERC20RelayerSig` MUST equal the digest that the processor recomputes from the accompanying parameters. Mismatches MUST be rejected.

---

## 15. Submission Flow

### 15.1 Payment Transfer — Step-by-Step

1. **Checkout Engine** issues a payment session: `ref` (16-byte Order Reference), `beneficiary`, `token`, principal amount, `acquirerId`, and `deadline`.
2. **Client Wallet** fetches the current ERC-2612 nonce for the `owner` from the token contract or wallet-gateway.
3. **Client Wallet** calls `calculateFees(principal, acquirerId)` to determine the fee. Computes `permitValue = principal + fee`.
4. **Client Wallet** constructs `PermitParams`: `owner`, `spender` (Settlement Contract), `value` = `permitValue`, `nonce`, `deadline`.
5. **Client Wallet** signs the ERC-2612 `Permit` typed-data using EIP-712 against the **token contract's domain separator** to produce `permitSig`.
6. **Client Wallet** constructs the 32-byte `ref`: `orderReference (16 bytes) || acquirerId (16 bytes)`. Missing parts filled with zeros.
7. **Client Wallet** constructs `PayWithPermitParams`: `token`, `beneficiary`, `ref`, `permitParams`.
8. **Client Wallet** signs `PayWithPermitParams` using EIP-712 against the **Settlement Contract's domain separator** to produce `payWithPermitSig`.
9. **Client Wallet** generates a `payloadId` (UUID) and stores it for idempotency tracking.
10. **Client Wallet** assembles the `TransferRequest` and submits it to the wallet-gateway over HTTPS.
11. **Payment Processor** validates the payload per Section 14 and, on success, broadcasts the settlement transaction.
12. **Settlement Contract** executes `permit()`, `transferFrom()`, and fee distribution atomically. The processor monitors the transaction and updates the session status accordingly.

### 15.2 Acquirer Registration — Step-by-Step

1. **Client Wallet** fetches `acquiringPrice` from the Settlement Contract or wallet-gateway.
2. **Client Wallet** fetches the ERC-2612 nonce for the `owner` from the token contract or wallet-gateway.
3. **Client Wallet** constructs `PermitParams`: `owner`, `spender` (Settlement Contract), `value` = `acquiringPrice`, `nonce`, `deadline`.
4. **Client Wallet** signs the ERC-2612 `Permit` typed-data against the **token contract's domain separator** to produce `permitSig`.
5. **Client Wallet** constructs `BuyAcquiringPackPermitParams`: `token`, `feeValue` = `acquiringPrice`, `acquiring`, `permitParams`.
6. **Client Wallet** signs `BuyAcquiringPackPermitParams` using EIP-712 against the **Settlement Contract's domain separator** to produce `payWithPermitSig`.
7. **Client Wallet** assembles the `BuyAcquiringPackRequest` and submits it to the wallet-gateway over HTTPS.
8. **Payment Processor** validates the payload per Section 14 and broadcasts the settlement transaction.
9. **Settlement Contract** executes `permit()`, collects the fee, registers the `acquiring` address with a randomly generated Acquirer ID, and emits `AcquirerCreated`.

---

## 16. End-to-End Payment Flow

### 16.1 Phase A — Session Creation

1. The Merchant Server sends a charge creation request to the **core-checkout-engine** over a mutually authenticated TLS session, including payment amount, accepted token, beneficiary address, and order metadata.
2. The core-checkout-engine generates a unique 16-byte Order Reference, a Payload ID, and a `deadline`. A new charge session is created in the awaiting-payment state.
3. The core-checkout-engine returns a Checkout Widget URL and an Ephemeral Token to the Merchant Server, which presents it to the payer via QR code, deep-link, or redirect.

### 16.2 Phase B — Payload Construction and Signing

4. The Client Wallet redeems the Ephemeral Token against the **checkout-public-widget** to retrieve session parameters. The Ephemeral Token is invalidated upon first use.
5. The wallet fetches the payer's ERC-2612 permit nonce and fee amount.
6. The wallet constructs and signs `PermitParams` using EIP-712 (Permit Signature).
7. The wallet constructs and signs `PayWithPermitParams` using EIP-712 (Binding Signature).

### 16.3 Phase C — Submission and Off-Chain Validation

8. The wallet assembles the `TransferRequest` and submits it to the **wallet-gateway**. A WebSocket connection is maintained for status updates.
9. The **broadcast-service** enqueues the submission; the wallet-gateway acknowledges receipt.
10. The Payment Processor performs structural, semantic, and cryptographic validation per Section 14. Any failure causes immediate rejection with a categorised error.

### 16.4 Phase D — On-Chain Settlement

11. The **broadcast-submitter** broadcasts the transaction by calling `transferWithPermit` on the Settlement Contract, paying gas from the Relayer's account.
12. The Settlement Contract verifies the Binding Signature, checks `usedHashes`, calls `permit()`, and grants itself an allowance.
13. The Settlement Contract calls `transferFrom`, distributes fees, credits the principal to the beneficiary, records the Binding Signature hash in `usedHashes`, and emits `PermittedTransfer` (and `CommissionGenerated` if applicable). All operations are atomic.
14. The broadcast-service delivers a status update to the wallet confirming the transaction was broadcast without revert. **This does not represent final settlement.**

### 16.5 Phase E — Confirmation and Reconciliation

15. The **transfer-history** service observes the `PermittedTransfer` event. After a configurable number of block confirmations, it publishes the confirmed event to the core-checkout-engine.
16. The core-checkout-engine matches the event's `orderReference` against its session registry and marks the charge as settled.
17. The core-checkout-engine dispatches a settlement webhook to the Merchant Server.
18. The **balance-and-history** service delivers a final settlement notification to the wallet over WebSocket.

---

## 17. Error Handling

A compliant Payment Processor MUST communicate validation failures clearly and without ambiguity.

| Error Category | Description |
| -------------- | ----------- |
| `STRUCTURAL_ERROR` | A required field is absent, of the wrong type, or violates encoding constraints. The submission MUST be rejected immediately without any on-chain activity. |
| `SEMANTIC_ERROR` | All fields are structurally valid but a business rule is violated (e.g., expired deadline, nonce mismatch, unknown `payloadId`). The submission MUST be rejected with a descriptive error indicating the violated rule. |
| `CRYPTOGRAPHIC_ERROR` | A signature cannot be verified or the recovered signer does not match the expected owner. The submission MUST be rejected. The processor MUST NOT disclose details that could aid signature forgery. |
| `BROADCAST_ERROR` | The payload passed all validation checks but the on-chain transaction reverted or could not be submitted. The processor MUST record the failure, return an appropriate error to the client, and MUST NOT mark the session as settled. |

---

## 18. Security Model

### 18.1 Threat Model

| Threat | Mitigation |
| ------ | ---------- |
| Forged payment commitment | ECDSA signature; only the payer's private key can produce a valid Permit or Binding Signature. |
| Replay of a valid signature | `usedHashes` registry in the Settlement Contract; ERC-2612 nonce on the token contract. |
| Parameter substitution (change recipient, amount, or token after signing) | Binding Signature covers all operation parameters; any modification invalidates the signature. |
| Processor submitting payment to wrong beneficiary | Beneficiary is embedded in the Binding Signature; alteration invalidates the signature. |
| Expired permit executed after deadline | Settlement Contract enforces `block.timestamp ≤ deadline`; token contract enforces the same. |
| Session replay (same payloadId submitted twice) | Processor enforces single-use payloadId at the off-chain validation layer. |
| Malicious Relayer double-spending | Impossible without a valid Binding Signature; the Relayer holds no signing authority. |
| Merchant server impersonation | mTLS mutual authentication between merchant servers and core-checkout-engine. |
| Ephemeral Token interception | Short lifetime and single-use invalidation; contains no signing authority. |

### 18.2 Dual-Signature Binding

The **Permit Signature** authorises a specific token contract to grant an allowance to the Settlement Contract. Its scope is limited to the allowance grant; it does not determine where funds go or how much is taken in fees.

The **Binding Signature** authorises the processor to execute a specific operation — including the token, the beneficiary, the amount, the fee structure, and the Order Reference — against the Settlement Contract. Without a valid Binding Signature, the Settlement Contract MUST revert.

Neither signature alone is sufficient to execute a payment. A valid Permit Signature without a valid Binding Signature results in an allowance with no execution. A valid Binding Signature without a valid Permit Signature results in a settlement attempt with no allowance.

### 18.3 Replay Protection

Replay protection is enforced at two independent layers:

- **ERC-2612 nonce** — the token contract tracks a per-owner nonce. Each issued permit increments the nonce, invalidating all previously issued permits for that owner.
- **`usedHashes` registry** — the Settlement Contract records the EIP-712 digest of every consumed Binding Signature. Any subsequent transaction presenting the same digest MUST revert.

Both controls MUST be active in a conformant implementation. They are complementary, not redundant.

### 18.4 Payer-Agnostic Execution and its Implications

The payer-agnostic model enables the gasless relayer model. Any party in possession of both valid signatures can submit the transaction. This is acceptable because:

1. the signatures bind the payment to specific parameters, including the beneficiary and the amount;
2. the `usedHashes` registry ensures the same signatures cannot be used more than once; and
3. submitting a valid transaction on behalf of a payer is not harmful to the payer — it is precisely what they authorised.

Clients MUST submit payloads over TLS and MUST NOT persist signed payloads in environments accessible to untrusted parties.

### 18.5 Ephemeral Token Security

The Ephemeral Token issued by the checkout-public-widget is a time-bounded, single-use credential:

- **Short lifetime**: the token MUST expire within a processor-configured window, RECOMMENDED to be no longer than 5 minutes.
- **Single use**: the token MUST be invalidated upon first successful redemption.
- **No signing authority**: possession of an Ephemeral Token does not enable payment. The payer's private key is still required for both signatures.

### 18.6 mTLS for Merchant Communication

Communication between Merchant Servers and the core-checkout-engine MUST be protected by mutual TLS. Both parties present certificates issued by the ca-server. JWT bearer tokens obtained via the login-server provide application-layer authorisation within the authenticated TLS session.

### 18.7 Administrative Privilege Boundaries

The Administrator MUST be able to set `baseFeeAmount`, `maxAcquiringFee`, `acquiringPrice`, and the fee recipient address.

The Administrator MUST NOT be able to:

- transfer tokens held in the contract's balance on behalf of any participant;
- modify the `usedHashes` registry; or
- alter the registered wallet address or fee percentage of an existing Acquirer without their consent.

This constraint MUST be enforced at the contract code level and verified by independent audit.

---

## 19. Conformance

### 19.1 Fully Compliant Stack

A **fully conformant** Stablecoin Stack deployment MUST include:

- a deployed Settlement Contract satisfying all requirements of Sections 10–13;
- a Checkout Engine satisfying all requirements of Section 5.2;
- a Broadcast Layer satisfying all requirements of Sections 5.3 and 14;
- a transfer-history service that monitors and reports confirmed on-chain events; and
- a Basic Data Service accessible to wallets and other components.

A fully conformant Client Wallet MUST satisfy all requirements stated in Section 5.6.

### 19.2 Partial Conformance

Components MAY be implemented and deployed independently. A conformant Settlement Contract MAY be used with a non-reference Checkout Engine. A conformant wallet MAY interact with any Settlement Contract and Broadcast Layer that satisfies the specified interfaces. Partial conformance MUST be declared explicitly by the implementer, with clear indication of which sections are and are not satisfied.

### 19.3 Future Specifications

The following companion specifications are planned and will reference this document:

| Planned ID | Subject |
| ---------- | ------- |
| SSF-SPEC-002 | Checkout Engine API — merchant-facing session and charge management |
| SSF-SPEC-003 | Broadcast Layer Protocol — WebSocket protocol between wallet and wallet-gateway |
| SSF-SPEC-004 | wallet-gateway Interface |
| SSF-SPEC-005 | broadcast-service Interface |
| SSF-SPEC-006 | broadcast-submitter Interface |
| SSF-SPEC-007 | transfer-history Interface |
| SSF-SPEC-008 | basic-data-server Interface |
| SSF-SPEC-009 | Client Wallet Certification |

---

## 20. Future Work

**Offline Transaction Signing** — Enabling payers to construct and sign a payment commitment without a live network connection. This is a priority for populations with limited or intermittent mobile network access.

**Multi-Network Support** — Extending conformance to non-EVM networks with equivalent programmable token standards.

**Wallet Certification Programme** — A conformance test suite and certification programme for third-party wallet implementations.

**Acquirer Incentive Structures** — Further specification of fee tier and pricing models available to Acquirers.

**`ref` Field Evolution** — The `ref` field currently concatenates two values (Order Reference and Acquirer ID) that serve distinct and non-dependent purposes. A future version will separate these into independent fields.

---

## 21. Versioning

This specification follows [Semantic Versioning 2.0.0](https://semver.org).

| Segment | Meaning |
| ------- | ------- |
| **MAJOR** | A backwards-incompatible change to any interface, encoding, security model, or conformance requirement. |
| **MINOR** | Addition of new components, flows, or requirements that do not break existing conformant implementations. |
| **PATCH** | Editorial corrections, clarifications, and non-normative changes that do not affect conformance. |

All future companion specifications MUST declare the version of SSF-SPEC-001 to which they conform.

---

## 22. Change Log

| Version | Date | Author | Summary |
| ------- | ---- | ------ | ------- |
| 1.0.0 | 2026-03-04 | Adalton Reis | Initial release. Unification of SSF-SPEC-000 (system overview), pre-unification SSF-SPEC-001 (submission protocol), and SSF-SPEC-002 (Settlement Contract) into a single specification under the Layered Docs structure. |

---

## 23. License

Copyright © 2026 Stablecoin Stack Foundation. All rights reserved.

This specification is published under the [Apache License, Version 2.0](https://www.apache.org/licenses/LICENSE-2.0). Unless required by applicable law or agreed to in writing, material distributed under the License is distributed on an "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
