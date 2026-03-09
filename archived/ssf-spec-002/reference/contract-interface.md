---
sidebar_position: 3
---

# Contract Interface Reference

Flat technical reference for all public and external functions, events, and state variables of the Settlement Contract. For normative definitions see [SSF-SPEC-002](../specifications/ssf-spec-002.md).

---

## State Variables

| Variable                 | Type                                              | Description                                                                                      |
| ------------------------ | ------------------------------------------------- | ------------------------------------------------------------------------------------------------ |
| `baseFeeAmount`          | `uint256`                                         | Minimum absolute fee (in token units) charged on every transfer.                                 |
| `maxAcquiringFee`        | `uint256`                                         | Maximum acquiring fee percentage an acquirer may configure. Enforced at registration and update. |
| `balances`               | `mapping(address => mapping(address => uint256))` | Internal balances: `token address => participant address => amount`.                             |
| `usedHashes`             | `mapping(bytes32 => bool)`                        | Registry of consumed Binding Signature digests. `true` = already processed, MUST revert.        |
| `acquirerWallets`        | `mapping(bytes16 => address)`                     | Maps Acquirer ID (UUID) to wallet address.                                                       |
| `acquiringFeePercent`    | `mapping(address => uint256)`                     | Maps acquirer wallet address to their configured fee percentage.                                 |
| `acquiringAllowedTokens` | `mapping(IERC20Permit => bool)`                   | Set of tokens accepted for acquirer registration payments.                                       |
| `acquiringPrice`         | `uint256`                                         | Token units required to register as a new acquirer.                                              |

---

## Functions

### `transferWithPermit`

Executes a stablecoin payment: verifies both signatures, collects tokens, distributes fees, credits principal to beneficiary.

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

| Parameter    | Type           | Description                                                                                        |
| ------------ | -------------- | -------------------------------------------------------------------------------------------------- |
| `token`      | `IERC20Permit` | ERC-2612-compliant stablecoin to transfer from.                                                    |
| `tokenOwner` | `address`      | Token holder whose permit authorises the transfer.                                                 |
| `amount`     | `uint256`      | Total amount inclusive of all fees. Must equal `value` in the signed permit.                       |
| `deadline`   | `uint256`      | Permit expiry timestamp. Forwarded to `token.permit()`.                                            |
| `v1, r1, s1` | signature      | **Permit Signature** — validated by the ERC-2612 token contract.                                  |
| `v2, r2, s2` | signature      | **Binding Signature** — validated by the Settlement Contract.                                     |
| `recipient`  | `address`      | Beneficiary address. Receives principal after fee deduction.                                       |
| `ref`        | `bytes32`      | 32-byte order reference (16-byte order ref + 16-byte acquirerId). Used for session reconciliation. |

**Reverts if:** Binding Signature digest in `usedHashes` · Binding Signature signer not authorised · `deadline <= block.timestamp` · `token.permit()` reverts · `tokenOwner` balance less than `amount`.

**Emits:** `PermittedTransfer` · `CommissionGenerated` (if acquirer fee applies).

---

### `buyAcquiringPack`

Registers a new acquirer by collecting a one-time fee. Payer and acquiring address may differ.

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

| Parameter    | Type           | Description                                                                            |
| ------------ | -------------- | -------------------------------------------------------------------------------------- |
| `token`      | `IERC20Permit` | Token used to pay registration fee. Must be in `acquiringAllowedTokens`.               |
| `payer`      | `address`      | Token holder paying the fee. May differ from `acquiring`.                              |
| `acquiring`  | `address`      | Wallet to be registered as acquirer. Must not already be registered.                   |
| `feePercent` | `uint256`      | Acquiring fee percentage this acquirer will charge. Must not exceed `maxAcquiringFee`. |
| `price`      | `uint256`      | Registration fee in token units. Must equal current `acquiringPrice`.                  |
| `deadline`   | `uint256`      | Permit expiry timestamp.                                                               |
| `v1, r1, s1` | signature      | **Permit Signature** — validated by the ERC-2612 token contract.                      |
| `v2, r2, s2` | signature      | **Binding Signature** — validated by the Settlement Contract.                         |

**Reverts if:** `token` not in `acquiringAllowedTokens` · `feePercent` exceeds `maxAcquiringFee` · `price` does not equal `acquiringPrice` · `acquiring` already registered · Binding Signature digest in `usedHashes` · Binding Signature signer not authorised · `token.permit()` reverts · `payer` balance less than `price`.

**Emits:** `AcquirerCreated`.

---

### `calculateFees`

Computes total fees to add on top of a principal amount.

```solidity
function calculateFees(
    uint256 amount,
    bytes16 acquirerId
) public view returns (uint256 totalFee)
```

| Parameter    | Type      | Description                                                                      |
| ------------ | --------- | -------------------------------------------------------------------------------- |
| `amount`     | `uint256` | Principal amount the beneficiary should receive, in token units.                 |
| `acquirerId` | `bytes16` | Acquirer ID for this payment. Pass Zero-UUID (`0x00...00`) if no acquirer.       |

