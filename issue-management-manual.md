# Issue Management Manual

This document defines the basic rules for creating, classifying, and managing issues in the repository.  
The goal is to keep discussions structured, reproducible, and easy to maintain without unnecessary bureaucracy.

This is **not** a strict bureaucracy. Use common sense.  
However, issues that do not follow these basic rules may be closed or returned for clarification.

---

# 1. General Principles

When opening an issue:

- Be **clear**
- Be **concise**
- Provide **enough information for others to understand and reproduce the problem**

Assume the person reading your issue has **no prior context**.

An issue should allow someone else to:

1. Understand the problem
2. Reproduce the problem
3. Propose a solution

---

# 2. When to Create an Issue

Create an issue when:

- A **bug** is discovered
- The **specification needs clarification or correction**
- An **implementation does not conform to the specification**
- **Documentation needs improvement**
- A **security concern** is identified
- A **new feature or improvement** is proposed
- **Research or exploration** is needed before design decisions

Do **not** create issues for:

- Personal notes
- Unclear ideas without context
- Questions that belong in discussions

---

# 3. Issue Categories

Each issue must include **one `Type` label**.

| Type | Description |
|-----|-----|
| type: specification | A change, correction, or improvement to the specification. |
| type: implementation | A change required in the reference implementation or prototype. |
| type: documentation | Changes needed in documentation without altering behavior. |
| type: internationalization | Issues related to translation of specifications, documentation, or interfaces. |
| type: clarification | The specification is correct but unclear or ambiguous. |
| type: enhancement | A proposal to extend functionality or capabilities. |
| type: security | A vulnerability, security risk, or cryptographic concern. |
| type: compliance | An implementation does not follow the specification correctly. |
| type: research | Investigation or exploration before design decisions are made. |
| type: tooling | Problems related to development tools, CI/CD, testing infrastructure, or build systems. |
| type: governance | Issues related to project governance, policies, or contribution processes. |

---

# 4. Scope Labels

Scope labels identify **which component is affected**.

Multiple scopes may be used if necessary.

Examples:

| Scope | Description |
|-----|-----|
| scope: specification | The specification itself |
| scope: smart-contract | Smart contracts |
| scope: checkout | Payment checkout system |
| scope: wallet | User wallet implementation |
| scope: merchant-dashboard | Merchant dashboard |
| scope: sdk | Developer SDK |
| scope: reference-implementation | Reference prototype |
| scope: documentation | Documentation |
| scope: website | Public documentation website |

---

# 5. Priority Labels

Priority labels indicate urgency.

| Priority | Description |
|-----|-----|
| priority: critical | Security issues or problems blocking core functionality |
| priority: high | Important issues affecting major functionality |
| priority: medium | Normal issues |
| priority: low | Minor improvements or non-urgent issues |

---

# 6. Status Labels

Status labels reflect the progress of an issue.

| Status | Description |
|-----|-----|
| status: needs-triage | Issue requires review |
| status: accepted | Issue confirmed and accepted |
| status: in-progress | Work is currently being done |
| status: blocked | Progress blocked by another issue |
| status: review | Solution proposed and under review |
| status: waiting-feedback | Waiting for additional information |

---

# 7. Writing a Good Issue

A good issue contains:

### Title

Short and descriptive.

**Good:** Wallet fails to validate signed checkout payload


**Bad:** Wallet problem


---

### Description

Explain the problem clearly.

Include:

- What happened
- What was expected to happen
- Why the issue matters (if relevant)

---

### Steps to Reproduce (for bugs)

Provide a minimal reproducible sequence.

Example:


Open the wallet

Generate a checkout payload

Submit the payload to the processor

Signature validation fails


---

### Environment (when applicable)

Provide context when relevant:


- Wallet version: 0.4.2
- Device: Androind Emulator
- Network: Polygon PoS


---

### Evidence

Include supporting material when appropriate:

- Logs
- Error messages
- Screenshots
- Code snippets

---

# 8. Screenshots

Screenshots must be included when the issue involves:

- User interfaces
- Visual bugs
- Incorrect rendering
- Dashboard errors

Guidelines for screenshots:

- Capture **only the relevant part**
- Ensure **sensitive information is removed**
- Use **clear and readable resolution**

---

# 9. Proposals and Enhancements

When proposing a change or improvement, include:

- The **problem being solved**
- The **proposed solution**
- Possible **alternatives**
- Potential **impact**

Major proposals may eventually lead to **a new specification version**.

---

# 10. Security Issues

Security issues must be handled carefully.

If the issue involves:

- Cryptographic vulnerabilities
- Smart contract exploits
- Private key risks
- Attack vectors

Avoid publishing full exploit details immediately.

Report responsibly so maintainers can evaluate the risk before public disclosure.

---

# 11. Reproducibility Rule

If another contributor **cannot reproduce the issue**, it may be labeled:


status: waiting-feedback


If sufficient information is not provided after a reasonable period, the issue may be closed.

---

# 12. Keep Discussions Focused

Issues should remain focused on **one problem**.

Do not mix unrelated topics in the same issue.

If a discussion expands into a new topic, create a **separate issue**.

---

# 13. Final Notes

This repository is a collaborative environment.

Good issues help the project:

- move faster
- avoid misunderstandings
- maintain high quality standards

When in doubt, prefer **clarity over volume**.