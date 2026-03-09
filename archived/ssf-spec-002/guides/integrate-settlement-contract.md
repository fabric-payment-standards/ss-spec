---
sidebar_position: 4
---

# Integrate the Settlement Contract

Guide for payment processor operators and off-chain service builders who need to interact with a deployed Settlement Contract.

---

## Reading Operational Parameters

Before processing any transactions, fetch the current operational parameters:

```javascript
const baseFeeAmount   = await settlementContract.baseFeeAmount();
const maxAcquiringFee = await settlementContract.maxAcquiringFee();
const acquiringPrice  = await settlementContract.acquiringPrice();
```

These values are set by the Administrator and may change over time. Fetch them fresh rather than caching indefinitely. A stale `acquiringPrice` will cause `buyAcquiringPack` calls to revert.

---

## Computing Correct Permit Amounts

### For payments

Use `calculateFees` to determine the total amount a payer must sign for:

```javascript
const totalFee   = await settlementContract.calculateFees(principal, acquirerId);
const permitValue = principal + totalFee;
```

Pass this `permitValue` as `PermitParams.value` and as the `amount` parameter in `transferWithPermit`.

### For acquirer registration

The permit value is simply `acquiringPrice`. No fee calculation is needed:

```javascript
const permitValue = await settlementContract.acquiringPrice();
```

### Decomposing a total amount

If you receive a total amount and need to display the split to a merchant:

```javascript
const [principalAmount, totalFees] = await settlementContract.breakdownTransferAmount(
  totalWithFees,
  acquirerId
);
```

---

## Verifying an Acquirer ID Before Broadcasting

Before broadcasting a `TransferRequest`, confirm that the `acquirerId` embedded in `ref` corresponds to a registered acquirer:

```javascript
// Extract acquirerId from the last 16 bytes of ref
const acquirerId = "0x" + ref.slice(34); // ref is "0x" + 64 hex chars

const wallet = await settlementContract.getAcquiringWallet(acquirerId);

if (wallet === "0x0000000000000000000000000000000000000000") {
  // Not registered — either it's the Zero-UUID (no acquirer) or an invalid ID
}
```

If `acquirerId` is the Zero-UUID, `getAcquiringWallet` returns the zero address, which is expected and valid. The acquiring fee component will be suppressed.

---

## Querying Internal Balances

### Single token, multiple participants

```javascript
const balances = await settlementContract.getBalances(
  tokenAddress,
  [address1, address2, address3]
);
// balances[0] is the internal balance of address1, etc.
```

### Multiple tokens, multiple participants

```javascript
const balances = await settlementContract.getBalances(
  [tokenA, tokenB],
  [address1, address2]
);
// balances[0][0] = address1's balance in tokenA
// balances[0][1] = address2's balance in tokenA
// balances[1][0] = address1's balance in tokenB
// balances[1][1] = address2's balance in tokenB
```

---

## Listening for Settlement Events

### Session reconciliation (`PermittedTransfer`)

The checkout engine resolves payment sessions by matching the `orderReference` in this event against the 16-byte order reference issued at session creation:

```javascript
settlementContract.on(
  "PermittedTransfer",
  (domainSeparator, token, payer, recipient, value, fee, orderReference, event) => {
    const orderRef  = "0x" + orderReference.slice(2, 34);  // first 16 bytes
    const acquirerId = "0x" + orderReference.slice(34);     // last 16 bytes
    // Match orderRef against open checkout sessions
  }
);
```

### Acquirer registration (`AcquirerCreated`)

```javascript
settlementContract.on(
  "AcquirerCreated",
  (acquirerId, wallet, feePercent, event) => {
    // Record new acquirer in your off-chain registry
  }
);
```

### Commission tracking (`CommissionGenerated`)

```javascript
settlementContract.on(
  "CommissionGenerated",
  (acquirerId, amount, event) => {
    // Update acquirer commission accounting
  }
);
```

---

## Replay Protection: Checking `usedHashes`

Before broadcasting, verify that the Binding Signature's digest has not already been processed:

```javascript
const digest = payWithPermitSig.hash;
const alreadyUsed = await settlementContract.usedHashes(digest);

if (alreadyUsed) {
  // Reject — this operation has already been processed
}
```

This check is also enforced on-chain, but performing it off-chain first avoids wasting gas on a guaranteed revert.

---

## Pre-Validating the Permit Signature Off-Chain

To avoid broadcasting a transaction that will revert due to an invalid Permit Signature, reconstruct the ERC-2612 digest using the token contract's domain separator and verify the signature locally before broadcasting:

```javascript
const tokenDomainSeparator = await tokenContract.DOMAIN_SEPARATOR();

// Reconstruct the permit digest
const permitStructHash = ethers.utils.keccak256(
  ethers.utils.defaultAbiCoder.encode(
    ["bytes32", "address", "address", "uint256", "uint256", "uint256"],
    [PERMIT_TYPEHASH, owner, spender, value, nonce, deadline]
  )
);

const permitDigest = ethers.utils.keccak256(
  ethers.utils.concat([
    "0x1901",
    tokenDomainSeparator,
    permitStructHash,
  ])
);

const recoveredSigner = ethers.utils.recoverAddress(permitDigest, { v, r, s });

if (recoveredSigner.toLowerCase() !== owner.toLowerCase()) {
  // Invalid permit signature — reject before broadcasting
}
```

---

## Security Reminders

- **Never rely solely on the `owner` field in the payload.** Always recover the signer from the Binding Signature independently and verify it matches `permitParams.owner`.
- **Record `usedHashes` before token movement** — follow the checks-effects-interactions pattern to prevent reentrancy-based replay.
- **The Administrator cannot move user funds.** If your deployment allows this, it is a configuration error.
- **The Zero-UUID must never be registerable** as an acquirer wallet. Verify this in your contract deployment.

---

## Related Documents

- [Settlement Contract](../overview/settlement-contract.md) — overview
- [Fee Model](../core-concepts/fee-model.md) — how fees are computed
- [Dual-Signature Pattern](../core-concepts/dual-signature-pattern.md) — signature verification logic
- [Contract Interface Reference](../reference/contract-interface.md) — complete function and event reference
- [SSF-SPEC-002](../specifications/ssf-spec-002.md) — normative contract specification
