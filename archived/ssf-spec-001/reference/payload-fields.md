---
sidebar_position: 2
---

# Payload Fields Reference

Flat field-level reference for all structures and payload types defined in SSF-SPEC-001. For normative rules see [SSF-SPEC-001](../specifications/ssf-spec-001.md).

---

## `ERC20RelayerSig`

Reused as `permitSig` and `payWithPermitSig` in all payload types.

| Field  | Type         | Format                   | Required | Constraints                                                                 |
| ------ | ------------ | ------------------------ | -------- | --------------------------------------------------------------------------- |
| `hash` | `bytes32`    | 66-char hex string       | REQUIRED | `0x` + 64 lowercase hex chars. The EIP-712 digest that was signed.          |
| `v`    | `uint8`      | integer                  | REQUIRED | `27` or `28`. Normalise `0`→`27`, `1`→`28`. Reject all other values.        |
| `r`    | `bytes32`    | 66-char hex string       | REQUIRED | `0x` + 64 lowercase hex chars. Leading zeros MUST be preserved.             |
| `s`    | `bytes32`    | 66-char hex string       | REQUIRED | `0x` + 64 lowercase hex chars. Leading zeros MUST be preserved.             |

---

## `PermitParams`

Nested inside `PayWithPermitParams` and `BuyAcquiringPackPermitParams`.

| Field      | Type      | Format                        | Required | Constraints                                                                                      |
| ---------- | --------- | ----------------------------- | -------- | ------------------------------------------------------------------------------------------------ |
| `owner`    | `address` | 42-char hex string            | REQUIRED | `0x` + 40 lowercase hex chars. Must match the signer recovered from `payWithPermitSig`.          |
| `spender`  | `address` | 42-char hex string            | REQUIRED | `0x` + 40 lowercase hex chars. Typically the Settlement Contract address.                        |
| `value`    | `uint256` | decimal or hex string         | REQUIRED | Non-negative integer in `uint256` range. Total amount inclusive of all fees.                     |
| `nonce`    | `uint256` | decimal or hex string         | REQUIRED | Must match current on-chain ERC-2612 nonce of `owner` on the token contract.                    |
| `deadline` | `uint256` | Unix timestamp (seconds)      | REQUIRED | Must be strictly greater than processor clock at submission time.                                |

---

## `PayWithPermitParams`

| Field          | Type           | Format                   | Required | Constraints                                                                                                                      |
| -------------- | -------------- | ------------------------ | -------- | -------------------------------------------------------------------------------------------------------------------------------- |
| `token`        | `address`      | 42-char hex string       | REQUIRED | ERC-2612-compliant stablecoin contract address.                                                                                  |
| `beneficiary`  | `address`      | 42-char hex string       | REQUIRED | Merchant's receiving address.                                                                                                    |
| `ref`          | `bytes32`      | 66-char hex string       | REQUIRED | 16-byte order reference concatenated with 16-byte `acquirerId`. Missing parts MUST be zero-padded. Must always be present.      |
| `permitParams` | `PermitParams` | nested object            | REQUIRED | As defined above. `owner` MUST match recovered signer of `payWithPermitSig`.                                                    |

### `ref` field composition

```
ref (32 bytes) = orderReference (16 bytes) || acquirerId (16 bytes)
```

When either part "../../../ssf-spec-001/reference"is absent, fill with zeros:

| Scenario                        | `ref` value                                          |
| ------------------------------- | ---------------------------------------------------- |
| Both present                    | `orderReference` + `acquirerId`                      |
| No acquirer                     | `orderReference` + `0x00000000000000000000000000000000` |
| No order reference              | `0x00000000000000000000000000000000` + `acquirerId`  |
| Neither                         | `0x0000000000000000000000000000000000000000000000000000000000000000` |

---

## `BuyAcquiringPackPermitParams`

| Field          | Type           | Format                   | Required | Constraints                                                                                           |
| -------------- | -------------- | ------------------------ | -------- | ----------------------------------------------------------------------------------------------------- |
| `token`        | `address`      | 42-char hex string       | REQUIRED | Must be present in `acquiringAllowedTokens` on the Settlement Contract.                               |
| `feeValue`     | `uint256`      | decimal or hex string    | REQUIRED | Must equal `permitParams.value`. The registration fee in token's smallest unit.                       |
| `acquiring`    | `address`      | 42-char hex string       | REQUIRED | Wallet address to be registered as acquirer. Must not already be registered.                          |
| `permitParams` | `PermitParams` | nested object            | REQUIRED | `value` MUST equal `feeValue`. `owner` MUST match recovered signer of `payWithPermitSig`.             |

---

## `TransferRequest` (full payload)

| Field                 | Type                    | Required | Description                                                                                                      |
| --------------------- | ----------------------- | -------- | ---------------------------------------------------------------------------------------------------------------- |
| `payWithPermitParams` | `PayWithPermitParams`   | REQUIRED | Payment parameters as defined above.                                                                             |
| `payWithPermitSig`    | `ERC20RelayerSig`       | REQUIRED | Binding Signature over `payWithPermitParams`. Recovered signer MUST equal `permitParams.owner`.                 |
| `permitSig`           | `ERC20RelayerSig`       | REQUIRED | Permit Signature over ERC-2612 `Permit` typed-data. Forwarded verbatim to the Settlement Contract.              |
| `payloadId`           | `string`                | REQUIRED | Client-generated UUID or processor-issued opaque ID. Must correspond to a known, unprocessed checkout session.  |

---

## `BuyAcquiringPackRequest` (full payload)

| Field                    | Type                            | Required | Description                                                                                                               |
| ------------------------ | ------------------------------- | -------- | ------------------------------------------------------------------------------------------------------------------------- |
| `buyAcquiringPackParams` | `BuyAcquiringPackPermitParams`  | REQUIRED | Acquirer registration parameters as defined above.                                                                        |
| `payWithPermitSig`       | `ERC20RelayerSig`               | REQUIRED | Binding Signature over `buyAcquiringPackParams`. Recovered signer MUST equal `permitParams.owner`.                       |
| `permitSig`              | `ERC20RelayerSig`               | REQUIRED | Permit Signature. Signed `value` MUST equal `buyAcquiringPackParams.feeValue`. Forwarded verbatim to the contract.       |

---

## Encoding Rules (all fields)

| Rule | Detail |
| ---- | ------ |
| Hex prefix | All hex strings MUST begin with `0x` |
| Case | Lowercase hex digits only (`a`–`f`). Uppercase MUST be normalised before processing. |
| Address length | Exactly 42 characters (`0x` + 40 hex digits) |
| `bytes32` length | Exactly 66 characters (`0x` + 64 hex digits) |
| Leading zeros | MUST be preserved. A `bytes32` is always 66 chars regardless of value. |
| `uint256` range | 0 to 2²⁵⁶ − 1 |
| `v` normalisation | `0` → `27`, `1` → `28`. Values outside `{0, 1, 27, 28}` MUST be rejected. |

---

## Error Categories

| Code                  | Trigger                                                                             |
| --------------------- | ----------------------------------------------------------------------------------- |
| `STRUCTURAL_ERROR`    | Missing field, wrong type, encoding violation (length, prefix, non-hex character)  |
| `SEMANTIC_ERROR`      | Expired deadline, nonce mismatch, unknown `payloadId`, `feeValue` ≠ `permitParams.value` |
| `CRYPTOGRAPHIC_ERROR` | Signature recovery failure, signer ≠ owner, hash field mismatch                    |
| `BROADCAST_ERROR`     | On-chain transaction revert or submission failure after successful validation        |
