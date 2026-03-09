# Introduction to the Stablecoin Stack

The **Stablecoin Stack** is an open architecture for processing stablecoin payments on Ethereum-compatible networks. It defines the protocols, interfaces, and on-chain contracts required to accept a stablecoin payment — from the moment a merchant creates a charge to the moment the funds are confirmed on-chain and the session is reconciled.

The system is designed around a single proposition: that **cryptographic signatures are a superior foundation for payment authorisation**, and that the tooling to make this practical for everyday users is both achievable and urgently needed.

---

## What Problem Does This Solve?

Today's payment systems are built on delegated trust. A user instructs an institution, which authorises a payment on their behalf. The proof of the user's intent lives in the institution's records — not in the payment itself.

A cryptographically signed payment commitment is different. It carries its own proof. Anyone can verify that the holder of a specific private key produced it. The payer's intent is bound to the specific parameters of the specific payment, unforgeable without their key, and verifiable without contacting any authority.

The Stablecoin Stack makes this model practical for commerce by combining:

- **ERC-2612 stablecoins** — price-stable tokens with built-in off-chain signing support;
- **a gasless relayer architecture** — so payers never need to hold or spend native gas tokens; and
- **a familiar checkout experience** — so merchants and users interact with concepts they already understand.

---

## Who Is This For?

| Reader | What to read |
|--------|-------------|
| Anyone wanting to understand the system | Start here, then [Why Cryptographic Payments](../core-concepts/why-cryptographic-payments.md) |
| Developers building a wallet or integration | [System Architecture](../core-concepts/system-architecture.md), then [SSF-SPEC-001](../specifications/SSF-SPEC-000-system-overview.md) |
| Smart-contract engineers | [SSF-SPEC-002](../../02-the-settlement-contract-canonical/specifications/SSF-SPEC-002-settlement-contract.md) |
| Auditors and security reviewers | [Security Model](../specifications/SSF-SPEC-000-system-overview.md#security-model) and [Threat Model](../core-concepts/system-architecture.md#trust-boundaries) |
| Contributors | [Governance](../governance/versioning-policy.md) |

---

## Scope

The Stablecoin Stack specification series covers:

- the on-chain **Settlement Contract** — permit verification, token transfer, fee distribution, and acquirer registration;
- the **payload structures and signing conventions** by which a wallet presents a payment authorisation to a processor;
- the **Checkout Engine** — session creation, charge lifecycle, and settlement reconciliation;
- the **Broadcast Layer** — off-chain validation, queuing, and on-chain submission;
- the **Event Indexing** service — monitoring confirmed on-chain events;
- the **Basic Data Service** — shared reference data for all components; and
- the **Acquiring Model** — how third-party participants distribute the service and earn fees.

### Out of Scope

The following are explicitly not addressed by this specification series:

- **Private key management and custody** — how keys are generated, stored, backed up, or recovered is a wallet implementation concern.
- **Token issuance and redemption** — the mechanisms by which stablecoins are minted and backed.
- **Regulatory compliance** — operators are responsible for applicable legal requirements.
- **Offline transaction signing** — identified as future work; see below.

---

## Specification Series

| ID | Title | Layer | Status |
|----|-------|-------|--------|
| SSF-SPEC-000 | The Stablecoin Stack: System Overview and Architecture | This document | Draft |
| [SSF-SPEC-001](../../01-instant-payment-with-permitted-token-transfer-submission/specifications/SSF-SPEC-001-submission.md) | Instant Payment With Permitted Token Transfer — Submission | Protocol | Draft |
| [SSF-SPEC-002](../../02-the-settlement-contract-canonical/specifications/SSF-SPEC-002-settlement-contract.md) | The Settlement Contract — Canonical | On-chain | Draft |

---

## Future Work

The Foundation has identified the following as priorities for future specification and implementation work:

**Offline Transaction Signing** — Constructing and signing a payment commitment without a live network connection. Of particular importance for populations with limited mobile connectivity; potentially transformative for financial access.

**Multi-Network Support** — Extending conformance beyond Ethereum-compatible networks to other EVM chains and, eventually, non-EVM networks with equivalent programmable token standards.

**Checkout Engine API Specification** — A formal specification of the merchant-facing API: session creation, charge lifecycle, webhook delivery, and reconciliation queries.

**Broadcast Layer Protocol Specification** — A formal specification of the WebSocket protocol between wallet and broadcast-gateway: message formats, state machine, and error handling.

**Wallet Certification Programme** — A conformance test suite enabling users and merchants to verify that a given wallet correctly implements the signing and submission protocols.

**Acquirer Incentive Structures** — Formal specification of fee tier models available to acquirers, enabling richer market structures while preserving on-chain enforceability.
