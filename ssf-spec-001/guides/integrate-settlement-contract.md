---
document: SSF-SPEC-001/guides/integrate-settlement-contract.md
spec: SSF-SPEC-001
version: 1.0.0
status: Draft
date: 2026-03-04
author: "Adalton Reis <reis@stablecoinstack.org>"
organization: Stablecoin Stack Foundation
license: Apache License 2.0
---

# Integrate the Settlement Contract

Guide for Payment Processor operators who need to deploy, configure, and integrate the Settlement Contract into a running stack.

**Prerequisites:** read [System Architecture](../core-concepts/system-architecture.md) and [Fee Model](../core-concepts/fee-model.md) first.

---

## What the Settlement Contract Does

The Settlement Contract is the only component of the Stablecoin Stack that holds and moves funds. It is deployed once per processor, and its address is published to the Basic Data Service so wallets and other components can discover it. All payments and acquirer registrations flow through it.

It has no upgrade mechanism. Its administrative functions are restricted to fee parameters and the fee recipient address. It cannot be used to drain participant funds. See [Security Model](../specifications/ssf-spec-001.md#18-security-model) for the full administrative privilege boundaries.

---

## Step 1 — Deploy

Deploy the contract with constructor parameters:

```solidity
constructor(
    uint256 _baseFeeAmount,       // initial base fee in token units
    uint256 _maxAcquiringFee,     // ceiling for acquirer fee percentages
    uint256 _acquiringPrice,      // registration fee in token units
    address _feeRecipient,        // address to receive processor base fees
    IERC20Permit[] _allowedTokens // tokens accepted for acquirer registration
)
```

Example deployment (Hardhat):

```javascript
const SettlementContract = await ethers.getContractFactory("SettlementContract");
const contract = await SettlementContract.deploy(
  ethers.parseUnits("2.50", 6),    // $2.50 USDC base fee (6 decimals)
  500,                              // maxAcquiringFee = 5.00% (processor-defined units)
  ethers.parseUnits("100", 6),     // $100 USDC acquirer registration price
  feeRecipientAddress,
  [usdcAddress, eurcAddress],
);
await contract.waitForDeployment();
console.log("Settlement Contract:", await contract.getAddress());
```

After deployment, publish the contract address to the basic-data-server.

---

## Step 2 — Fund the Relayer

The broadcast-submitter holds the Relayer's account. Fund it with the network's native gas token to cover transaction costs.

The Relayer recovers gas costs indirectly through the `baseFeeAmount` — this fee is designed to cover at minimum the cost of one `transferWithPermit` execution at expected gas prices.

Monitor Relayer balance and top up regularly. A Relayer with insufficient gas will cause transactions to fail at broadcast.

---

## Step 3 — Verify the Contract

Submit the source code and constructor arguments to the block explorer for public verification. This is not optional — a conformant deployment MUST have its contract verified. Auditors, regulators, and institutional partners will expect to be able to inspect the bytecode independently.

---

## Step 4 — Configure the broadcast-submitter

Provide the broadcast-submitter with:
- the Settlement Contract address
- the Relayer's private key (or HSM reference — recommended for production)
- the RPC endpoint for the target network

The broadcast-submitter will call `transferWithPermit` and `buyAcquiringPack` based on payloads delivered by the broadcast-service. It MUST NOT be the component that validates payloads — that responsibility belongs to the broadcast-service.

---

## Step 5 — Configure the transfer-history Service

Provide transfer-history with:
- the Settlement Contract address
- the list of supported token contract addresses
- the block confirmation depth before publishing settled events (RECOMMENDED: 12 blocks minimum for EVM mainnet)
- the core-checkout-engine endpoint to receive confirmed events

transfer-history MUST NOT publish a settlement event until the required confirmation depth is reached.

---

## Step 6 — Publish to the Basic Data Service

The basic-data-server MUST expose the following processor configuration to all clients:

```json
{
  "settlementContract": "0x...",
  "supportedTokens": [
    {
      "address": "0x...",
      "symbol": "USDC",
      "decimals": 6,
      "name": "USD Coin"
    }
  ],
  "walletGatewayUrl": "https://gateway.processor.example",
  "feeRecipient": "0x...",
  "chainId": 1
}
```

Wallets use this data to construct correct EIP-712 domains and fee calculations without hardcoding processor-specific values.

---

## Ongoing Operations

### Adjusting Fee Parameters

```javascript
// Set a new base fee
await settlementContract.setBaseFeeAmount(ethers.parseUnits("3.00", 6));

// Update max acquiring fee ceiling
await settlementContract.setMaxAcquiringFee(750);   // 7.50%

// Update registration price
await settlementContract.setAcquiringPrice(ethers.parseUnits("150", 6));
```

Fee changes take effect immediately for all future transactions. They do not affect in-flight payments where the payer has already signed.

### Adding a New Supported Token

To accept a new token for acquirer registration:

```javascript
await settlementContract.addAcquiringAllowedToken(newTokenAddress);
```

To add a token to the supported payment tokens list, update the basic-data-server. The Settlement Contract itself is token-agnostic for payment transfers — any ERC-2612-compliant token can be used, whether or not it is on the supported list. The list in the basic-data-server is advisory.

### Monitoring

Recommended events to monitor:
- `PermittedTransfer` — all successful payments; feed to transfer-history
- `AcquirerCreated` — new registrations; useful for onboarding notifications
- `CommissionGenerated` — acquirer fee activity
- `Withdrawal` — participant balance withdrawals

Monitor the Relayer account balance continuously. Consider automated alerting below a minimum threshold.

### Checking Balances

Use `getBalances` to inspect internal balances for fee recipients and acquirers:

```javascript
// Single token
const [processorBalance] = await settlementContract.getBalances(
  usdcAddress,
  [feeRecipientAddress]
);

// Multiple tokens, multiple participants
const balanceMatrix = await settlementContract.getBalances(
  [usdcAddress, eurcAddress],
  [feeRecipientAddress, acquirerAddress]
);
// balanceMatrix[0][0] = USDC balance of fee recipient
// balanceMatrix[0][1] = USDC balance of acquirer
// balanceMatrix[1][0] = EURC balance of fee recipient
// balanceMatrix[1][1] = EURC balance of acquirer
```

---

## Security Checklist

| Item | Required |
| ---- | -------- |
| Contract source verified on block explorer | MUST |
| Independent security audit completed | MUST |
| Relayer key in HSM or equivalent | RECOMMENDED |
| Admin key in multi-sig | RECOMMENDED |
| Relayer balance monitoring with alerts | RECOMMENDED |
| Confirmation depth ≥ 12 blocks for mainnet | RECOMMENDED |
| Basic Data Service address matches deployed contract | MUST |

---

## Related Documents

- [Contract Interface Reference](../reference/contract-interface.md) — all functions and events
- [Fee Model](../core-concepts/fee-model.md) — fee calculation model
- [Security Model](../specifications/ssf-spec-001.md#18-security-model) — administrative privilege boundaries
- [Formal Specification, Sections 10–13](../specifications/ssf-spec-001.md#10-settlement-contract--state-variables) — normative contract requirements
