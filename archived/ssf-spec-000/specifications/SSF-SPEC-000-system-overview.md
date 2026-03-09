# SSF-SPEC-000 — The Stablecoin Stack: System Overview and Architecture

| Field | Value |
|-------|-------|
| **Document ID** | SSF-SPEC-000 |
| **Title** | The Stablecoin Stack: System Overview and Architecture |
| **Version** | 1.0.0 |
| **Status** | Draft |
| **Date** | 2026-03-04 |
| **Supersedes** | — |
| **Author(s)** | [Author Name] \<reis@stablecoinstack.org\> |
| **Reviewers** |  |
| **Organization** | Stablecoin Stack Foundation |
| **Contact** | contact@stablecoinstack.org |
| **Public Address** | `0x0000000000000000000000000000000000000000` |
| **License** | Apache License 2.0 |

> **Non-normative context:** The motivation, design philosophy, system architecture, component descriptions, acquiring model, and end-to-end payment flow are documented in the supporting materials under `overview/` and `core-concepts/`. This document contains only the normative requirements: the security model, the conformance conditions, and the authoritative references. Readers unfamiliar with the system should read the supporting materials first.

---

## Table of Contents

1. [Abstract](#1-abstract)
2. [Normative References](#2-normative-references)
3. [Security Model](#3-security-model)
   - 3.1 [Threat Model](#31-threat-model)
   - 3.2 [Dual-Signature Binding](#32-dual-signature-binding)
   - 3.3 [Replay Protection](#33-replay-protection)
   - 3.4 [Payer-Agnostic Execution](#34-payer-agnostic-execution)
   - 3.5 [Ephemeral Token Requirements](#35-ephemeral-token-requirements)
   - 3.6 [Merchant Channel Security](#36-merchant-channel-security)
   - 3.7 [Administrative Privilege Boundaries](#37-administrative-privilege-boundaries)
4. [Conformance](#4-conformance)
   - 4.1 [Fully Conformant Stack](#41-fully-conformant-stack)
   - 4.2 [Partial Conformance](#42-partial-conformance)
   - 4.3 [Companion Specifications](#43-companion-specifications)
5. [Change Log](#5-change-log)
6. [License](#6-license)

---

## 1. Abstract

This document is the root specification of the **Stablecoin Stack** — an open architecture for processing stablecoin payments on Ethereum-compatible networks using ERC-2612 permit-based authorisation. It establishes the normative security requirements and conformance conditions that apply to the system as a whole. Individual component specifications are listed in Section 4.3.

The Stablecoin Stack is built on the proposition that cryptographic signatures are a superior foundation for payment authorisation compared to identity-based delegation models. All normative requirements in this specification and its companions follow from this proposition.

---

## 2. Normative References

The following documents are normative references for this specification. Where this specification refers to a standard by name, the requirements of that standard apply.

| Reference | Description |
|-----------|-------------|
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
| [SSF-SPEC-001](../../01-instant-payment-with-permitted-token-transfer-submission/specifications/SSF-SPEC-001-submission.md) | Instant Payment With Permitted Token Transfer — Submission |
| [SSF-SPEC-002](../../02-the-settlement-contract-canonical/specifications/SSF-SPEC-002-settlement-contract.md) | The Settlement Contract — Canonical |

The key words **MUST**, **MUST NOT**, **REQUIRED**, **SHALL**, **SHOULD**, **RECOMMENDED**, **MAY**, and **OPTIONAL** in this document are to be interpreted as described in RFC 2119.

Terminology used in this specification is defined in the [Glossary](../reference/glossary.md).

---

## 3. Security Model

### 3.1 Threat Model

A conformant Stablecoin Stack deployment MUST be designed to resist the following threat classes. The stated mitigations are normative requirements.

| Threat | Required Mitigation |
|--------|-------------------|
| Forged payment commitment | ECDSA over secp256k1; only the holder of the payer's private key can produce a valid Permit Signature or Binding Signature. |
| Replay of a valid signature | `usedHashes` registry in the Settlement Contract MUST record all consumed Binding Signature digests. ERC-2612 nonce on the token contract invalidates reused Permit Signatures. Both controls MUST be active independently. |
| Parameter substitution attack | The Binding Signature MUST cover all operation parameters (token, beneficiary, amount, ref, deadline). Any modification to any parameter invalidates the signature. |
| Expired permit executed after deadline | The Settlement Contract MUST enforce `block.timestamp ≤ deadline`. The token contract enforces the same condition independently. |
| Session replay | The Payment Processor MUST enforce single-use `payloadId` validation at the off-chain layer. |
| Relayer double-spend or substitution | The Relayer holds no signing authority. Without a valid Binding Signature, the Settlement Contract MUST revert. |
| Merchant server impersonation | mTLS MUST be enforced on all communication between Merchant Servers and the core-checkout-engine. |
| Ephemeral token interception | Ephemeral Tokens MUST be short-lived and single-use. Possession of an Ephemeral Token MUST NOT confer signing authority. |
| Administrator fund manipulation | The Settlement Contract MUST enforce that Administrator privileges do not include any mechanism to transfer participant balances. |

### 3.2 Dual-Signature Binding

Every payment and acquirer registration operation in the Stablecoin Stack MUST be authorised by exactly two independent ECDSA signatures.

**The Permit Signature** is an ERC-2612 typed-data signature covering the permit parameters: `owner`, `spender`, `value`, `nonce`, and `deadline`. It is validated by the ERC-2612 token contract inside its `permit()` function. Its scope is limited to granting a token allowance; it does not determine where funds go or how fees are calculated.

**The Binding Signature** is an EIP-712 typed-data signature covering the full operation parameters. For a payment transfer, this includes the token address, beneficiary address, amount, order reference (`ref`), and deadline. It is validated by the Settlement Contract. Without a valid Binding Signature, the Settlement Contract MUST revert.

The two signatures MUST be validated independently. A Payment Processor MUST NOT accept a submission in which either signature is absent, invalid, or inconsistent with the accompanying parameters.

Neither signature alone is sufficient to execute a payment:
- A valid Permit Signature without a valid Binding Signature yields a token allowance with no execution path.
- A valid Binding Signature without a valid Permit Signature yields a settlement attempt with no allowance — the `transferFrom` call will fail.

### 3.3 Replay Protection

Replay protection MUST be enforced at two independent layers, both of which MUST be active in a conformant implementation.

**Layer 1 — ERC-2612 nonce.** The token contract maintains a per-owner nonce. A permit signed with a given nonce is invalidated once a permit with that nonce is consumed on-chain, because the on-chain nonce is incremented. This prevents a Permit Signature from being reused.

**Layer 2 — `usedHashes` registry.** The Settlement Contract MUST record the EIP-712 digest of every consumed Binding Signature in the `usedHashes` mapping. Any subsequent transaction presenting a digest already present in this mapping MUST revert immediately, before any token transfer occurs. This prevents a Binding Signature from being reused even if a new permit with identical parameters were issued.

The two layers are complementary. Implementations MUST NOT rely on one to compensate for the absence of the other.

### 3.4 Payer-Agnostic Execution

The Settlement Contract MUST NOT validate the identity of `msg.sender` as a condition for accepting a payment. The validity of a payment is determined solely by the cryptographic signatures it carries, not by which account submits the transaction. This is the mechanism that enables the gasless Relayer model.

The security properties of this design are:

1. The signatures bind the payment to specific parameters (beneficiary, amount, token, deadline, ref). Submitting the transaction cannot alter any of these.
2. The `usedHashes` registry ensures the signatures cannot be reused.
3. Submitting a valid transaction on behalf of a payer is not harmful — it is precisely what the payer authorised.

The operational consequence is that **the confidentiality of signed payloads prior to submission is a security requirement**. Clients MUST submit payloads exclusively over TLS. Clients MUST NOT persist signed payloads in environments accessible to untrusted parties.

### 3.5 Ephemeral Token Requirements

The Ephemeral Token issued by the checkout-public-widget is a time-bounded, single-use credential that authorises retrieval of session payment details.

A conformant checkout-public-widget MUST enforce the following:

- **Expiry**: the token MUST expire within a processor-configured window. The RECOMMENDED maximum lifetime is 5 minutes.
- **Single use**: the token MUST be invalidated immediately upon first successful redemption.
- **Scope limitation**: the token MUST authorise only the retrieval of session payment details. It MUST NOT confer any signing authority, payment capability, or access to any other session or resource.

### 3.6 Merchant Channel Security

All communication between Merchant Servers and the core-checkout-engine MUST be protected by mutual TLS (mTLS) in accordance with RFC 8705. Both the server and the client MUST present certificates issued by the processor's ca-server.

Application-layer authorisation within the mTLS session MUST be enforced using JWT bearer tokens issued by the login-server. TLS authentication and JWT authorisation are independent controls and MUST both be enforced.

Connections that do not present a valid client certificate MUST be rejected before any application-layer processing occurs.

### 3.7 Administrative Privilege Boundaries

The Settlement Contract Administrator (the Payment Processor account) MUST have the ability to configure the following operational parameters:

- `baseFeeAmount` — the minimum absolute fee per transfer;
- `maxAcquiringFee` — the ceiling on acquirer fee percentages;
- `acquiringPrice` — the registration fee for new acquirers; and
- the fee recipient address to which base fees are credited.

The Administrator MUST NOT have the ability to:

- transfer or encumber token balances held by the contract on behalf of any participant;
- modify the `usedHashes` registry;
- alter the registered wallet address or fee percentage of an existing Acquirer without their participation in a new signed transaction; or
- bypass the signature verification logic of any public function.

These constraints MUST be enforced at the contract code level. Compliance MUST be verified by independent security audit before a deployment is considered production-ready.

---

## 4. Conformance

### 4.1 Fully Conformant Stack

A **fully conformant** Stablecoin Stack deployment MUST include all of the following:

- a deployed **Settlement Contract** that satisfies all requirements of SSF-SPEC-002;
- a **Checkout Engine** (core-checkout-engine and checkout-public-widget) that satisfies the session lifecycle and Ephemeral Token requirements of this specification and SSF-SPEC-001;
- a **Broadcast Layer** (broadcast-gateway, broadcast-service, broadcast-submitter) that validates payloads per SSF-SPEC-001 Section 7, queues submissions, and broadcasts transactions on-chain;
- a **transfer-history** service that monitors the Settlement Contract for `PermittedTransfer` events and delivers confirmations to the Checkout Engine after the required block confirmation depth; and
- a **basic-data-server** instance that is accessible to compliant wallets and other system components.

A **fully conformant Client Wallet** MUST satisfy all wallet requirements stated in SSF-SPEC-001 and this document. In particular, a conformant wallet:

- MUST construct Permit Signatures and Binding Signatures using EIP-712 with the correct domain separator;
- MUST submit payloads exclusively over TLS;
- MUST NOT transmit the payer's private key or seed phrase to any remote service; and
- MUST NOT persist signed payloads in environments accessible to untrusted parties.

### 4.2 Partial Conformance

Components MAY be implemented and deployed independently. A conformant Settlement Contract MAY be used with a non-reference Checkout Engine, provided that Checkout Engine satisfies the session and Ephemeral Token requirements stated in this document. A conformant wallet MAY interact with any deployment whose Settlement Contract and Broadcast Layer satisfy the interfaces specified in SSF-SPEC-001 and SSF-SPEC-002.

Partial conformance MUST be declared explicitly by the implementer. The declaration MUST identify which specifications — and which versions of those specifications — are and are not satisfied.

### 4.3 Companion Specifications

The following companion specifications are part of the Stablecoin Stack specification series and provide detailed normative requirements for individual components.

| ID | Title | Status | Scope |
|----|-------|--------|-------|
| [SSF-SPEC-001](../../01-instant-payment-with-permitted-token-transfer-submission/specifications/SSF-SPEC-001-submission.md) | Instant Payment With Permitted Token Transfer — Submission | Draft | Payload structures, signing conventions, submission protocol, off-chain validation rules |
| [SSF-SPEC-002](../../02-the-settlement-contract-canonical/specifications/SSF-SPEC-002-settlement-contract.md) | The Settlement Contract — Canonical | Draft | On-chain interface: state variables, events, functions, dual-signature pattern, security requirements |

Additional specifications are planned for:

- the Checkout Engine merchant API;
- the Broadcast Layer WebSocket protocol; and
- the Basic Data Service API.

All companion specifications conform to this document and MUST declare the version of SSF-SPEC-000 to which they apply.

---

## 5. Change Log

| Version | Date | Author | Summary |
|---------|------|--------|---------|
| 1.0.0 | 2026-03-04 | [Author Name] | Initial release. |

---

## 6. License

Copyright © 2026 Stablecoin Stack Foundation. All rights reserved.

This specification is published under the [Apache License, Version 2.0](https://www.apache.org/licenses/LICENSE-2.0). Unless required by applicable law or agreed to in writing, material distributed under the License is distributed on an "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
