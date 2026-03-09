---
sidebar_position: 1
---

# SSF-SPEC-001 — Instant Payment With Permitted Token Transfer - Submission

| Field              | Value                                                      |
| ------------------ | ---------------------------------------------------------- |
| **Document ID**    | SSF-SPEC-001                                               |
| **Title**          | Instant Payment With Permitted Token Transfer - Submission |
| **Version**        | 1.0.0                                                      |
| **Status**         | Draft                                                      |
| **Date**           | 2026-03-04                                                 |
| **Supersedes**     | —                                                          |
| **Author(s)**      | Adalton Reis \<reis@stablecoinstack.org\>                  |
| **Reviewers**      | —                                                          |
| **Organization**   | Stablecoin Stack Foundation                                |
| **Contact**        | contact@stablecoinstack.org                                |
| **Public Address** | `0x0000000000000000000000000000000000000000`               |
| **License**        | Apache License 2.0                                         |

---

## Table of Contents

1. [Abstract](#1-abstract)
2. [Introduction](#2-introduction)
   - 2.1 [Purpose](#21-purpose)
   - 2.2 [Scope](#22-scope)
   - 2.3 [Normative References](#23-normative-references)
   - 2.4 [Conventions and Terminology](#24-conventions-and-terminology)
3. [Architecture Overview](#3-architecture-overview)
4. [Cryptographic Conventions](#4-cryptographic-conventions)
   - 4.1 [Signature Algorithm](#41-signature-algorithm)
   - 4.2 [Typed Data Signing (EIP-712)](#42-typed-data-signing-eip-712)
   - 4.3 [Signature Representation](#43-signature-representation)
5. [Data Structures](#5-data-structures)
   - 5.1 [Signature Object — ERC20RelayerSig](#51-signature-object--erc20relayersig)
   - 5.2 [ERC-2612 Permit Parameters — PermitParams](#52-erc-2612-permit-parameters--permitparams)
   - 5.3 [Payment Transfer Parameters — PayWithPermitParams](#53-payment-transfer-parameters--paywithpermitparams)
   - 5.4 [Acquirer Service Purchase Parameters — BuyAcquiringPackPermitParams](#54-acquirer-service-purchase-parameters--buyacquiringpackpermitparams)
6. [Submission Payloads](#6-submission-payloads)
   - 6.1 [Two-Signature Pattern](#61-two-signature-pattern)
   - 6.2 [Payment Transfer Payload — TransferRequest](#62-payment-transfer-payload--transferrequest)
   - 6.3 [Acquirer Service Purchase Payload — BuyAcquiringPackRequest](#63-acquirer-service-purchase-payload--buyacquiringpackrequest)
7. [Validation Rules](#7-validation-rules)
   - 7.1 [Structural Validation](#71-structural-validation)
   - 7.2 [Semantic Validation](#72-semantic-validation)
   - 7.3 [Cryptographic Validation](#73-cryptographic-validation)
8. [Submission Flow](#8-submission-flow)
   - 8.1 [Payment Transfer — Step-by-Step](#81-payment-transfer--step-by-step)
   - 8.2 [Acquirer Service Purchase — Step-by-Step](#82-acquirer-service-purchase--step-by-step)
9. [Error Handling](#9-error-handling)
10. [Security Considerations](#10-security-considerations)
    - 10.1 [Replay Protection](#101-replay-protection)
    - 10.2 [Deadline Enforcement](#102-deadline-enforcement)
    - 10.3 [Signature Scope](#103-signature-scope)
    - 10.4 [Address Verification](#104-address-verification)
    - 10.5 [Transport Security](#105-transport-security)
11. [Versioning](#11-versioning)
12. [Change Log](#12-change-log)
13. [License](#13-license)

---

## 1. Abstract

This specification defines the data structures, cryptographic conventions, and submission protocols that a compliant client must observe when initiating a stablecoin payment or acquiring a service entitlement through the Stablecoin Stack Foundation payment processing infrastructure. The mechanisms described herein operate over Ethereum-compatible networks and rely exclusively on ERC-2612 permit-based authorisations, enabling gasless, off-chain-signed token transfers that are settled on-chain by a designated relayer.

The present document covers one well-defined functional subset of the broader Stablecoin Stack architecture: the construction, signing, and submission of permitted transfer payloads. Adjacent concerns — the checkout engine, the merchant dashboard, the end-user wallet, and the on-chain fee distribution logic — are addressed in companion specifications.

---

## 2. Introduction

### 2.1 Purpose

This document provides a precise, implementation-neutral description of the interfaces and procedures required to submit a permitted token transfer to a compliant Stablecoin Stack payment processor. It is intended for:

- developers integrating a wallet or checkout front-end with the payment processor API;
- implementers building a compliant payment processor service; and
- auditors and reviewers assessing conformance of an existing implementation.

### 2.2 Scope

This specification covers:

- the structure of all payload types accepted by the payment processor;
- the cryptographic properties of every field, including signature schemes, encoding conventions, and byte-length constraints;
- the two primary submission flows: a stablecoin payment transfer and an acquirer service entitlement purchase; and
- the validation rules that a compliant processor must enforce before broadcasting a transaction.

This specification does not cover smart-contract internals, key-management procedures, network-layer transport security, or the checkout session lifecycle.

### 2.3 Normative References

| Reference  | Description                                             |
| ---------- | ------------------------------------------------------- |
| ERC-20     | Token Standard — Ethereum Improvement Proposal 20       |
| ERC-2612   | Permit Extension for EIP-20 Signed Approvals            |
| EIP-712    | Typed Structured Data Hashing and Signing               |
| SEC 1 v2.0 | Elliptic Curve Cryptography — SECG Standard             |
| ECDSA      | Elliptic Curve Digital Signature Algorithm (FIPS 186-4) |
| RFC 4648   | The Base16, Base32, and Base64 Data Encodings           |

### 2.4 Conventions and Terminology

The key words **MUST**, **MUST NOT**, **REQUIRED**, **SHALL**, **SHOULD**, **RECOMMENDED**, and **OPTIONAL** in this document are to be interpreted as described in RFC 2119.

| Term              | Definition                                                                                                                                                                                                                                                                                     |
| ----------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Payment Processor | A service that receives signed payloads, validates them, and broadcasts the corresponding on-chain transaction.                                                                                                                                                                                |
| Relayer           | The entity that holds a funded account and submits transactions on behalf of payers. May be the processor itself.                                                                                                                                                                              |
| Permit            | An ERC-2612 off-chain authorisation allowing a spender to transfer tokens on behalf of an owner.                                                                                                                                                                                               |
| Acquirer          | A registered participant entitled to a share of the processing fee.                                                                                                                                                                                                                            |
| Payload ID        | Client-generated identifier correlating this submission with a specific broadcasting intent. Both the client and the processor MUST use this ID to ensure request idempotency; the processor MUST validate that this ID is known, currently unprocessed, and unique to the broadcast session.  |
| Hex string        | A UTF-8 string with a `0x` prefix followed by an even number of lowercase hexadecimal digits.                                                                                                                                                                                                  |
| EVM address       | A 20-byte account identifier, represented as a 42-character hex string (`0x` + 40 hex digits).                                                                                                                                                                                                 |
| acquirerId        | A 16-byte acquirer identifier issued by the Settlement Contract.                                                                                                                                                                                                                               |
| acquiringPrice    | The amount that must be transferred to the Settlement Contract in order to register a new acquirer address, written in the smallest unit of the respective token.                                                                                                                              |

---

## 3. Architecture Overview

The Stablecoin Stack payment infrastructure is composed of four principal components that work in concert to deliver a complete, custody-free payment experience:

| Component             | Role within the stack                                                                                                                      |
| --------------------- | ------------------------------------------------------------------------------------------------------------------------------------------ |
| Checkout Engine       | Generates payment sessions, computes the precise token amount and deadline, and presents a signable payload to the payer.                  |
| Client Wallet         | Holds the payer's private key, constructs both EIP-712 typed-data signatures required by the processor, and submits the composite payload. |
| Payment Processor API | Receives the signed payload over HTTPS, validates all fields and signatures, and forwards the transaction to the relayer.                  |
| Settlement Contract   | On-chain logic that verifies the permit signature, executes the token transfer, and distributes fees to acquirers.                         |

This specification governs the interface between the **Client Wallet** and the **Payment Processor API** exclusively. The settlement contract ABI and the checkout engine session protocol are defined in separate documents.

---

## 4. Cryptographic Conventions

### 4.1 Signature Algorithm

All signatures in this specification are produced using the **Elliptic Curve Digital Signature Algorithm (ECDSA)** over the **secp256k1** curve, as specified in FIPS 186-4 and SEC 1 v2.0. This is the same scheme used natively by the Ethereum protocol for transaction and typed-data signing.

### 4.2 Typed Data Signing (EIP-712)

Every signature MUST be computed over a structured hash produced in accordance with **EIP-712**. This scheme binds each signature to a specific domain (chain identifier, contract address, and version) and to a well-typed data structure, preventing cross-domain and cross-type replay attacks.

A compliant signer MUST construct the digest as:

```
digest = keccak256(
    0x1901                    ||
    domainSeparator           ||
    hashStruct(message)
)
```

where `domainSeparator` encodes the `name`, `version`, `chainId`, and `verifyingContract` of the settlement contract.

### 4.3 Signature Representation

An ECDSA signature over a secp256k1 curve is represented by three components. Within this specification they are always carried as a structured object with the following fields:

| Field  | Type / Format                  | Required | Description                                                                                                                                                               |
| ------ | ------------------------------ | -------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `hash` | `bytes32` — 66-char hex string | REQUIRED | The EIP-712 digest that was signed. Encoded as `0x` followed by 64 lowercase hex characters.                                                                              |
| `v`    | `uint8` — integer              | REQUIRED | Recovery identifier. MUST be `27` or `28` (Ethereum convention). Some signers produce `0` or `1`; the processor MUST normalise by adding 27 if the value is less than 27. |
| `r`    | `bytes32` — 66-char hex string | REQUIRED | The `r` component of the ECDSA signature. Encoded as `0x` followed by 64 lowercase hex characters.                                                                        |
| `s`    | `bytes32` — 66-char hex string | REQUIRED | The `s` component of the ECDSA signature. Encoded as `0x` followed by 64 lowercase hex characters.                                                                        |

All hex strings throughout this specification MUST use the `0x` prefix and lowercase hexadecimal digits. Leading zero bytes MUST be preserved; the encoded string MUST always be the full expected length.

---

## 5. Data Structures

This section defines all structures accepted by the payment processor in a language-agnostic notation. Each structure is described with field names, types, encoding conventions, and normative constraints.

### 5.1 Signature Object — `ERC20RelayerSig`

A generic container for an ECDSA signature produced by an Ethereum-compatible signer. This structure is reused across all payload types.

| Field  | Type / Format                    | Required | Description                                                                                                                               |
| ------ | -------------------------------- | -------- | ----------------------------------------------------------------------------------------------------------------------------------------- |
| `hash` | `bytes32` — 66-char hex string   | REQUIRED | The EIP-712 message digest over which the signature was computed. Must be exactly 32 bytes (`0x` + 64 hex chars).                         |
| `v`    | `uint8` — integer (`27` or `28`) | REQUIRED | ECDSA recovery parameter. Processors MUST reject values outside `{0, 1, 27, 28}` and normalise `0` and `1` to `27` and `28` respectively. |
| `r`    | `bytes32` — 66-char hex string   | REQUIRED | First 32-byte component of the ECDSA signature. Must be exactly 32 bytes.                                                                 |
| `s`    | `bytes32` — 66-char hex string   | REQUIRED | Second 32-byte component of the ECDSA signature. Must be exactly 32 bytes.                                                                |

### 5.2 ERC-2612 Permit Parameters — `PermitParams`

Encapsulates the parameters of an ERC-2612 permit authorisation. These values are incorporated verbatim into the on-chain permit call executed by the settlement contract.

| Field      | Type / Format                        | Required | Description                                                                                                                                                         |
| ---------- | ------------------------------------ | -------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `owner`    | `address` — 42-char hex string       | REQUIRED | The Ethereum address of the token holder granting the approval. Must be a 20-byte EVM address (`0x` + 40 hex chars).                                                |
| `spender`  | `address` — 42-char hex string       | REQUIRED | The Ethereum address authorised to transfer tokens on behalf of the owner. Typically the settlement contract address.                                               |
| `value`    | `uint256` — decimal or hex string    | REQUIRED | The token amount approved for transfer, expressed in the token's smallest indivisible unit. Must be a non-negative integer.                                         |
| `nonce`    | `uint256` — decimal or hex string    | REQUIRED | The current ERC-2612 permit nonce of the owner on the token contract. The processor MUST verify this matches the on-chain nonce at submission time.                 |
| `deadline` | `uint256` — Unix timestamp (seconds) | REQUIRED | The expiry of the permit. The settlement contract will revert if `block.timestamp` exceeds this value.                                                              |

### 5.3 Payment Transfer Parameters — `PayWithPermitParams`

Defines the business-level parameters of a stablecoin payment. Used in conjunction with two signatures to form a complete payment submission payload.

| Field          | Type / Format                  | Required | Description                                                                                                                                                                      |
| -------------- | ------------------------------ | -------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `token`        | `address` — 42-char hex string | REQUIRED | The contract address of the ERC-2612-compliant stablecoin token to be transferred.                                                                                               |
| `beneficiary`  | `address` — 42-char hex string | REQUIRED | The merchant's receiving address. Funds are credited here after fee deduction by the settlement contract.                                                                        |
| `ref`          | `bytes32` — 66-char hex string | REQUIRED | An opaque 32-byte field, formed by the concatenation of a 16-byte reference that correlates this transfer to an upstream payment session and the `acquirerId`.                   |
| `permitParams` | `PermitParams` — nested object | REQUIRED | The ERC-2612 permit parameters as defined in Section 5.2. The `owner` field MUST match the address recovered from the permit signature.                                          |

### 5.4 Acquirer Service Purchase Parameters — `BuyAcquiringPackPermitParams`

Defines the parameters for purchasing an acquirer service entitlement. This flow enables a participant to register as an acquirer by pre-paying a fee denominated in a supported stablecoin.

| Field          | Type / Format                     | Required | Description                                                                                                                                                                            |
| -------------- | --------------------------------- | -------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `token`        | `address` — 42-char hex string    | REQUIRED | The contract address of the ERC-2612-compliant stablecoin token used to pay the acquirer fee.                                                                                          |
| `feeValue`     | `uint256` — decimal or hex string | REQUIRED | The fee amount to be deducted in favour of the `acquirer`. Whether the amount is expressed in the token's smallest unit or as a percentage depends on the smart contract's implementation. |
| `acquiring`    | `address` — 42-char hex string    | REQUIRED | The Ethereum address that will be registered as the acquirer upon successful on-chain settlement.                                                                                      |
| `permitParams` | `PermitParams` — nested object    | REQUIRED | The ERC-2612 permit parameters as defined in Section 5.2. The `value` field MUST equal `feeValue`.                                                                                     |

---

## 6. Submission Payloads

The payment processor exposes two distinct submission endpoints, each accepting a composite payload that bundles the operation parameters with two independent ECDSA signatures.

### 6.1 Two-Signature Pattern

Every submission payload carries exactly two signature objects:

| Signature field    | What it signs and why                                                                                                                                                                                                                                                                                    |
| ------------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `permitSig`        | An EIP-712 signature over the ERC-2612 `Permit` typed-data structure. It authorises the settlement contract to call `permit()` on the token contract, granting an allowance. Produced by the token holder (`owner`) and verified by the token contract on-chain.                                         |
| `payWithPermitSig` | An EIP-712 signature over the operation-specific parameters (`PayWithPermitParams` or `BuyAcquiringPackPermitParams`). It authorises the payment processor to submit this specific operation. Verified off-chain by the processor and on-chain by the settlement contract before executing the transfer. |

Separating these two signatures ensures that neither the payment processor nor the token contract can unilaterally reuse an authorisation in an unintended context.

### 6.2 Payment Transfer Payload — `TransferRequest`

Submitted to initiate a stablecoin payment from a payer to a merchant beneficiary. The processor validates this payload, broadcasts the transaction, and settles any applicable fees.

| Field                 | Type / Format                                 | Required | Description                                                                                                                                                                                            |
| --------------------- | --------------------------------------------- | -------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `payWithPermitParams` | `PayWithPermitParams` — nested object         | REQUIRED | Payment parameters as defined in Section 5.3.                                                                                                                                                          |
| `payWithPermitSig`    | `ERC20RelayerSig` — nested object             | REQUIRED | ECDSA signature over `payWithPermitParams`, produced by the payer. The processor MUST recover the signer address and verify it matches `permitParams.owner`.                                           |
| `permitSig`           | `ERC20RelayerSig` — nested object             | REQUIRED | ERC-2612 permit signature over the token's `Permit` typed-data, produced by the token holder. Forwarded verbatim to the settlement contract.                                                           |
| `payloadId`           | `string` — UUID or processor-issued opaque ID | REQUIRED | Client-generated identifier correlating this submission with a specific broadcasting intent. Both the client and the processor MUST use this ID to ensure request idempotency; the processor MUST validate that this ID is known, currently unprocessed, and unique to the broadcast session. |

### 6.3 Acquirer Service Purchase Payload — `BuyAcquiringPackRequest`

Submitted to purchase an acquirer entitlement. Upon settlement, the `acquiring` address specified in the parameters is registered on-chain with the corresponding service tier.

| Field                    | Type / Format                                  | Required | Description                                                                                                                                                              |
| ------------------------ | ---------------------------------------------- | -------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `buyAcquiringPackParams` | `BuyAcquiringPackPermitParams` — nested object | REQUIRED | Acquirer fee parameters as defined in Section 5.4.                                                                                                                       |
| `payWithPermitSig`       | `ERC20RelayerSig` — nested object              | REQUIRED | ECDSA signature over `buyAcquiringPackParams`, produced by the fee payer. The processor MUST recover the signer and verify it matches `permitParams.owner`.              |
| `permitSig`              | `ERC20RelayerSig` — nested object              | REQUIRED | ERC-2612 permit signature, produced by the token holder. Forwarded verbatim to the settlement contract. The signed `value` MUST equal `buyAcquiringPackParams.feeValue`. |

---

## 7. Validation Rules

A compliant payment processor MUST enforce all of the following checks before broadcasting a transaction. Any failure MUST result in the submission being rejected with an appropriate error code; partial processing is not permitted.

### 7.1 Structural Validation

- All REQUIRED fields MUST be present and non-empty.
- All hex strings MUST have a `0x` prefix and consist exclusively of hexadecimal digits (`0`–`9`, `a`–`f`). Uppercase hex MUST be normalised to lowercase before processing.
- Address fields MUST be exactly 42 characters (`0x` + 40 hex digits).
- `bytes32` fields MUST be exactly 66 characters (`0x` + 64 hex digits).
- The `v` field MUST be an integer. Values of `0` or `1` MUST be normalised to `27` or `28` respectively. All other values MUST be rejected.
- Numeric fields (`value`, `nonce`, `deadline`, `feeValue`) MUST represent non-negative integers within the `uint256` range (0 to 2²⁵⁶ − 1).

### 7.2 Semantic Validation

- The `deadline` field in `permitParams` MUST be strictly greater than the current processor clock at the time of receipt. Processors SHOULD apply a configurable tolerance (e.g., 30 seconds) to account for clock skew.
- The `nonce` field in `permitParams` MUST match the current on-chain permit nonce of the `owner` address on the specified token contract.
- For `TransferRequest`, the `payloadId` MUST correspond to a known, unprocessed checkout session held by the processor.
- For `TransferRequest`, the `ref` field in `payWithPermitParams` MUST be present, even in edge cases where a reference is not necessary. This field is formed by the merchant's-issued order reference concatenated with the `acquirerId`. In edge cases where a payment is to be made without one or both, the missing parts must be filled with zeros.
- For `BuyAcquiringPackRequest`, `feeValue` MUST equal `permitParams.value`.

### 7.3 Cryptographic Validation

- The signer recovered from `payWithPermitSig` MUST equal `permitParams.owner` for both payload types.
- The processor SHOULD verify `permitSig` locally using the token contract's EIP-712 domain separator before broadcasting, to avoid on-chain reversal.
- The `hash` field in each `ERC20RelayerSig` MUST equal the digest that the processor recomputes from the accompanying parameters. Mismatches MUST be rejected.

---

## 8. Submission Flow

### 8.1 Payment Transfer — Step-by-Step

1. **Checkout Engine** issues a payment session, returning a `ref` (16-byte number / UUID), the `beneficiary` address, the `token` address, the amount, and a `deadline`.
2. **Client Wallet** fetches the current ERC-2612 nonce for the `owner` address from the token contract.
3. **Client Wallet** fetches the fee amount to be applied in the transfer.
4. **Client Wallet** constructs `PermitParams` with `owner`, `spender` (settlement contract), `value`, `nonce`, and `deadline`.
5. **Client Wallet** signs the ERC-2612 `Permit` typed-data using EIP-712 to produce `permitSig`.
6. **Client Wallet** concatenates the `order reference` (16-byte) with the `acquirerId` (16-byte) forming the `ref` (32-byte).
7. **Client Wallet** constructs `PayWithPermitParams` with `token`, `beneficiary`, `ref`, and the `PermitParams` from step 4.
8. **Client Wallet** signs `PayWithPermitParams` using EIP-712 to produce `payWithPermitSig`.
9. **Client Wallet** generates a `payloadId` and stores it for idempotency tracking.
10. **Client Wallet** assembles the `TransferRequest` payload and submits it to the processor endpoint.
11. **Payment Processor** validates the payload per Section 7 and, on success, broadcasts the settlement transaction.
12. **Settlement Contract** executes `permit()`, `safeTransferFrom()`, and fee distribution atomically. The processor monitors the transaction and updates the session status accordingly.

### 8.2 Acquirer Service Purchase — Step-by-Step

1. **Client Wallet** fetches `acquiringPrice` from the Settlement Contract.
2. **Client Wallet** fetches the ERC-2612 nonce for the `owner` address from the token contract.
3. **Client Wallet** constructs `PermitParams` with `owner`, `spender` (settlement contract), `value` equal to `acquiringPrice`, `nonce`, and `deadline`.
4. **Client Wallet** signs the ERC-2612 `Permit` typed-data to produce `permitSig`.
5. **Client Wallet** constructs `BuyAcquiringPackPermitParams` with `token`, `feeValue`, `acquiring`, and the `PermitParams`.
6. **Client Wallet** signs `BuyAcquiringPackPermitParams` using EIP-712 to produce `payWithPermitSig`.
7. **Client Wallet** assembles the `BuyAcquiringPackRequest` payload and submits it to the processor endpoint.
8. **Payment Processor** validates the payload per Section 7 and broadcasts the settlement transaction.
9. **Settlement Contract** executes `permit()`, collects the fee, registers the `acquiring` address on-chain along with a randomly generated `acquirerId` and the `feeValue` specified in the payload. The Settlement Contract emits an event to make the new acquirer publicly known.

---

## 9. Error Handling

The payment processor MUST communicate validation failures clearly and without ambiguity. Errors SHOULD be categorised according to the following taxonomy:

| Error Category        | Description and expected response                                                                                                                                                                                                     |
| --------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `STRUCTURAL_ERROR`    | A required field is absent, of the wrong type, or violates encoding constraints (e.g., wrong hex string length). The submission MUST be rejected immediately without any on-chain activity.                                           |
| `SEMANTIC_ERROR`      | All fields are structurally valid but a business rule is violated (e.g., expired deadline, nonce mismatch, unknown `payloadId`). The submission MUST be rejected with a descriptive error indicating the violated rule.               |
| `CRYPTOGRAPHIC_ERROR` | A signature cannot be verified or the recovered signer does not match the expected owner. The submission MUST be rejected. The processor MUST NOT disclose details that could aid signature forgery.                                  |
| `BROADCAST_ERROR`     | The payload passed all validation checks but the on-chain transaction reverted or could not be submitted. The processor MUST record the failure, return an appropriate error to the client, and MUST NOT mark the session as settled. |

---

## 10. Security Considerations

### 10.1 Replay Protection

ERC-2612 nonces provide single-use permit protection at the token contract level. The `payloadId` provides single-use protection at the processor level. Compliant implementations MUST enforce both controls independently.

### 10.2 Deadline Enforcement

Deadlines protect against long-lived authorisations being replayed after a user has changed their intent. Processors MUST reject permits with deadlines in the past and SHOULD warn if the remaining window is shorter than the estimated on-chain inclusion time.

### 10.3 Signature Scope

The `payWithPermitSig` signature covers the full operation parameter object, including the token address, value, and beneficiary. Clients MUST construct this signature over the exact parameters they intend to authorise; servers MUST reject any submission where the recomputed digest does not match the `hash` field in the signature object.

### 10.4 Address Verification

The effective verification of the signature is executed on-chain by the Settlement Contract and the Token's Contract respectively. Nevertheless, this specification establishes that processors MUST NOT rely solely on the address fields provided in the payload for any purpose. The signer address recovered from each signature MUST be independently verified against the expected addresses. Accepting a payload where the provided owner does not match the recovered signer constitutes a critical vulnerability.

### 10.5 Transport Security

All submissions to the payment processor API MUST be made over TLS 1.2 or higher. Cleartext transmission of signed payloads is prohibited, as doing so would expose signatures to interception and potential misuse within the permitted window.

---

## 11. Versioning

This specification follows [Semantic Versioning 2.0.0](https://semver.org). Version increments carry the following meaning:

| Segment   | Meaning                                                                                                |
| --------- | ------------------------------------------------------------------------------------------------------ |
| **MAJOR** | A backwards-incompatible change to any payload structure, field encoding, or required validation rule. |
| **MINOR** | Addition of new optional fields or payload types that do not break existing compliant implementations. |
| **PATCH** | Editorial corrections, clarifications, and non-normative changes that do not affect compliance.        |

Implementations MUST declare which version of this specification they conform to. A processor that conforms to version `X.Y.Z` MUST accept all payloads valid under any version `X.Y'.Z'` where `Y' ≤ Y`.

---

## 12. Change Log

| Version | Date       | Author       | Summary          |
| ------- | ---------- | ------------ | ---------------- |
| 1.0.0   | 2026-03-04 | Adalton Reis | Initial release. |

---

## 13. License

Copyright © 2025 Stablecoin Stack Foundation. All rights reserved.

This specification is published under the [Apache License, Version 2.0](https://www.apache.org/licenses/LICENSE-2.0). Unless required by applicable law or agreed to in writing, material distributed under the License is distributed on an "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
