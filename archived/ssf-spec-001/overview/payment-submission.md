---
sidebar_position: 2
---

# Payment Submission

## What It Is

Payment submission is the process by which a client wallet constructs a signed payload and delivers it to the Payment Processor API, which then broadcasts the corresponding transaction on-chain.

No native gas token is required from the payer. The Relayer holds a funded account and covers gas costs. The payer's only on-chain footprint is an ERC-2612 permit signature — produced entirely off-chain.

## Who It Involves

| Participant        | Role                                                                                              |
| ------------------ | ------------------------------------------------------------------------------------------------- |
| **Payer**          | Holds the stablecoin. Signs two off-chain authorisations and submits the payload.                 |
| **Payment Processor** | Receives the payload over HTTPS, validates it, and forwards the transaction to the Relayer.    |
| **Relayer**        | Broadcasts the on-chain transaction and pays the gas fee.                                         |
| **Settlement Contract** | Verifies the signatures, executes the token transfer, and distributes fees atomically.       |
| **Acquirer**       | A registered participant entitled to a share of the processing fee on payments they referred.     |

## Two Submission Flows

### 1. Payment Transfer

A payer sends a stablecoin amount to a merchant. The payload type is `TransferRequest`. The processor validates it and the Settlement Contract transfers the principal to the merchant while distributing fees.

### 2. Acquirer Service Purchase

A participant pays a one-time registration fee to become an Acquirer. The payload type is `BuyAcquiringPackRequest`. Upon settlement, the participant's wallet address is registered on-chain and assigned a unique Acquirer ID.

## What This Layer Covers

The submission layer — defined in **SSF-SPEC-001** — governs everything between the client wallet and the Payment Processor API:

- how payloads are structured
- what signatures must be produced and how
- what validation the processor must enforce before broadcasting

It does **not** cover smart-contract internals, checkout session management, or key-management procedures. Those are addressed in companion specifications.

## Related Documents

- [Permit-Based Payments](../core-concepts/permit-based-payments.md) — conceptual explanation of how gasless transfers work
- [SSF-SPEC-001](../specifications/ssf-spec-001.md) — the full formal specification
- [Submit a Payment](../guides/submit-a-payment.md) — step-by-step integration guide
- [Buy an Acquiring Pack](../guides/buy-acquiring-pack.md) — step-by-step guide for acquirer registration
- [Payload Fields Reference](../reference/payload-fields.md) — complete field-level reference