**Returns:** `totalFee` — the fee amount to add to `amount` to get the correct `PermitParams.value`.

---

### `breakdownTransferAmount`

Decomposes a total amount (principal + fees) into its constituent parts.

```solidity
function breakdownTransferAmount(
    uint256 totalWithFees,
    bytes16 acquirerId
) public view returns (uint256 principalAmount, uint256 totalFees)
```

| Parameter       | Type      | Description                                                              |
| --------------- | --------- | ------------------------------------------------------------------------ |
| `totalWithFees` | `uint256` | Total token amount inclusive of fees. The value in the permit's `value`. |
| `acquirerId`    | `bytes16` | Acquirer ID for this payment. Pass Zero-UUID if no acquirer.             |

**Returns:**

| Return value      | Description                                                      |
| ----------------- | ---------------------------------------------------------------- |
| `principalAmount` | Amount credited to beneficiary after fee deduction.              |
| `totalFees`       | Total fees distributed to processor and acquirer (if applicable). |

---

### `getBalances` (single token)

Returns internal balances for multiple participants in one token.

```solidity
function getBalances(
    address token,
    address[] calldata users
) external view returns (uint256[] memory balances)
```

| Parameter | Type        | Description                                              |
| --------- | ----------- | -------------------------------------------------------- |
| `token`   | `address`   | Token contract address to query.                         |
| `users`   | `address[]` | Participant addresses to query.                          |

**Returns:** `uint256[]` parallel to `users` — internal balance of each address for `token`.

---

### `getBalances` (multi-token overload)

Returns internal balances for multiple participants across multiple tokens.

```solidity
function getBalances(
    address[] calldata tokens,
    address[] calldata users
) external view returns (uint256[][] memory balances)
```

| Parameter | Type        | Description                      |
| --------- | ----------- | -------------------------------- |
| `tokens`  | `address[]` | Token contract addresses.        |
| `users`   | `address[]` | Participant addresses.           |

**Returns:** `uint256[tokens.length][users.length]` — `balances[i][j]` is the internal balance of `users[j]` for `tokens[i]`.

---

### `getAcquiringWallet`

Resolves the wallet address for a given Acquirer ID.

```solidity
function getAcquiringWallet(
    bytes16 acquirerId
) public view returns (address)
```

| Parameter    | Type      | Description                  |
| ------------ | --------- | ---------------------------- |
| `acquirerId` | `bytes16` | UUID Acquirer ID to resolve. |

**Returns:** Registered wallet address, or the zero address if not registered.

---

## Events

### `PermittedTransfer`

Emitted on every successful `transferWithPermit` call. Primary event for checkout session reconciliation.

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

| Parameter       | Description                                                                                      |
| --------------- | ------------------------------------------------------------------------------------------------ |
| `domainSeparator` | EIP-712 domain separator of the ERC-2612 token.                                                |
| `token`         | ERC-2612 token contract address.                                                                 |
| `payer`         | Token owner whose permit authorised the transfer.                                                |
| `recipient`     | Beneficiary address that received the principal.                                                 |
| `value`         | Principal amount credited to recipient, in token units.                                          |
| `fee`           | Total fees deducted, in token units.                                                             |
| `orderReference` | 32-byte order reference. MUST match the `ref` submitted in the corresponding `TransferRequest`. |

---

### `AcquirerCreated`

Emitted when a new acquirer is registered via `buyAcquiringPack`.

```solidity
event AcquirerCreated(
    bytes16 indexed acquirerId,
    address indexed wallet,
    uint256 feePercent
)
```

| Parameter    | Description                                               |
| ------------ | --------------------------------------------------------- |
| `acquirerId` | The `bytes16` UUID assigned to the new acquirer.          |
| `wallet`     | The registered acquirer wallet address.                   |
| `feePercent` | The acquiring fee percentage the acquirer will charge.    |

---

### `CommissionGenerated`

Emitted when an acquiring fee is generated on a payment.

```solidity
event CommissionGenerated(
    bytes16 indexed acquirerId,
    uint256 amount
)
```

| Parameter    | Description                                         |
| ------------ | --------------------------------------------------- |
| `acquirerId` | The Acquirer ID that earned the commission.         |
| `amount`     | Commission amount credited to the acquirer, in token units. |

---

### `AcquiringFeeUpdated`

Emitted when an acquirer updates their fee percentage.

```solidity
event AcquiringFeeUpdated(
    address indexed acquiring,
    uint256 feePercent
)
```

| Parameter    | Description                     |
| ------------ | ------------------------------- |
| `acquiring`  | The acquirer's wallet address.  |
| `feePercent` | The new fee percentage value.   |

---

### `Withdrawal`

Emitted when a participant withdraws their accumulated internal balance.

```solidity
event Withdrawal(
    address indexed owner,
    address indexed beneficiary,
    uint256 amount
)
```

| Parameter     | Description                                       |
| ------------- | ------------------------------------------------- |
| `owner`       | The address whose internal balance was debited.   |
| `beneficiary` | The address that received the withdrawn funds.    |
| `amount`      | The amount withdrawn, in token units.             |
