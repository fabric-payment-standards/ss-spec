# Why Cryptographic Payments

This document explains the foundational reasoning behind the Stablecoin Stack: why cryptographic payment commitments represent a meaningful improvement over existing payment models, what role stablecoins play in making this practical, and why the Foundation believes experimentation is urgent despite the real challenges ahead.

---

## The Problem with Delegated Trust

The dominant model of payment authorisation today is one of **delegated trust**. When a cardholder taps a card or enters their details online, they are not authorising the payment directly — they are instructing an institution to authorise it on their behalf. The credential they present (a card number, a PIN, a one-time code) is a token of identity that the institution recognises. The institution then decides whether to honour the payment.

This model has served commerce well for decades. It is also, however, fundamentally asymmetric:

- The institution holds the authoritative record of consent. The user holds nothing.
- Disputes, reversals, and fraud decisions are resolved by institutional policy, not by verifiable proof.
- The user's ability to pay depends entirely on the institution's willingness to act on their behalf.
- Every payment leaks identity information — card numbers, billing addresses, transaction histories — to intermediaries who may not protect it.

None of this is a failure of the people who built these systems. It is a consequence of building payment infrastructure before the tools for cryptographic proof at scale existed.

Those tools now exist.

---

## Cryptographic Payment Commitments

A **cryptographically signed payment commitment** is fundamentally different from a credential-based authorisation.

When a payer signs a payment using their private key, they produce a piece of data that:

- **carries its own proof of authenticity** — no third party needs to vouch for it;
- **is verifiable by anyone** with access to the corresponding public key;
- **is bound to specific parameters** — the amount, the recipient, the token, the deadline — so it cannot be redirected or reused without invalidating the signature; and
- **is unforgeable** without the payer's private key.

The payer's intent is encoded in the data itself. The payment commitment is the authorisation. Infrastructure is needed only to execute and settle it — not to validate it.

This represents a shift in the **locus of authority**: from institution to individual, from delegated trust to verifiable proof. The payer is a principal, not a subject. Their key is their credential. Their signature is their consent.

We believe this shift has profound implications for the accessibility, portability, and integrity of payment infrastructure — implications that are only beginning to be understood.

---

## The Role of Stablecoins

Cryptographically signed payment commitments require a **programmable money layer** — a form of value transfer that can be authorised by a signature and executed deterministically by software. Native cryptocurrencies provide this. Their price volatility, however, makes them unsuitable as a unit of account for everyday commerce. A merchant cannot price goods in an asset that may lose 20% of its value overnight.

**Stablecoins** resolve this tension. A stablecoin is a token whose value is pegged to a stable reference asset — typically a fiat currency. It combines:

- the **programmability** of on-chain tokens (transferable by smart contract, authorisable by signature);
- the **self-custody** properties of cryptocurrency (held in a wallet the user controls); and
- the **price stability** required for practical commerce.

For the purposes of this specification series, a compliant stablecoin is any ERC-20 token that additionally implements the **ERC-2612 permit extension**. ERC-2612 is the mechanism that makes the Stablecoin Stack's gasless model possible.

### ERC-2612 and Gasless Payments

Normally, authorising a smart contract to spend tokens requires an on-chain `approve()` transaction — which itself requires the user to hold and spend native gas tokens. ERC-2612 eliminates this step by allowing the approval to be expressed as an **off-chain signature** using EIP-712 typed data. The signature can be forwarded to any party who then submits it on-chain at their own expense.

The consequence for the Stablecoin Stack is that:

- the payer signs their payment commitment entirely off-chain, spending no gas;
- the Relayer (operated by the Payment Processor) submits the transaction and pays gas; and
- the payer need only hold the stablecoin they wish to spend.

Gas is an infrastructure cost. The Stablecoin Stack places it where it belongs — with the infrastructure operator.

---

## Vision and Design Principles

Every architectural decision in the Stablecoin Stack follows from a small number of explicit principles.

### User-Centric by Design

Technical complexity must not be a barrier to making a payment. The end user should encounter a flow as simple as any mobile payment: open a wallet, review the details, confirm. All complexity — signature construction, nonce management, fee calculation, gas payment — is absorbed by the infrastructure. This principle applies equally to merchants: a merchant integrating the stack should work with familiar concepts (charges, sessions, webhooks, reconciliation) and need not understand ERC-2612 or EIP-712.

### Payer Sovereignty

The authority to authorise a payment belongs exclusively to the payer and is expressed through cryptographic signature. No component of the stack — processor, relayer, checkout engine — can move funds on a payer's behalf without a valid signed authorisation. The signature is the payment commitment. Everything else is infrastructure for delivering and executing it.

A corollary is that the system is **payer-agnostic**: the identity of the account that submits a transaction to the network is irrelevant to its validity. Provided the signatures are correct, the payment executes. This enables the gasless relayer model and removes any coupling between the payer's identity and their ability to transact.

### Gas Abstraction as a Baseline

Requiring users to hold native gas tokens is a systems problem, not a user problem. Gas abstraction — where the Relayer pays gas and recovers costs through the processing fee — is not a premium feature. It is the default. The user holds only the stablecoin they wish to spend.

### Complementary Rails, Not Replacement

The Stablecoin Stack does not aim to replace existing payment infrastructure. For communities with reliable mobile network access, it offers **complementary rails** — particularly well-suited to cross-border transfers, privacy-preserving payments, contexts where the parties lack a shared banking relationship, and scenarios where programmable settlement logic adds value.

For communities with limited connectivity, the vision extends further. **Offline transaction signing** — constructing and authenticating a payment commitment without a live network connection — may be transformative for populations that today have limited access to financial services. The Foundation has identified this as a priority area for future specification work.

### Open by Default

All specifications and reference implementations are published under open licences. There are no proprietary extensions and no tollgates on conformance information. The Foundation's value is in the quality of the specification, not in controlling access to it.

---

## The Challenges Are Real

The Foundation does not approach this space with naive optimism. The obstacles to practical stablecoin payment adoption are significant and deserve honest acknowledgement.

**Key management** remains the most serious usability barrier. Losing a private key means losing access to funds, with no institutional recourse. Meaningful progress requires better tooling, better defaults, and sustained investment in education — not hand-waving.

**Network access** is uneven. Reliable mobile connectivity, which this system's primary flow requires, is not universal. The Foundation has identified offline transaction signing as a priority area for future work, but this work has not yet begun.

**Regulatory clarity** is still developing in most jurisdictions. Operators deploying compliant stacks must navigate this landscape with care and qualified legal counsel.

**Ecosystem maturity** — wallets, merchant tooling, consumer familiarity — requires sustained investment that no single organisation can provide alone.

---

## Why Act Now

The correct response to these challenges is not to wait. The infrastructure being specified here is novel. Building it, testing it, documenting it, and sharing it openly is itself the work. **Experimentation and education are not preparatory steps before the real work begins — they are the real work.**

Every deployed instance advances collective understanding of what works and what does not. Every open-source contribution reduces the cost of the next implementation. Every specification published sets a baseline that future work can build on or improve.

The foundations laid today — in specifications, in reference implementations, in open tooling — will determine how accessible and interoperable this infrastructure becomes when adoption accelerates. We believe that moment is closer than it appears, and we intend to be ready for it.
