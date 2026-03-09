# Glossary

This glossary defines all terms used across the Stablecoin Stack specification series. Where a term has a specific normative meaning in a particular specification, that specification is noted. Terms defined here supersede informal usage elsewhere in the documentation.

---

## A

**Acquirer**
A registered third-party participant who distributes the Stablecoin Stack payment service to payers and earns a share of the processing fee on payments they referred. Acquirers are registered on-chain via the `buyAcquiringPack` function of the Settlement Contract. See also: *Service Provider*, *Acquirer ID*.

**Acquirer ID**
A 16-byte (`bytes16`) UUID that uniquely identifies a registered Acquirer within the Settlement Contract. Assigned on-chain at registration time. Used by payers to indicate the Acquirer to credit on a given payment. See also: *Zero-UUID*.

**Administrator**
The privileged account that controls operational parameters of the Settlement Contract: fee amounts, acquirer price, and the fee recipient address. The Administrator MUST NOT have the ability to move participant funds. Operated by the Payment Processor.

**Atomic Settlement**
The property that all steps of a payment — permit verification, token transfer, fee distribution, and event emission — occur within a single transaction. If any step fails, all steps revert. Partial settlement is impossible.

---

## B

**Basic Data Service** (`basic-data-server`)
A publicly accessible, read-only service that provides shared reference data: supported token list, Service Provider (Acquirer) registry, and processor public configuration. Designed to be operated as a shared public resource.

**Binding Signature**
An EIP-712 signature over the full operation parameters (token, beneficiary, amount, ref, deadline), produced by the payer and validated by the Settlement Contract. Authorises the processor to execute a specific payment or acquirer registration operation. One of the two signatures required by the dual-signature pattern. Corresponds to the `payWithPermitSig` field in SSF-SPEC-001.

**Broadcast Layer**
The set of off-chain services responsible for receiving, validating, queuing, submitting, and reporting the status of signed payment payloads. Comprises: broadcast-gateway, broadcast-service, broadcast-submitter, balance-and-history.

**broadcast-gateway**
The external entry point for wallet connections. Accepts payload submissions and maintains WebSocket connections for real-time status delivery.

**broadcast-service**
Manages the internal submission lifecycle: queuing, state tracking, and real-time status publication to connected wallets.

**broadcast-submitter**
Holds the Relayer's funded account. The only off-chain component that issues Ethereum transactions.

---

## C

**ca-server**
The internal Certificate Authority operated by the Payment Processor. Issues and manages TLS certificates used in mTLS sessions between merchant servers and the core-checkout-engine.

**Charge**
A processor-issued record representing an expected payment of a specific amount, in a specific token, to a specific merchant, within a specific time window. Created by the Checkout Engine upon request from the Merchant Server.

**Checkout Engine**
The merchant-facing subsystem. Manages the payment session lifecycle from charge creation through to settlement reconciliation. Comprises: core-checkout-engine, checkout-public-widget, ca-server, login-server, credentials-manager, merchant-dashboard.

**checkout-public-widget**
A payment page hosted by the Payment Processor. Delivers session payment details to the wallet via an Ephemeral Token. The information bridge between the Checkout Engine and the Client Wallet.

**Client Wallet**
Any application that constructs and signs compliant payment payloads, submits them to the broadcast-gateway, and displays real-time payment status to the user.

**core-checkout-engine**
The central service of the Checkout Engine. Manages session creation, charge management, and settlement reconciliation. Exposes a merchant API protected by mTLS.

**credentials-manager**
Administrative service for certificate and user lifecycle management. Interfaces with ca-server and Keycloak.

---

## D

**Deadline**
A Unix timestamp (seconds since 1970-01-01T00:00:00Z) after which a permit or payment authorisation is no longer valid. Enforced by both the token contract (`permit()`) and the Settlement Contract (`transferWithPermit`).

**Dual-Signature Pattern**
The requirement that every payment and acquirer registration operation carry exactly two independent ECDSA signatures: the Permit Signature (validated by the token contract) and the Binding Signature (validated by the Settlement Contract). Neither alone is sufficient to execute an operation.

---

## E

**ECDSA**
Elliptic Curve Digital Signature Algorithm. The signature scheme used for all signatures in the Stablecoin Stack, operating over the secp256k1 curve (FIPS 186-4, SEC 1 v2.0). The same scheme used by the Ethereum protocol for transaction signing.

**EIP-712**
Ethereum Improvement Proposal 712 — Typed Structured Data Hashing and Signing. The standard used to construct the digests over which Permit Signatures and Binding Signatures are computed. Binds each signature to a specific domain (chain, contract, version) and typed data structure, preventing cross-domain replay.

**ERC-2612**
The Permit Extension for EIP-20 Signed Approvals. Allows a token holder to sign an off-chain authorisation (a *permit*) that delegates a spending allowance to a third party without requiring an on-chain `approve()` transaction. Central to the Stablecoin Stack's gasless payment model.

**Ephemeral Token**
A short-lived, single-use credential issued by the checkout-public-widget. Authorises its holder to retrieve the payment parameters for a specific session. Confers no signing authority. MUST expire within a processor-configured window (RECOMMENDED: no longer than 5 minutes) and MUST be invalidated upon first redemption.

**EVM Address**
A 20-byte Ethereum account identifier. Encoded in this specification as a 42-character hex string: `0x` followed by 40 lowercase hexadecimal digits.

