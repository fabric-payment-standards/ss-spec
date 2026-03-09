---
document: SSF-SPEC-001/reference/contract-interface.md
spec: SSF-SPEC-001
version: 1.0.0
status: Draft
date: 2026-03-04
author: "Adalton Reis <reis@stablecoinstack.org>"
organization: Stablecoin Stack Foundation
license: Apache License 2.0
---

# Contract Interface Reference

Flat technical reference for all state variables, functions, and events of the Settlement Contract. For normative definitions see [SSF-SPEC-001, Sections 10–13](../specifications/ssf-spec-001.md#10-settlement-contract--state-variables).

---

## State Variables

| Variable | Type | Description |
| -------- | ---- | ----------- |
| `baseFeeAmount` | `uint256` | Minimum absolute fee (in token units) charged on every transfer. |
| `maxAcquiringFee` | `uint256` | Maximum acquiring fee percentage an acquirer may configure. Enforced at registration and update. |
| `balances` | `mapping(address => mapping(address => uint256))` | Internal balances: `token → participant → amount`. |
| `usedHashes` | `mapping(bytes32 => bool)` | Registry of consumed Binding Signature digests. `true` = already processed, MUST revert. |
| `acquirerWallets` | `mapping(bytes16 => address)` | Maps Acquirer ID (UUID) to wallet address. |
| `acquiringFeePercent` | `mapping(address => uint256)` | Maps acquirer wallet address to configured fee percentage. |
| `acquiringAllowedTokens` | `mapping(IERC20Permit => bool)` | Set of tokens accepted for acquirer registration payments. |
| `acquiringPrice` | `uint256` | Token units required to register as a new Acquirer. |

---

## Functions

### `transferWithPermit`

```solidity
function transferWithPermit(
    IERC20Permit token,
    address tokenOwner,
    uint256 amount,
    uint256 deadline,
    uint8 v1, bytes32 r1, bytes32 s1,
    uint8 v2, bytes32 r2, bytes32 s2,
    address recipient,
    bytes32 ref
) external
```

| Parameter | Type | Description |
| --------- | ---- | ----------- |
| `token` | `IERC20Permit` | ERC-2612-compliant stablecoin to transfer from. |
| `tokenOwner` | `address` | Token holder whose permit authorises the transfer. |
| `amount` | `uint256` | Total amount inclusive of fees. Must equal `value` in the signed permit. |
| `deadline` | `uint256` | Permit expiry timestamp. Forwarded to `token.permit()`. |
| `v1, r1, s1` | signature | **Permit Signature** — validated by the ERC-2612 token contract. |
| `v2, r2, s2` | signature | **Binding Signature** — validated by the Settlement Contract. |
| `recipient` | `address` | Beneficiary address. Receives principal after fee deduction. |
| `ref` | `bytes32` | 32-byte field: 16-byte Order Reference \|\| 16-byte Acquirer ID. |

**Reverts if:** digest of Binding Signature in `usedHashes` · Binding Signature signer not authorised · `deadline ≤ block.timestamp` · `token.permit()` reverts · `tokenOwner` balance less than `amount`.

**Emits:** `PermittedTransfer` · `CommissionGenerated` (if acquirer fee applies).

---

### `buyAcquiringPack`

```solidity
function buyAcquiringPack(
    IERC20Permit token,
    address payer,
    address acquiring,
    uint256 feePercent,
    uint256 price,
    uint256 deadline,
    uint8 v1, bytes32 r1, bytes32 s1,
    uint8 v2, bytes32 r2, bytes32 s2
) public
```

| Parameter | Type | Description |
| --------- | ---- | ----------- |
| `token` | `IERC20Permit` | Token for registration fee. Must be in `acquiringAllowedTokens`. |
| `payer` | `address` | Token holder paying the fee. May differ from `acquiring`. |
| `acquiring` | `address` | Wallet to register as Acquirer. Must not already be registered. |
| `feePercent` | `uint256` | Acquiring fee percentage. Must not exceed `maxAcquiringFee`. |
| `price` | `uint256` | Registration fee in token units. Must equal current `acquiringPrice`. |
| `deadline` | `uint256` | Permit expiry timestamp. |
| `v1, r1, s1` | signature | **Permit Signature** — validated by the ERC-2612 token contract. |
| `v2, r2, s2` | signature | **Binding Signature** — validated by the Settlement Contract. |

**Reverts if:** `token` not in `acquiringAllowedTokens` · `feePercent` exceeds `maxAcquiringFee` · `price` ≠ `acquiringPrice` · `acquiring` already registered · Binding Signature digest in `usedHashes` · Binding Signature signer not authorised · `token.permit()` reverts · `payer` balance less than `price`.

**Emits:** `AcquirerCreated`.

---

### `calculateFees`

```solidity
function calculateFees(
    uint256 amount,
    bytes16 acquirerId
) public view returns (uint256 totalFee)
```

| Parameter | Type | Description |
| --------- | ---- | ----------- |
| `amount` | `uint256` | Principal amount the beneficiary should receive, in token units. |
| `acquirerId` | `bytes16` | Acquirer ID. Pass Zero-UUID if no acquirer. |

**Returns:** `totalFee` — the fee to add to `amount` to get the correct `PermitParams.value`.

---

### `breakdownTransferAmount`

```solidity
function breakdownTransferAmount(
    uint256 totalWithFees,
    bytes16 acquirerId
) public view returns (uint256 principalAmount, uint256 totalFees)
```

| Parameter | Type | Description |
| --------- | ---- | ----------- |
| `totalWithFees` | `uint256` | Total inclusive of fees. The permit `value`. |
| `acquirerId` | `bytes16` | Acquirer ID. Pass Zero-UUID if no acquirer. |

**Returns:** `principalAmount` — credited to beneficiary; `totalFees` — distributed to processor and acquirer.

---

### `getBalances` (single token)

```solidity
function getBalances(
    address token,
    address[] calldata users
) external view returns (uint256[] memory)
```

Returns internal balances for multiple participants in one token. Response array is parallel to `users`.

---

### `getBalances` (multi-token overload)

```solidity
function getBalances(
    address[] calldata tokens,
    address[] calldata users
) external view returns (uint256[][] memory)
```

Returns `balances[i][j]` = internal balance of `users[j]` for `tokens[i]`.

---

### `getAcquiringWallet`

```solidity
function getAcquiringWallet(
    bytes16 acquirerId
) public view returns (address)
```

Returns the wallet address registered under `acquirerId`, or the zero address if not registered.

---

## Events

### `PermittedTransfer`

Primary event for session reconciliation. `orderReference` MUST match the `ref` submitted in `TransferRequest`.

```solidity
event PermittedTransfer(
    bytes32 indexed domainSeparator,
    address indexed token,
    address indexed payer,
    address recipient,
    uint256 value,
    uint256 fee,
    bytes32 orderReference
)
```

| Parameter | Description |
| --------- | ----------- |
| `domainSeparator` | EIP-712 domain separator of the ERC-2612 token. |
| `token` | Token contract address. |
| `payer` | Token owner whose permit authorised the transfer. |
| `recipient` | Beneficiary address that received the principal. |
| `value` | Principal amount credited to recipient, in token units. |
| `fee` | Total fees deducted, in token units. |
| `orderReference` | 32-byte field: 16-byte Order Reference \|\| 16-byte Acquirer ID. |

---

### `AcquirerCreated`

```solidity
event AcquirerCreated(
    bytes16 indexed acquirerId,
    address indexed wallet,
    uint256 feePercent
)
```

| Parameter | Description |
| --------- | ----------- |
| `acquirerId` | The `bytes16` UUID assigned to the new Acquirer. |
| `wallet` | The registered Acquirer wallet address. |
| `feePercent` | The acquiring fee percentage. |

---

### `CommissionGenerated`

```solidity
event CommissionGenerated(
    bytes16 indexed acquirerId,
    uint256 amount
)
```

| Parameter | Description |
| --------- | ----------- |
| `acquirerId` | The Acquirer ID that earned the commission. |
| `amount` | Commission credited to the Acquirer, in token units. |

---

### `AcquiringFeeUpdated`

```solidity
event AcquiringFeeUpdated(
    address indexed acquiring,
    uint256 feePercent
)
```

---

### `Withdrawal`

```solidity
event Withdrawal(
    address indexed owner,
    address indexed beneficiary,
    uint256 amount
)
```
