# Stablecoin Stack Foundation — Specification Repository

The Stablecoin Stack Foundation publishes open, implementation-neutral specifications for stablecoin payment infrastructure on Ethereum-compatible networks. This repository is the authoritative source for all SSF specifications and their supporting documentation.

All specifications are in **Draft** status. We are actively seeking contributors, reviewers, and institutional partners. See [Contributing](#contributing) below.

---

## What This Is

The Stablecoin Stack defines a complete, custody-free payment protocol built on ERC-2612 permit-based token transfers. Payers sign authorisations off-chain; a designated relayer broadcasts transactions on-chain; a Settlement Contract verifies signatures, executes transfers, and distributes fees — atomically, without holding user funds.

A working prototype exists. This specification series formalises the protocol for production implementation, independent audit, and institutional review.

---

## Specifications

| ID | Title | Status | Description |
| -- | ----- | ------ | ----------- | 
| [SSF-SPEC-001](./ssf-spec-001/) | Stablecoin Stack Specification | Draft |The foundational specification of the **Stablecoin Stack** — an open architecture for processing stablecoin payments on Ethereum-compatible networks. |
| [SSF-SPEC-002](./ssf-spec-002/) |  Wallet-gateway Interface Specification | Draft | This specification defines the normative interface that a conformant wallet-gateway MUST implement, and that a conformant wallet client MUST be capable of consuming|

---

## Start Here — By Audience

### Developers
You want to build against the protocol — integrate a wallet, implement a processor, or deploy a Settlement Contract.

1. Read [SSF-SPEC-001 — System Overview](./ssf-spec-001/overview/) to understand the architecture.
2. Read [SSF-SPEC-001 — Payment Submission](./ssf-spec-001/) for the client-side protocol.
3. Read [SSF-SPEC-002 — Settlement Contract](./ssf-spec-002/) for the on-chain interface.
4. Use the [guides](./ssf-spec-001/guides/) to walk through implementation step by step.

### Institutions
You are evaluating the protocol for adoption, integration, or investment.

1. Read [SSF-SPEC-001 — Introduction](./ssf-spec-001/overview/) for system scope and design rationale.
2. Read [Why Cryptographic Payments](./ssf-spec-001/core-concepts/) for the motivation and design philosophy.
3. Review the [Architecture Overview](./ssf-spec-001/core-concepts/) for component boundaries and trust model.

### Auditors
You are assessing cryptographic correctness, security model, or protocol conformance.

1. Start with [SSF-SPEC-001 — Formal Specification](./ssf-spec-001/specifications/ssf-spec-001.md) for the submission protocol.
2. Continue with [SSF-SPEC-002 — Formal Specification](./ssf-spec-002/specifications/ssf-spec-002.md) for the on-chain contract interface.
3. Review [Security Considerations](./ssf-spec-001/specifications/ssf-spec-001.md#10-security-considerations) in each spec.
4. See [Normative References](./ssf-spec-001/reference/normative-references.md) for all cited standards.

### Regulators
You are reviewing the protocol for compliance, consumer protection, or systemic risk assessment.

1. Read [SSF-SPEC-001 — Introduction](./ssf-spec-001/overview/) for scope and purpose.
2. Read [System Architecture](./ssf-spec-001/core-concepts/) for custody model and participant roles.
3. Review [Security Considerations](./ssf-spec-002/specifications/ssf-spec-002.md#9-security-considerations) for non-custodial guarantees and administrative privilege boundaries.

---

## Repository Structure

Each specification is self-contained under its own folder, organised into six layers:

```
ssf-spec-xxx/
  overview/          Quick orientation — what the spec covers and why it exists
  core-concepts/     Mental models and conceptual understanding
  specifications/    Formal normative definitions (the spec itself)
  reference/         Lookup-optimised technical reference tables
  guides/            Step-by-step implementation tutorials
  governance/        Changelog and spec-level governance
```
 
---

## Contributing

This project is in active development. Contributions are welcome across all profiles:

- **Protocol review** — read the formal specifications and open an issue for any ambiguity, gap, or error you find.
- **Implementation** — build against the spec and report any friction or missing detail.
- **Security audit** — review cryptographic assumptions and signature schemes.
- **Legal and regulatory** — provide professional opinion on compliance considerations.
>##
>**Important:** 
>- Please read [CONTRIBUTING.md](./CONTRIBUTING.md) before opening a pull request or issue.
>- Report issues according to the [**Issue Management Manual**](./issue-management-manual.md) 
>- Respect the [**Code of Conduct**](./CODE_OF_CONDUCT.md)
>##

---


## Prototypes & Implementations

Reference **implementations** live in dedicated repositories:

|Prototype	|Repository link|
|-----------|---------------|
|Checkout Engine	|prototype-checkout|
|Merchant Dashboard |prototype-marchant-dashboard|
|Wallet	|prototype-mobile-wallet|
|SDK	|prototype-sdk|

---
## Websites

 - Specifications

The public **Specifications** be better read/visualized in the [specifications website](https://specifications.stablecoinstack.org/). The site is built from this repository:

- Repository: [REPO](https://github.com/Stablecoin-Stack/specs-website) stablecoin-stack-website
---

## Support the Foundation

The Stablecoin Stack Foundation is an independent, non-profit initiative. Every specification, reference implementation, and piece of documentation we produce is made freely available to builders, auditors, and the broader open-source community — with no paywalls and no proprietary lock-in.

This work is sustained entirely by the generosity of individuals and organisations who believe that open, well-specified payment infrastructure is a public good worth building. If our work has saved you time, informed your architecture, or contributed to something you are proud of, please consider giving back.

**Your support directly funds:**
- research and authorship of new specifications;
- independent security review and formal audits of published standards;
- tooling, reference implementations, and developer documentation; and
- the operational continuity of the Foundation itself.

### Donate in Cryptocurrency

| Network | Address |
|---------|---------|
| Ethereum-base networks (ETH / ERC-20, etc)| `0x0000000000000000000000000000000000000000` |
| Solana (SOL / SPL tokens) | `11111111111111111111111111111111` |
| Bitcoin (BTC) | `bc1qplaceholderaddressplaceholderaddress` |

All on-chain donations are publicly verifiable and gratefully acknowledged. If you would like your contribution attributed, please send us a note at **info@stablecoinstack.org** with your transaction hash.

### Donate in Fiat

For wire transfers, invoiced sponsorships, or institutional giving arrangements, please reach out to us directly at **info@stablecoinstack.org**. We are happy to provide any documentation required by your finance or compliance team.

---

## License

Copyright © 2026 Stablecoin Stack Foundation. All rights reserved.

All specifications and documentation in this repository are published under the [Apache License, Version 2.0](https://www.apache.org/licenses/LICENSE-2.0).