---

## F

**Fee Recipient**
The address to which the base fee component of each payment is credited. Configured by the Administrator in the Settlement Contract.

**Finality**
The state of a transaction after a sufficient number of block confirmations have elapsed for it to be treated as irreversible. Distinct from *Submission Completion*. Settlement reconciliation occurs only upon finality.

---

## G

**Gas Abstraction**
The architectural property that payers are never required to hold or spend native gas tokens. Gas is paid by the Relayer and recovered through the processing fee.

---

## H

**Hex String**
A UTF-8 string encoding binary data as hexadecimal. All hex strings in this specification MUST have a `0x` prefix and use lowercase digits (`0`–`9`, `a`–`f`). Leading zero bytes MUST be preserved; the string MUST always be the full expected length.

---

## L

**login-server**
Issues JWT bearer tokens to merchants within the mTLS context, enabling programmatic access to the core-checkout-engine API.

---

## M

**merchant-dashboard**
The merchant-facing web interface. Provides account management, real-time order monitoring, fee reporting, and statistics.

**Merchant Server**
The server-side application operated by a merchant. Communicates with the core-checkout-engine over mTLS to create payment sessions and receive settlement confirmation.

**mTLS (Mutual TLS)**
A TLS session in which both the client and the server present certificates, enabling bidirectional authentication. Used to secure communication between Merchant Servers and the core-checkout-engine per RFC 8705.

---

## O

**Order Reference (`ref`)**
A 32-byte (`bytes32`) value generated by the core-checkout-engine that uniquely identifies a charge. Embedded in the signed payload and in the on-chain `PermittedTransfer` event. Used by the Checkout Engine to reconcile confirmed on-chain transactions with off-chain sessions. Protected from modification by the Binding Signature.

---

## P

**Payload ID**
A processor-issued opaque identifier correlating an off-chain checkout session with a submitted `TransferRequest`. Enforced as single-use by the Payment Processor's off-chain validation layer.

**Payer**
The individual who holds the stablecoin and produces the cryptographic signatures that authorise a payment.

**Payer-Agnostic Execution**
The property that the Settlement Contract does not validate the identity of `msg.sender`. A payment is valid if and only if it carries valid signatures; the submitter's identity is irrelevant.

**Payment Processor**
An operator who deploys and runs a conformant instance of the Stablecoin Stack. Responsible for operating the Checkout Engine, Broadcast Layer, Relayer, and Settlement Contract.

**Permit**
An ERC-2612 off-chain authorisation that grants a specified spender the ability to transfer a specified amount of tokens from a token holder's account, without requiring an on-chain `approve()` transaction.

**Permit Signature**
An ERC-2612 typed-data signature over the permit parameters (owner, spender, value, nonce, deadline), produced by the token holder and validated by the ERC-2612 token contract. Authorises the Settlement Contract to call `permit()`. One of the two signatures required by the dual-signature pattern. Corresponds to the `permitSig` field in SSF-SPEC-001.

**Principal Amount**
The token amount intended to reach the beneficiary after fee deduction. Equal to the total transfer amount minus all applicable fees.

---

## R

**Relayer**
The entity that holds a funded Ethereum account and submits signed transactions on-chain on behalf of payers, absorbing gas costs. Operated by the Payment Processor. Recovered through the base fee mechanism.

---

## S

**Service Provider**
A registered Acquirer who has opted in to public discovery via the basic-data-server. Visible to wallets as a selectable payment provider. Users choose a Service Provider on the basis of fee rates, reputation, or personal preference.

**Session**
The lifecycle context of a single Charge, from creation by the Checkout Engine through to settlement or expiry.

**Settlement Contract**
The on-chain Solidity smart contract that is the trust anchor of the Stablecoin Stack. Verifies permit and binding signatures, executes token transfers, distributes fees, and registers acquirers. Fully specified in SSF-SPEC-002.

**Stablecoin Stack**
The full set of components, protocols, and interfaces specified by the Stablecoin Stack Foundation specification series, taken together.

**Submission Completion**
The state of a transaction after the Relayer has submitted it to the network and received no revert. Does not imply *Finality*. The broadcast-service notifies the wallet upon submission completion; settlement reconciliation occurs only upon finality.

---

## T

**Total With Fees**
The total token amount that must be held and permitted by the payer. Equal to the Principal Amount plus all applicable fees (base fee and, if applicable, acquiring fee). This is the value that appears in the `value` field of `PermitParams`.

**transfer-history**
The service that monitors the Settlement Contract and supported token contracts for on-chain events. The authoritative source of confirmed transaction data in the off-chain stack. Publishes `PermittedTransfer` event confirmations to the core-checkout-engine after the required block confirmation depth.

---

## U

**`usedHashes`**
A mapping in the Settlement Contract from `bytes32` to `bool`. Records the EIP-712 digest of every consumed Binding Signature. Any transaction presenting a digest already present in this mapping MUST revert. The primary on-chain replay protection mechanism for Binding Signatures.

---

## Z

**Zero-UUID**
The Acquirer ID consisting entirely of zero bytes (`0x00000000000000000000000000000000`). A reserved value representing the absence of an Acquirer on a given payment. When used as the Acquirer ID, the acquiring fee component is suppressed without reverting. No wallet address can be registered under the Zero-UUID.
