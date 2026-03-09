---
sidebar_position: 2
---

# SSF-SPEC-002 — The Settlement Contract - Canonical

| Field              | Value                                        |
| ------------------ | -------------------------------------------- |
| **Document ID**    | SSF-SPEC-002                                 |
| **Title**          | The Settlement Contract — Canonical          |
| **Version**        | 1.0.0                                        |
| **Status**         | Draft                                        |
| **Date**           | 2026-03-04                                   |
| **Supersedes**     | —                                            |
| **Author(s)**      | Adalton Reis \<reis@stablecoinstack.org\>    |
| **Reviewers**      | —                                            |
| **Organization**   | Stablecoin Stack Foundation                  |
| **Contact**        | contact@stablecoinstack.org                  |
| **Public Address** | `0x0000000000000000000000000000000000000000` |
| **License**        | Apache License 2.0                           |

---

## Table of Contents

1. [Abstract](#1-abstract)
2. [Introduction](#2-introduction)
   - 2.1 [Purpose](#21-purpose)
   - 2.2 [Scope](#22-scope)
   - 2.3 [Relation to SSF-SPEC-001](#23-relation-to-ssf-spec-001)
   - 2.4 [Normative References](#24-normative-references)
   - 2.5 [Conventions and Terminology](#25-conventions-and-terminology)
3. [Design Principles](#3-design-principles)
   - 3.1 [Payer-Agnostic Execution](#31-payer-agnostic-execution)
   - 3.2 [Non-Custodial Administration](#32-non-custodial-administration)
   - 3.3 [Atomic Settlement](#33-atomic-settlement)
4. [State Variables](#4-state-variables)
5. [Events](#5-events)
6. [Fee Model](#6-fee-model)
   - 6.1 [Overview](#61-overview)
   - 6.2 [calculateFees](#62-calculatefees)
   - 6.3 [breakdownTransferAmount](#63-breakdowntransferamount)
7. [Functions](#7-functions)
   - 7.1 [transferWithPermit](#71-transferwithpermit)
   - 7.2 [buyAcquiringPack](#72-buyacquiringpack)
   - 7.3 [getBalances (single token)](#73-getbalances-single-token)
   - 7.4 [getBalances (multi-token overload)](#74-getbalances-multi-token-overload)
   - 7.5 [getAcquiringWallet](#75-getacquiringwallet)
8. [Dual-Signature Pattern](#8-dual-signature-pattern)
9. [Security Considerations](#9-security-considerations)
   - 9.1 [Hash Replay Protection](#91-hash-replay-protection)
   - 9.2 [Acquirer Fee Cap](#92-acquirer-fee-cap)
   - 9.3 [Administrative Privilege Boundaries](#93-administrative-privilege-boundaries)
   - 9.4 [Zero-UUID Convention](#94-zero-uuid-convention)
10. [Versioning](#10-versioning)
11. [Change Log](#11-change-log)
12. [License](#12-license)

---

## 1. Abstract

This specification defines the canonical Settlement Contract for the Stablecoin Stack payment processing system. The Settlement Contract is the on-chain component responsible for verifying cryptographic authorisations, executing token transfers, distributing processing fees, and registering acquiring participants. It is written in Solidity and deployed on Ethereum-compatible networks, operating exclusively with ERC-2612-compliant stablecoin tokens.

This document presents the minimal set of state variables, events, and functions that a conformant Settlement Contract implementation MUST provide. Solidity snippets are included as normative illustrations of the required interface; compliant implementations MAY extend the contract with additional functionality provided they do not alter the behaviour of the specified interface.

---

## 2. Introduction

### 2.1 Purpose

This document provides a precise, canonical description of the Settlement Contract's on-chain interface. It is intended for:

- smart-contract engineers implementing or auditing a conformant Settlement Contract;
- payment processor operators who need to understand the on-chain guarantees and constraints upon which their off-chain service depends; and
- integrators and auditors assessing the correctness and security of a deployed instance.

### 2.2 Scope

This specification covers:

- the design principles that govern the contract's behaviour;
- all state variables required to manage fees, balances, permit replay protection, and acquirer registration;
- all events that a compliant contract MUST emit;
- the complete interface of each public and external function, including parameter semantics, preconditions, postconditions, and revert conditions; and
- the dual-signature verification pattern used to bind off-chain payment authorisations to on-chain execution.

This specification does not cover the off-chain payload construction and submission protocol (addressed in [SSF-SPEC-001](../../ssf-spec-001/specifications/ssf-spec-001.md)), the checkout engine session lifecycle, the merchant dashboard, or deployment and upgrade procedures.

### 2.3 Relation to SSF-SPEC-001

[SSF-SPEC-001](../../ssf-spec-001/specifications/ssf-spec-001.md) defines how a client constructs, signs, and submits a payment or acquirer registration payload to the Payment Processor API. The present specification defines what the Settlement Contract does once the Payment Processor broadcasts that payload on-chain. The two documents are complementary and MUST be read together to understand the full conformance surface of the Stablecoin Stack.

Terminology established in SSF-SPEC-001 — including **Acquirer**, **Permit**, **Relayer**, **Payload ID**, and **Binding Signature** — is adopted without redefinition in this document. Where this specification introduces additional terms, they are defined in Section 2.5.

### 2.4 Normative References

| Reference                    | Description                                                |
| ---------------------------- | ---------------------------------------------------------- |
| [SSF-SPEC-001](../../ssf-spec-001/specifications/ssf-spec-001.md) | Instant Payment With Permitted Token Transfer — Submission |
| ERC-20                       | Token Standard — Ethereum Improvement Proposal 20          |
| ERC-2612                     | Permit Extension for EIP-20 Signed Approvals               |
| EIP-712                      | Typed Structured Data Hashing and Signing                  |
| Solidity 0.8.x               | Smart Contract Programming Language — solidity-lang.org    |
| OpenZeppelin Contracts       | Audited Solidity contract library — openzeppelin.com       |

### 2.5 Conventions and Terminology

The key words **MUST**, **MUST NOT**, **REQUIRED**, **SHALL**, **SHOULD**, **RECOMMENDED**, and **OPTIONAL** in this document are to be interpreted as described in RFC 2119.

| Term                | Definition                                                                                                                                                                                                                  |
| ------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Settlement Contract | The on-chain smart contract specified by this document. Responsible for permit verification, token transfer, fee distribution, and acquirer registration.                                                                   |
| Administrator       | The privileged account that controls operational parameters of the Settlement Contract (fees, pricing, fee recipient). The Administrator MUST NOT have the ability to move user funds.                                      |
| Acquirer            | A registered participant entitled to a share of the processing fee on payments they referred. Defined in SSF-SPEC-001.                                                                                                      |
| Acquirer ID         | A 16-byte (`bytes16`) UUID uniquely identifying an acquirer within the system. Referred to as `referral` in the contract's storage mapping for legacy reasons; the canonical term in all specifications is **Acquirer ID**. |
| Zero-UUID           | The acquirer ID composed entirely of zero bytes (`0x00000000000000000000000000000000`). Represents the absence of an acquirer on a given payment.                                                                           |
| Permit Signature    | The ERC-2612 signature authorising the Settlement Contract to call `permit()` on the token contract. Corresponds to `permitSig` in SSF-SPEC-001.                                                                            |
| Binding Signature   | The EIP-712 signature over the operation parameters, authorising the processor to submit the specific operation. Corresponds to `payWithPermitSig` in SSF-SPEC-001.                                                         |
| Principal Amount    | The token amount intended to reach the beneficiary before fee deduction.                                                                                                                                                    |
| Total With Fees     | The total token amount that must be held by the payer, equal to the principal amount plus all applicable fees.                                                                                                              |
| Base Fee            | The minimum fee charged by the processor on every transfer, expressed as an absolute token unit amount.                                                                                                                     |
| Acquiring Fee       | The additional percentage-based fee charged on behalf of a registered acquirer. Expressed in basis points or a processor-defined unit.                                                                                      |

---

## 3. Design Principles

### 3.1 Payer-Agnostic Execution

The Settlement Contract does not record, validate, or require knowledge of the payer's identity. Execution proceeds solely on the basis of a valid cryptographic authorisation: provided the Permit Signature and the Binding Signature are valid and all other preconditions are met, the transfer MUST be executed regardless of which account submits the transaction. This design enables the Relayer to broadcast transactions on behalf of payers, eliminating the need for payers to hold native gas tokens.

The reconciliation of payment sessions at the checkout level equally does not depend on who submitted the transaction. Checkout sessions are resolved by the `ref` order reference embedded in the on-chain event, not by the `msg.sender`.

### 3.2 Non-Custodial Administration

The Administrator holds privileged control over the operational parameters of the Settlement Contract — specifically the `baseFeeAmount`, the `maxAcquiringFee`, the acquirer entitlement price, and the address designated to receive collected fees. However, the Administrator MUST NOT possess any mechanism to transfer, freeze, or otherwise encumber token balances held within the contract on behalf of participants. This constraint MUST be enforced at the contract level, not merely by policy.

### 3.3 Atomic Settlement

Each call to `transferWithPermit` and `buyAcquiringPack` MUST execute all of its constituent operations atomically. Partial execution — for example, collecting the full amount from the payer but failing to distribute fees — is not permitted. If any step fails, the entire transaction MUST revert, leaving all balances unchanged.

---

## 4. State Variables

The following state variables constitute the minimal persistent state that a conformant Settlement Contract MUST maintain.

```solidity
/// @notice Minimum absolute fee (in token units) charged on every transfer.
/// Protects the processor against gas losses on very small payments.
uint256 public baseFeeAmount;

/// @notice Maximum acquiring fee percentage an acquirer may configure.
/// Enforced at registration time and on any subsequent fee update.
uint256 public maxAcquiringFee;

/// @notice Internal token balances per participant.
/// Mapping: token address => participant address => amount (in token units).
mapping(address => mapping(address => uint256)) public balances;

/// @notice Registry of consumed permit hashes.
/// A hash present in this mapping MUST cause the transaction to revert,
/// preventing permit replay attacks.
mapping(bytes32 => bool) public usedHashes;

/// @notice Registry mapping Acquirer IDs to their wallet addresses.
/// Key: bytes16 Acquirer ID (UUID). Value: acquirer wallet address.
mapping(bytes16 => address) public acquirerWallets;

/// @notice Fee percentage charged by each registered acquirer.
/// Key: acquirer wallet address. Value: fee in processor-defined units.
mapping(address => uint256) public acquiringFeePercent;

/// @notice Set of ERC-2612-compliant tokens accepted for acquirer entitlement purchases.
mapping(IERC20Permit => bool) public acquiringAllowedTokens;

/// @notice Price (in token units) required to register as a new acquirer.
uint256 public acquiringPrice;
```

**Notes:**

- `baseFeeAmount` provides a floor that ensures every transaction covers at minimum the cost of its on-chain execution. Administrators SHOULD set this value to reflect prevailing gas costs.
- `maxAcquiringFee` is a ceiling applied at acquirer registration. Acquirers MAY set their fee anywhere between zero and this ceiling, allowing them to balance competitiveness against revenue.
- `balances` uses a two-level mapping (`token → participant → amount`) to support multiple token types within a single contract instance.
- `usedHashes` stores the EIP-712 digest of each processed Binding Signature. Once recorded, any subsequent submission presenting the same hash MUST revert.
- `acquirerWallets` uses `bytes16` (UUID) keys. The canonical term for this key is **Acquirer ID**. The identifier `referral` used in earlier drafts is deprecated in favour of this terminology.

---

## 5. Events

A conformant Settlement Contract MUST emit the following events. Off-chain services — including the Payment Processor, the Merchant Dashboard, and monitoring infrastructure — rely on these events for settlement confirmation, fee accounting, and session reconciliation.

```solidity
/// @notice Emitted when a participant withdraws their accumulated balance.
/// @param owner        The address whose internal balance was debited.
/// @param beneficiary  The address that received the withdrawn funds.
/// @param amount       The amount withdrawn, in token units.
event Withdrawal(
    address indexed owner,
    address indexed beneficiary,
    uint256 amount
);

/// @notice Emitted upon successful execution of a permitted token transfer.
/// @param domainSeparator  The EIP-712 domain separator of the ERC2612 token.
/// @param token        The ERC-2612 token contract address.
/// @param payer        The token owner whose permit authorised the transfer.
/// @param recipient    The beneficiary address that received the principal amount.
/// @param value        The principal amount transferred to the recipient, in token units.
/// @param fee          The total fees deducted from the transfer, in token units.
/// @param orderReference   The 32-byte order reference linking this transfer to a checkout session.
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
/// @param acquirerId   The bytes16 UUID assigned to the new acquirer.
/// @param wallet       The wallet address of the registered acquirer.
/// @param feePercent   The acquiring fee percentage the acquirer will charge.
event AcquirerCreated(
    bytes16 indexed acquirerId,
    address indexed wallet,
    uint256 feePercent
);

/// @notice Emitted when an acquirer updates their fee percentage.
/// @param acquiring    The acquirer's wallet address.
/// @param feePercent   The new fee percentage value.
event AcquiringFeeUpdated(
    address indexed acquiring,
    uint256 feePercent
);

/// @notice Emitted when an acquiring fee is generated on a payment.
/// @param acquirerId   The Acquirer ID that earned the commission.
/// @param amount       The commission amount, in token units.
event CommissionGenerated(
    bytes16 indexed acquirerId,
    uint256 amount
);
```

**Notes on event usage:**

- `PermittedTransfer` is the primary event consumed by the checkout engine for session reconciliation. The `orderReference` field MUST match the `ref` value submitted in the `TransferRequest` payload defined in SSF-SPEC-001, Section 6.2.
- `AcquirerCreated` replaces the earlier draft name `AcquirringCreated`. The parameter formerly named `feeValue` is renamed `feePercent` for clarity, as it represents a rate, not an absolute amount.
- `CommissionGenerated` replaces the earlier draft name `CommissonGenerated`.

---

## 6. Fee Model

### 6.1 Overview

Every payment processed by the Settlement Contract is subject to two potentially applicable fee components:

| Fee Component              | Type                                                | Applicability                                                  |
| -------------------------- | --------------------------------------------------- | -------------------------------------------------------------- |
| Base Fee (`baseFeeAmount`) | Absolute, in token units                            | Applied to every transfer unconditionally.                     |
| Acquiring Fee              | Percentage of the principal, as set by the acquirer | Applied only when the payment includes a non-zero Acquirer ID. |

The contract provides two pure view functions to assist payers, merchants, and checkout integrations in computing correct amounts prior to submitting a transaction.

When no acquirer is involved in a payment, the caller MUST pass the **Zero-UUID** (`0x00000000000000000000000000000000`) as the `acquirerId` parameter. The Zero-UUID is a reserved identifier with no wallet assigned; it suppresses the acquiring fee component entirely.

### 6.2 `calculateFees`

Computes the total fee to be added on top of a given principal amount, such that the recipient receives exactly the principal after deduction.

```solidity
function calculateFees(
    uint256 amount,
    bytes16 acquirerId
) public view returns (uint256 totalFee)
```

**Parameters:**

| Parameter    | Type      | Description                                                                      |
| ------------ | --------- | -------------------------------------------------------------------------------- |
| `amount`     | `uint256` | The principal amount intended to reach the beneficiary, in token units.          |
| `acquirerId` | `bytes16` | The Acquirer ID for this payment. Pass the Zero-UUID if no acquirer is involved. |

**Returns:** `totalFee` — the absolute fee amount in token units that must be added to `amount` to yield the correct value for the `value` field in `PermitParams`.

**Usage:** A checkout integration that wishes to present the payer with a total amount inclusive of fees SHOULD call this function with the intended payment principal to determine the exact amount for which the permit must be signed.

### 6.3 `breakdownTransferAmount`

Decomposes a total amount (principal plus fees) into its constituent parts.

```solidity
function breakdownTransferAmount(
    uint256 totalWithFees,
    bytes16 acquirerId
) public view returns (uint256 principalAmount, uint256 totalFees)
```

**Parameters:**

| Parameter       | Type      | Description                                                                                                                 |
| --------------- | --------- | --------------------------------------------------------------------------------------------------------------------------- |
| `totalWithFees` | `uint256` | The total token amount inclusive of all fees, in token units. This is the value that appears in the permit's `value` field. |
| `acquirerId`    | `bytes16` | The Acquirer ID for this payment. Pass the Zero-UUID if no acquirer is involved.                                            |

**Returns:**

| Return value      | Description                                                                                |
| ----------------- | ------------------------------------------------------------------------------------------ |
| `principalAmount` | The amount that will be credited to the beneficiary after fee deduction.                   |
| `totalFees`       | The total fees that will be distributed to the processor and, if applicable, the acquirer. |

**Usage:** This function is the inverse of `calculateFees`. It is particularly useful for merchant-facing integrations that display the net amount a merchant will receive after processing fees, and for audit tooling that verifies the fee distribution against emitted events.

---

## 7. Functions

### 7.1 `transferWithPermit`

Executes a complete stablecoin payment: verifies both signatures, collects the full amount from the payer via `permit` and `transferFrom`, distributes fees, and credits the principal to the beneficiary.

```solidity
function transferWithPermit(
    IERC20Permit token,
    address tokenOwner,
    uint256 amount,
    uint256 deadline,
    uint8 v1,
    bytes32 r1,
    bytes32 s1,
    uint8 v2,
    bytes32 r2,
    bytes32 s2,
    address recipient,
    bytes32 ref
) external
```

**Parameters:**

| Parameter        | Type                          | Description                                                                                                                                                           |
| ---------------- | ----------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `token`          | `IERC20Permit`                | The ERC-2612-compliant stablecoin token contract to transfer from.                                                                                                    |
| `tokenOwner`     | `address`                     | The address of the token holder whose permit authorises the transfer.                                                                                                 |
| `amount`         | `uint256`                     | The total token amount inclusive of all fees, in token units. Must equal the `value` in the signed permit.                                                            |
| `deadline`       | `uint256`                     | The Unix timestamp at which the permit expires. Forwarded verbatim to the token's `permit()` call.                                                                    |
| `v1`, `r1`, `s1` | `uint8`, `bytes32`, `bytes32` | The ECDSA components of the **Permit Signature** — validated by the ERC-2612 token contract.                                                                          |
| `v2`, `r2`, `s2` | `uint8`, `bytes32`, `bytes32` | The ECDSA components of the **Binding Signature** — validated by the Settlement Contract itself.                                                                      |
| `recipient`      | `address`                     | The beneficiary address to which the principal amount is credited.                                                                                                    |
| `ref`            | `bytes32`                     | The 32-byte order reference linking this transfer to a checkout session. Emitted in the `PermittedTransfer` event and used by the checkout engine for reconciliation. |

**Preconditions — the function MUST revert if any of the following hold:**

- The EIP-712 digest of the Binding Signature is present in `usedHashes`.
- The Binding Signature (`v2`, `r2`, `s2`) does not recover to a signer authorised by the processor.
- `deadline` is less than or equal to `block.timestamp`.
- The `token` contract's `permit()` call reverts (invalid Permit Signature or expired deadline).
- The token balance of `tokenOwner` is less than `amount`.

**Postconditions — upon successful execution:**

- The Binding Signature digest is recorded in `usedHashes`.
- The full `amount` is transferred from `tokenOwner` to the Settlement Contract via `transferFrom`.
- The fee components are distributed to their respective recipients (processor fee recipient and, if applicable, the acquirer's internal balance).
- The principal amount (`amount` minus total fees) is transferred or credited to `recipient`.
- A `PermittedTransfer` event is emitted.
- If an acquirer fee was generated, a `CommissionGenerated` event is emitted.

### 7.2 `buyAcquiringPack`

Registers a new acquirer by accepting a fee payment on behalf of the processor. The payer and the account being registered as an acquirer do not need to be the same address, enabling delegation of the registration payment.

```solidity
function buyAcquiringPack(
    IERC20Permit token,
    address payer,
    address acquiring,
    uint256 feePercent,
    uint256 price,
    uint256 deadline,
    uint8 v1,
    bytes32 r1,
    bytes32 s1,
    uint8 v2,
    bytes32 r2,
    bytes32 s2
) public
```

**Parameters:**

| Parameter        | Type                          | Description                                                                                                                                                      |
| ---------------- | ----------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `token`          | `IERC20Permit`                | The ERC-2612-compliant stablecoin token used to pay the registration fee. Must be present in `acquiringAllowedTokens`.                                           |
| `payer`          | `address`                     | The token holder whose permit authorises the fee payment. May differ from `acquiring`.                                                                           |
| `acquiring`      | `address`                     | The wallet address to be registered as a new acquirer.                                                                                                           |
| `feePercent`     | `uint256`                     | The acquiring fee percentage this acquirer will charge on referred payments. MUST NOT exceed `maxAcquiringFee`.                                                  |
| `price`          | `uint256`                     | The registration price in token units, as currently configured by the Administrator in `acquiringPrice`. The contract MUST verify this matches the stored value. |
| `deadline`       | `uint256`                     | The Unix timestamp at which the permit expires.                                                                                                                  |
| `v1`, `r1`, `s1` | `uint8`, `bytes32`, `bytes32` | The ECDSA components of the **Permit Signature** — validated by the ERC-2612 token contract.                                                                     |
| `v2`, `r2`, `s2` | `uint8`, `bytes32`, `bytes32` | The ECDSA components of the **Binding Signature** — validated by the Settlement Contract.                                                                        |

**Preconditions — the function MUST revert if any of the following hold:**

- `token` is not present in `acquiringAllowedTokens`.
- `feePercent` exceeds `maxAcquiringFee`.
- `price` does not equal the current value of `acquiringPrice`.
- The `acquiring` address is already registered as an acquirer.
- The Binding Signature digest is present in `usedHashes`.
- The Binding Signature does not recover to a signer authorised by the processor.
- The token contract's `permit()` call reverts.
- The token balance of `payer` is less than `price`.

**Postconditions — upon successful execution:**

- The Binding Signature digest is recorded in `usedHashes`.
- `price` tokens are transferred from `payer` to the processor's fee recipient.
- A new `bytes16` Acquirer ID (UUID) is assigned to the `acquiring` address and recorded in `acquirerWallets`.
- The `acquiring` address is recorded in `acquiringFeePercent` with the supplied `feePercent`.
- An `AcquirerCreated` event is emitted.

### 7.3 `getBalances` (single token)

Returns the internal contract balances for a list of participant addresses, all denominated in the same token.

```solidity
function getBalances(
    address token,
    address[] calldata users
) external view returns (uint256[] memory balances)
```

**Parameters:**

| Parameter | Type        | Description                                                     |
| --------- | ----------- | --------------------------------------------------------------- |
| `token`   | `address`   | The token contract address for which balances are queried.      |
| `users`   | `address[]` | An array of participant addresses whose balances are requested. |

**Returns:** An array of `uint256` values, parallel to `users`, each representing the internal balance of the corresponding address for the specified token.

### 7.4 `getBalances` (multi-token overload)

Returns the internal contract balances for a list of participant addresses across multiple tokens simultaneously. This overload enables efficient multicall-style balance retrieval.

```solidity
function getBalances(
    address[] calldata tokens,
    address[] calldata users
) external view returns (uint256[][] memory balances)
```

**Parameters:**

| Parameter | Type        | Description                                                     |
| --------- | ----------- | --------------------------------------------------------------- |
| `tokens`  | `address[]` | An array of token contract addresses to query.                  |
| `users`   | `address[]` | An array of participant addresses whose balances are requested. |

**Returns:** A two-dimensional array of `uint256` values with dimensions `[tokens.length][users.length]`, where `balances[i][j]` is the internal balance of `users[j]` for `tokens[i]`.

### 7.5 `getAcquiringWallet`

Resolves the wallet address associated with a given Acquirer ID.

```solidity
function getAcquiringWallet(
    bytes16 acquirerId
) public view returns (address)
```

**Parameters:**

| Parameter    | Type      | Description                      |
| ------------ | --------- | -------------------------------- |
| `acquirerId` | `bytes16` | The UUID Acquirer ID to resolve. |

**Returns:** The wallet address registered under the supplied Acquirer ID, or the zero address if the ID is not registered.

**Usage:** This function is used by off-chain services to verify that an Acquirer ID submitted in a payment payload corresponds to a legitimately registered acquirer before broadcasting the transaction.

---

## 8. Dual-Signature Pattern

The Settlement Contract enforces a **dual-signature pattern** on both `transferWithPermit` and `buyAcquiringPack`. This pattern is defined conceptually in SSF-SPEC-001, Section 6.1, and is implemented on-chain as described here.

Each operation requires two independent ECDSA signatures:

| Signature                                | Parameters                                                                                 | Validated by                                                 | Purpose                                                                                                    |
| ---------------------------------------- | ------------------------------------------------------------------------------------------ | ------------------------------------------------------------ | ---------------------------------------------------------------------------------------------------------- |
| **Permit Signature** (`v1`, `r1`, `s1`)  | Covers the ERC-2612 `Permit` typed-data: `owner`, `spender`, `value`, `nonce`, `deadline`. | The ERC-2612 token contract, inside its `permit()` function. | Authorises the Settlement Contract to pull tokens from the payer's wallet.                                 |
| **Binding Signature** (`v2`, `r2`, `s2`) | Covers the full operation parameters: token, recipient, amount, ref, and deadline.         | The Settlement Contract itself.                              | Binds the processor's authorisation to this specific operation, preventing parameter substitution attacks. |

The digest of the Binding Signature MUST be stored in `usedHashes` upon first use. Any subsequent transaction presenting the same digest MUST revert. This provides operation-level replay protection independent of the ERC-2612 nonce mechanism.

The two controls are complementary:

- The ERC-2612 nonce prevents the Permit Signature from being replayed across different operations.
- The `usedHashes` registry prevents the Binding Signature from being replayed even if a new permit with the same parameters were issued.

---

## 9. Security Considerations

### 9.1 Hash Replay Protection

The `usedHashes` mapping is the primary on-chain guard against replay attacks at the operation level. Implementations MUST write to this mapping before executing any token transfer, following the checks-effects-interactions pattern. Failure to do so would expose the contract to reentrancy-based replay attacks.

### 9.2 Acquirer Fee Cap

The `maxAcquiringFee` ceiling MUST be enforced at registration time. Implementations MUST also re-validate the fee cap on any subsequent fee update to prevent a registered acquirer from escalating their fee beyond the permitted ceiling after registration.

### 9.3 Administrative Privilege Boundaries

The Administrator's privileges MUST be strictly limited to parameter configuration. Specifically, the Administrator MUST NOT be able to:

- call `transferFrom` on any token held by a participant;
- alter the `balances` mapping directly; or
- drain the fee recipient balance to an arbitrary address without going through the standard withdrawal mechanism.

These constraints SHOULD be enforced through the contract's access control logic and verified during security audits.

### 9.4 Zero-UUID Convention

The Zero-UUID (`0x00000000000000000000000000000000`) is a reserved Acquirer ID. The contract MUST ensure that no wallet address can be registered under the Zero-UUID. Passing the Zero-UUID as the `acquirerId` in any fee calculation or transfer function MUST be interpreted as "no acquirer" and MUST suppress the acquiring fee component without reverting.

---

## 10. Versioning

This specification follows [Semantic Versioning 2.0.0](https://semver.org). Version increments carry the following meaning:

| Segment   | Meaning                                                                                            |
| --------- | -------------------------------------------------------------------------------------------------- |
| **MAJOR** | A backwards-incompatible change to the contract interface, event signatures, or storage layout.    |
| **MINOR** | Addition of new functions or events that do not alter the behaviour of existing interface members. |
| **PATCH** | Editorial corrections, clarifications, and non-normative changes that do not affect conformance.   |

---

## 11. Change Log

| Version | Date       | Author       | Summary                                                                                                                                                                                                                                                                                            |
| ------- | ---------- | ------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 1.0.0   | 2026-03-04 | Adalton Reis | Initial release. Terminology aligned with SSF-SPEC-001: `acquirring` → `acquiring`, `distributor` → `acquirer`, `referral` identifier → `acquirerId`, `breakdowTransferAmount` → `breakdownTransferAmount`, `AcquirringCreated` → `AcquirerCreated`, `CommissonGenerated` → `CommissionGenerated`. |

---

## 12. License

Copyright © 2026 Stablecoin Stack Foundation. All rights reserved.

This specification is published under the [Apache License, Version 2.0](https://www.apache.org/licenses/LICENSE-2.0). Unless required by applicable law or agreed to in writing, material distributed under the License is distributed on an "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
