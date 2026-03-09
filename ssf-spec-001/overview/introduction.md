---
document: SSF-SPEC-001/overview/introduction.md
spec: SSF-SPEC-001
version: 1.0.0
status: Draft
date: 2026-03-04
author: "Adalton Reis <reis@stablecoinstack.org>"
organization: Stablecoin Stack Foundation
license: Apache License 2.0
---

# Introduction

The Stablecoin Stack is an open architecture for processing stablecoin payments on Ethereum-compatible networks. It is built on a single central proposition: that cryptographic signatures are a superior foundation for payment authorisation compared to identity-based delegation models, and that the tooling to make this practical for everyday users is both achievable and urgently needed.

A working prototype exists. This specification series formalises that prototype for production implementation, independent audit, and institutional review. The Foundation does not theorise and then build — it builds, tests, and then specifies. Everything documented here has been demonstrated to work.

---

## What the Stablecoin Stack Is

The Stablecoin Stack defines a complete, custody-free payment protocol. A payer signs two off-chain authorisations — no blockchain transaction required from them, no gas token needed. A Relayer operated by the Payment Processor submits the transaction on their behalf. A Settlement Contract verifies the signatures, moves the tokens, distributes fees, and registers acquirers — atomically, non-custodially, and in a single on-chain transaction.

The system is designed around three guarantees that are enforced at the contract level, not by policy:

- **The payer's funds cannot be moved without their signature.** No operator, no processor, no acquirer can authorise a transfer on the payer's behalf.
- **The administrator cannot touch participant funds.** Fee parameters can be adjusted; balances cannot be drained.
- **Every operation either completes entirely or reverts entirely.** There is no partial execution.

---

## Who This Is For

This repository is the authoritative specification source for the Stablecoin Stack. It is written for four distinct audiences, each of whom has a different entry point:

**Developers** who want to build against the protocol — integrate a wallet, implement a processor, or deploy a Settlement Contract — should start with the [core concepts](../core-concepts/) to build a mental model, then use the [guides](../guides/) for step-by-step implementation, and consult the [formal specification](../specifications/ssf-spec-001.md) and [reference](../reference/) for normative detail.

**Institutions** evaluating the protocol for adoption, integration, or partnership should read this document, then [Why Cryptographic Payments](../core-concepts/why-cryptographic-payments.md) for the design rationale, and [System Architecture](../core-concepts/system-architecture.md) for component boundaries and trust model.

**Auditors** assessing cryptographic correctness, security model, or protocol conformance should go directly to the [formal specification](../specifications/ssf-spec-001.md), paying particular attention to Sections 11 (Security Model) and 12 (Conformance). The [normative references](../reference/normative-references.md) list all cited standards.

**Regulators** reviewing custody model, consumer protection, or systemic risk should read this document and [System Architecture](../core-concepts/system-architecture.md), focusing on the trust boundaries and administrative privilege constraints in Section 11.7 of the formal specification.

---

## Scope

The current specification covers:

- the on-chain Settlement Contract (permit verification, token transfer, fee distribution, acquirer registration)
- the payload structures, signing conventions, and submission protocol for the Payment Processor API
- the system architecture, component responsibilities, and trust model for a fully conformant deployment
- the acquiring model and fee distribution mechanics
- the end-to-end payment flow from session creation through settlement confirmation

The following are planned for subsequent specifications and are explicitly out of scope here:

- Checkout Engine API (merchant-facing session and charge management)
- Broadcast Layer WebSocket protocol (wallet-to-gateway communication)
- Individual microservice interface specifications
- Key management and custody procedures
- Token issuance and redemption mechanics
- Regulatory compliance guidance

---

## Status and Contributing

All content in this repository is in **Draft** status. The Foundation actively seeks contributors across all profiles — protocol reviewers, implementers, security auditors, and legal and regulatory professionals. See [governance/contributing.md](../governance/contributing.md) for how to participate.

---

## Related Documents

- [Why Cryptographic Payments](../core-concepts/why-cryptographic-payments.md)
- [System Architecture](../core-concepts/system-architecture.md)
- [Formal Specification — SSF-SPEC-001](../specifications/ssf-spec-001.md)
