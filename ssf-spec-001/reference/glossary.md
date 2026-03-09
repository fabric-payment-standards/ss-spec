---
document: SSF-SPEC-001/reference/glossary.md
spec: SSF-SPEC-001
version: 1.0.0
status: Draft
date: 2026-03-04
author: "Adalton Reis <reis@stablecoinstack.org>"
organization: Stablecoin Stack Foundation
license: Apache License 2.0
---

# Glossary

Canonical term definitions for the Stablecoin Stack. For normative usage see [SSF-SPEC-001](../specifications/ssf-spec-001.md), Section 2.4.

---

| Term | Definition |
| ---- | ---------- |
| **Acquirer** | A registered third-party participant entitled to a share of the processing fee on payments they referred. Identified on-chain by an Acquirer ID. |
| **Acquirer ID** | A 16-byte (`bytes16`) UUID uniquely identifying a registered Acquirer within the Settlement Contract. |
| **acquiringPrice** | The amount, in the smallest unit of the respective token, that must be transferred to the Settlement Contract to register a new Acquirer. |
| **Administrator** | The privileged account that controls operational parameters of the Settlement Contract (fees, pricing, fee recipient). MUST NOT have the ability to move user funds. |
| **Base Fee** | The minimum absolute fee charged on every transfer, expressed in token units. Set by the Administrator. |
| **Acquiring Fee** | A percentage-based fee charged on behalf of a registered Acquirer. Applied only when the payment includes a non-zero Acquirer ID. |
| **Binding Signature** | An EIP-712 signature over the full operation parameters, authorising the processor to submit a specific payment or registration operation. Corresponds to `payWithPermitSig` in payload structures. |
| **Broadcast Layer** | The off-chain subsystem responsible for receiving, validating, queuing, and broadcasting signed payment payloads. Comprises wallet-gateway, broadcast-service, broadcast-submitter, and balance-and-history. |
| **Charge** | A processor-issued record representing an expected payment of a specific amount, in a specific token, to a specific merchant, within a specific time window. |
| **Checkout Engine** | The merchant-facing subsystem managing the lifecycle of payment sessions. Comprises core-checkout-engine, checkout-public-widget, ca-server, login-server, credentials-manager, and merchant-dashboard. |
| **Client Wallet** | Any compliant application that constructs and submits signed payment payloads on behalf of a payer. |
| **Ephemeral Token** | A short-lived, single-use token issued by the checkout-public-widget, authorising a wallet to retrieve session payment details. Confers no signing authority. |
| **EVM Address** | A 20-byte Ethereum account identifier, encoded as a 42-character hex string (`0x` + 40 lowercase hex digits). |
| **Hex string** | A UTF-8 string with a `0x` prefix followed by an even number of lowercase hexadecimal digits. |
| **Merchant** | A business or individual that accepts stablecoin payments by integrating with the Checkout Engine. |
| **Order Reference** | A 16-byte reference created by the merchant for tracking purposes. Embedded in the on-chain `PermittedTransfer` event for session reconciliation. |
| **Payload ID** | A client-generated identifier correlating a submission with a specific broadcasting intent. Used to enforce request idempotency at the processor level. |
| **Payer** | The individual who holds the stablecoin and signs the payment commitment. Never submits transactions directly to the network. |
| **Payment Processor** | An operator that deploys and runs a conformant instance of the Stablecoin Stack. Responsible for the Broadcast Layer, Checkout Engine, and Relayer. |
| **Permit** | An ERC-2612 off-chain authorisation allowing a spender to transfer tokens on behalf of an owner. |
| **Permit Signature** | An ERC-2612 off-chain signature authorising the Settlement Contract to call `permit()` on a token contract, granting a token allowance. Corresponds to `permitSig` in payload structures. |
| **Principal Amount** | The token amount intended to reach the beneficiary before fee deduction. |
| **Relayer** | The account that submits signed transactions on-chain on behalf of payers, absorbing gas costs. Operated by the Payment Processor. |
| **Service Provider** | An Acquirer who has opted in to public discovery via the Basic Data Service. Visible to wallets as a selectable payment provider. |
| **Session** | The lifecycle context of a single charge, from creation through to settlement or expiry. |
| **Settlement Contract** | The on-chain Solidity smart contract that verifies signatures, executes token transfers, distributes fees, and registers acquirers. The trust anchor of the system. |
| **Stablecoin Stack** | The full set of components, protocols, and interfaces specified by SSF-SPEC-001. |
| **Total With Fees** | The total token amount the payer must hold and authorise. Equals Principal Amount plus all applicable fees. This is the value in `PermitParams.value`. |
| **Zero-UUID** | The Acquirer ID consisting entirely of zero bytes (`0x00000000000000000000000000000000`). Used to indicate the absence of an Acquirer on a payment. Suppresses the Acquiring Fee without reverting. |
