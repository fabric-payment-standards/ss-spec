---
document: SSF-SPEC-001/overview/payment-submission.md
spec: SSF-SPEC-001
version: 1.0.0
status: Draft
date: 2026-03-04
author: "Adalton Reis <reis@stablecoinstack.org>"
organization: Stablecoin Stack Foundation
license: Apache License 2.0
---

# Payment Submission

Payment submission is the process by which a Client Wallet constructs a signed payload and delivers it to the Payment Processor API, which validates it and broadcasts the corresponding transaction on-chain.

No native gas token is required from the payer. The Relayer holds a funded account and covers all gas costs. The payer's only actions are two off-chain signatures and one HTTPS request.

---

## Who It Involves

| Participant | Role |
| ----------- | ---- |
| **Payer** | Holds the stablecoin. Signs two off-chain authorisations and submits the payload. |
| **Payment Processor** | Receives the payload over HTTPS, validates all fields and signatures, forwards the transaction to the Relayer. |
| **Relayer** | Broadcasts the on-chain transaction and pays the gas fee. Operated by the Payment Processor. |
| **Settlement Contract** | Verifies the signatures, executes the token transfer, and distributes fees atomically. |
| **Acquirer** | A registered participant entitled to a share of the processing fee on payments they referred. |

---

## Two Submission Flows

### Payment Transfer

A payer sends a stablecoin amount to a merchant beneficiary. The payload type is `TransferRequest`. The Settlement Contract transfers the principal to the merchant after deducting fees.

### Acquirer Registration

A participant pays a one-time fee to become a registered Acquirer. The payload type is `BuyAcquiringPackRequest`. Upon settlement, the participant's wallet is registered on-chain with a unique Acquirer ID.

---

## Where It Fits in the Stack

```
Client Wallet
     │
     │  two EIP-712 signatures + HTTPS POST
     ▼
wallet-gateway  ──►  broadcast-service  ──►  broadcast-submitter (Relayer)
                                                      │
                                                      │  on-chain transaction
                                                      ▼
                                             Settlement Contract
```

---

## Related Documents

- [Permit-Based Payments](../core-concepts/permit-based-payments.md) — how gasless transfers work
- [Dual-Signature Pattern](../core-concepts/dual-signature-pattern.md) — why two signatures are required
- [Submit a Payment](../guides/submit-a-payment.md) — step-by-step guide
- [Buy an Acquiring Pack](../guides/buy-acquiring-pack.md) — acquirer registration guide
- [Payload Fields Reference](../reference/payload-fields.md) — complete field-level reference
