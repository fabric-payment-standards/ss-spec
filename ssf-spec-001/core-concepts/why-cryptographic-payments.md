---
document: SSF-SPEC-001/core-concepts/why-cryptographic-payments.md
spec: SSF-SPEC-001
version: 1.0.0
status: Draft
date: 2026-03-04
author: "Adalton Reis <reis@stablecoinstack.org>"
organization: Stablecoin Stack Foundation
license: Apache License 2.0
---

# Why Cryptographic Payments

---

## The Problem with Delegated Trust

The dominant model of payment authorisation today is one of delegated trust. A user instructs a financial institution, which in turn authorises a payment on their behalf. The user's intent is represented by a credential — a card number, a PIN, a one-time code — which is transmitted to and verified by a chain of intermediaries. At no point in this flow does the payment itself carry a verifiable proof of the payer's consent. The proof lives in the intermediary's records.

This architecture has served commerce well for decades. It is also fundamentally asymmetric: the intermediaries hold the authoritative record of consent, and the user holds nothing. Disputes, reversals, and fraud mitigation are resolved by institutional decision, not by cryptographic proof.

## The Alternative: Verifiable Commitment

A cryptographically signed payment commitment is a piece of data that carries its own proof of authenticity. It does not require a third party to vouch for it. Anyone with access to the public key can verify that the holder of the corresponding private key produced it. The payer's intent is encoded in the data itself, bound to the specific parameters of the specific payment, and unforgeable without the payer's key.

This is not merely a technical preference. It is a meaningful shift in the locus of authority: from institution to individual, from delegated trust to verifiable proof. The implications for accessibility, portability, and integrity of payment infrastructure are significant — and only beginning to be understood.

## The Role of Stablecoins

Cryptographically signed payment commitments require a programmable money layer. Native cryptocurrencies provide this, but their price volatility makes them unsuitable as a unit of account for commerce.

Stablecoins — tokens pegged to a stable reference asset, typically a fiat currency — resolve this tension. They combine the programmability and self-custody properties of on-chain tokens with the price stability required for practical commerce. For the Stablecoin Stack, a compliant stablecoin is any ERC-20 token that implements the **ERC-2612 permit extension**, enabling off-chain signed approvals without a separate on-chain allowance transaction.

ERC-2612 is central to the gasless payment model. The permit mechanism allows a payer to sign an authorisation entirely off-chain — no transaction required, no gas spent — and for a Relayer to submit the resulting proof on-chain on their behalf.

## The Imperative to Act Now

The Foundation does not approach this space with naive optimism. The challenges are real:

**Key management** remains the most serious usability barrier. Losing a private key means losing access to funds, with no institutional recourse. Meaningful progress requires better tooling, better defaults, and better education.

**Network access** is uneven. Reliable mobile connectivity, which this system's primary flow requires, is not universal. Offline transaction signing is identified as a priority area for future specification work.

**Regulatory clarity** is still developing in most jurisdictions. Operators of compliant stacks must navigate this landscape with care and legal counsel.

**Ecosystem maturity** — wallets, merchant tooling, and consumer familiarity — requires sustained investment.

Despite these challenges, the correct response is not to wait. The infrastructure specified here is novel. Building it, testing it, documenting it, and sharing it openly is itself the work. Every deployed instance, every open-source contribution, and every specification published advances the state of collective understanding. Experimentation and education are not preparatory steps before the real work begins — they are the real work.

---

## Related Documents

- [Introduction](../overview/introduction.md) — system scope and entry point
- [System Architecture](./system-architecture.md) — how the principles translate into components
- [Permit-Based Payments](./permit-based-payments.md) — the technical mechanism in detail
