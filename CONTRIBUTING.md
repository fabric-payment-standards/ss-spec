# Contributing to Stablecoin Stack

Welcome! This document explains how to contribute to the **Stablecoin Stack Foundation** repositories.

We organize our work following the **Layered Docs model**:

- **Spec repo (`stablecoin-stack-spec`)**: specifications, governance, tutorials, and developer guides.
- **Prototypes (`prototype-checkout`, `prototype-wallet`, etc.)**: reference implementations.
- **Website repo (`stablecoin-stack-website`)**: builds documentation from the spec repo.

All contributions must respect this structure.

---

## 1. Reporting Issues

- Use **clear, descriptive titles**.
- Provide **enough detail for reproduction**:
  - Steps to reproduce (if applicable)
  - Environment / version info
  - Screenshots or logs (for UI or bug issues)
- **Classify issues** with the proper labels (`type`, `scope`, `priority`, `status`).
- Reference the **Issue Management Manual**:  
  [Issue Management Manual](./governance/issue-management.md)

---

## 2. Proposing Changes

- **Fork the repo** and create a branch for your changes.
- Make **small, focused commits** with clear messages.
- Include references to **issue numbers** when relevant:  


git commit -m "Fix wallet signature validation (#42)"


- Submit a **Pull Request (PR)**.
- PR title should be clear and concise.
- PR description must explain **what changed and why**.
- Link to relevant **issues or specifications**.

---

## 3. Specifications

- Keep the **core specification coherent**.
- Split content into **short, readable modules** (10–20 min reading each).
- Avoid mixing:
- Concepts
- Normative rules
- Tutorials / guides
- Include **references to implementations** where relevant.

---

## 4. Implementations

- Each prototype is a **separate repository**.
- Reference the specification version you are implementing.
- Use **tests, examples, and CI** to verify conformance.
- Link your repo back to the spec repo and guides.

---

## 5. Documentation & Tutorials

- Write **short, task-focused guides**.
- Use **screenshots and examples** where relevant.
- Keep tutorials separate from normative specifications.
- All documentation changes must **link to a relevant specification section**.

---

## 6. Governance

- All repositories follow the **foundation-wide governance rules**.
- Decisions are tracked in the **spec repo governance folder**:


stablecoin-stack-spec/governance/


- Check the **Issue Management Manual** before opening issues or PRs.

---

## 7. Communication

- Be **professional and respectful**.
- Questions or discussions are better suited for **issues or discussion threads**, not PR comments alone.
- Follow the **CODE_OF_CONDUCT.md** at all times.

---

## 8. Quick Reference

| Action                     | Location/Link |
|------------------------------|---------------|
| Contribution process         | This file (`CONTRIBUTING.md`) |
| Spec repository              | `stablecoin-stack-spec` |
| Tutorials & developer guides | `stablecoin-stack-spec/guides` |
| Governance rules             | `stablecoin-stack-spec/governance/` |
| Issue rules                  | `stablecoin-stack-spec/governance/issue-management.md` |
| Prototypes                   | `prototype-<name>` repos |
| Website                      | `stablecoin-stack-website` |

---

By following this guide, you help the foundation **stay organized, maintain high quality, and onboard contributors faster**.