# Unit Mismatch in Fee Validation Prevents Protocol Fee Updates

* **Severity**: Medium
* **Source**: [Code4rena](https://github.com/code-423n4/2024-08-phi-findings/issues/251)
* **Affected Contract**: [PhiFactory.sol](https://github.com/code-423n4/2024-08-phi/blob/8c0985f7a10b231f916a51af5d506dd6b0c54120/src/PhiFactory.sol#L422-L434)
* **Vulnerability Type**: Unit Mismatch / Incorrect Input Validation / Configuration Bug

## Summary

The protocol allows the owner to configure two protocol-wide fees:

* **Mint Protocol Fee** (charged whenever a Cred NFT is minted)
* **Art Creation Fee** (charged whenever a new art is created)

However, the setter functions incorrectly assume these fees are small values capped at **10,000**, while the rest of the protocol treats them as **raw wei amounts**.

As a result, the documented fee (`0.00005 ether`) exceeds the hardcoded limit by several orders of magnitude, making it impossible for the owner to configure the intended fee or adjust it in the future.

Although users can still mint if the fee was initialized correctly during deployment, the protocol permanently loses the ability to update its fee configuration through the provided admin functions.

## A Better Explanation (With Simplified Example)

### Intended Behavior

The protocol documentation specifies:

```text
Mint Fee = 0.00005 ETH
Art Creation Fee = 0.00005 ETH
```

The owner should be able to change these fees whenever necessary.

For example:

```text
Today:
Mint Fee = 0.00005 ETH

Later:
Mint Fee = 0.00003 ETH

Even Later:
Mint Fee = 0.00002 ETH
```

The setter functions are supposed to make these updates possible.

### How the Protocol Actually Uses These Fees

Whenever someone mints NFTs, the protocol transfers:

```solidity
protocolFeeDestination.safeTransferETH(
    mintProtocolFee * quantity
);
```

Suppose:

```text
mintProtocolFee = 0.00005 ETH
quantity = 3
```

The protocol charges:

```text
0.00005 × 3
=
0.00015 ETH
```

Notice that **no percentage calculation is performed**.

There is no:

```solidity
fee / 10000
```

or

```solidity
price * fee / 10000
```

Instead, `mintProtocolFee` is directly interpreted as a **wei amount**.

The art creation fee behaves exactly the same way:

```solidity
protocolFeeDestination.safeTransferETH(artFee);
```

Again, `artFee` is treated as a direct ETH amount.

### What Actually Happens (Bug)

The setter functions enforce:

```solidity
if (protocolFee_ > 10000)
    revert();
```

and

```solidity
if (artCreateFee_ > 10000)
    revert();
```

The problem is that **10,000 means 10,000 wei**, not 10,000 basis points or some percentage.

However, the documented fee is:

```text
0.00005 ETH
```

which equals:

```text
50,000,000,000,000 wei
```

Since

```text
50,000,000,000,000 > 10,000
```

the setter always reverts.

Therefore, the protocol refuses to accept the very fee that its own documentation specifies.

### Why This Happens

This is a classic **unit mismatch**.

The developer appears to have written the validation as though the fee were stored as a **basis points (bps)** value.

In many Solidity projects:

```text
10000 = 100%
300 = 3%
```

so limiting values to `10000` makes sense.

However, the rest of the protocol does **not** treat the fee as basis points.

Instead, it stores and transfers the value directly as **wei**.

The setter assumes one unit.

The payment logic assumes another.

This disagreement creates the bug.

### Concrete Walkthrough (Protocol Owner)

Imagine the protocol has already been deployed.

The owner decides to update the mint fee.

He calls:

```solidity
setProtocolFee(0.00005 ether);
```

Solidity converts this into:

```text
50,000,000,000,000 wei
```

The setter evaluates:

```text
50,000,000,000,000 > 10,000
```

which is true.

The transaction immediately reverts.

The owner tries:

```text
0.00003 ETH
```

Still reverts.

He tries:

```text
0.00001 ETH
```

Still reverts.

In practice, **every realistic fee exceeds 10,000 wei**, so the protocol owner can no longer update either fee through the intended admin functions.

> **Analogy:** Imagine a parking garage where the entrance sign says "Maximum vehicle height: 10 centimeters" instead of "10 meters." Every normal car is rejected—not because the parking system is broken, but because the limit was specified in the wrong unit.

## Vulnerable Code Reference

### 1) Incorrect validation assumes fees are capped at `10,000`

```solidity
function setProtocolFee(uint256 protocolFee_) external onlyOwner {
    if (protocolFee_ > 10_000)
        revert ProtocolFeeTooHigh();

    mintProtocolFee = protocolFee_;
}
```

```solidity
function setArtCreatFee(uint256 artCreateFee_) external onlyOwner {
    if (artCreateFee_ > 10_000)
        revert ArtCreatFeeTooHigh();

    artCreateFee = artCreateFee_;
}
```

---

### 2) Fee is later treated as a raw wei amount

```solidity
protocolFeeDestination.safeTransferETH(
    mintProtocolFee * quantity_
);
```

No percentage calculation is performed.

---

### 3) Art creation fee is also transferred directly

```solidity
uint256 artFee = phiFactoryContract.artCreateFee();

protocolFeeDestination.safeTransferETH(artFee);
```

Again, the fee is interpreted as an absolute wei amount.

## Recommended Mitigation

### 1. Validate against the intended ETH-denominated maximum

Instead of using an arbitrary `10_000` limit:

```solidity
uint256 constant MAX_PROTOCOL_FEE = 0.00005 ether;
uint256 constant MAX_ART_CREATE_FEE = 0.00005 ether;
```

Then:

```solidity
if (protocolFee_ > MAX_PROTOCOL_FEE)
    revert ProtocolFeeTooHigh();
```

and

```solidity
if (artCreateFee_ > MAX_ART_CREATE_FEE)
    revert ArtCreatFeeTooHigh();
```

---

### 2. Keep units consistent throughout the codebase

If fees are stored as **wei**, then every validation, setter, getter, and payment function should consistently treat them as wei.

If fees are intended to be percentages, then the payment logic should perform percentage calculations instead.

---

### 3. Clearly document fee units

Variables like:

```solidity
mintProtocolFee
artCreateFee
```

should explicitly document whether they represent:

* wei
* ether
* basis points
* percentages

This greatly reduces the likelihood of future unit mismatches.

---

### 4. Add unit tests for configuration values

Tests should verify that documented configuration values can actually be set.

For example:

* Setting `0.00005 ether` succeeds.
* Setting values above the maximum reverts.
* Setting smaller valid values succeeds.

This ensures configuration APIs remain consistent with protocol documentation.

## Pattern Recognition Notes

* **Unit Mismatch**: One part of the protocol interprets a variable in one unit (wei), while another interprets it in another (basis points). Always verify that every read and write of a variable agrees on its units.
* **Configuration Validation Bugs**: Admin setters are part of the protocol's public interface. Incorrect validation can permanently disable intended governance or protocol operations even if user-facing functionality still works.
* **Magic Numbers**: Hardcoded values like `10000` should raise immediate suspicion. Ask yourself what unit they represent and whether that matches how the variable is used elsewhere.
* **Documentation vs Implementation**: Whenever documentation specifies values like `0.00005 ETH`, verify that the implementation actually allows those values to be configured.
* **End-to-End Unit Consistency**: Follow configuration variables through their complete lifecycle—from setter → storage → getter → usage—to ensure they retain the same meaning everywhere.

## Quick Recall (TL;DR)

* **Bug**: Fee setter limits values to `10,000 wei`, while the protocol stores and transfers fees as raw wei amounts like `0.00005 ether`.
* **Impact**: The documented fee exceeds the allowed limit, preventing the owner from updating mint and art creation fees through the provided admin functions.
* **Root Cause**: Different parts of the protocol interpret the same fee variable using different units (basis points vs wei).
* **Fix**: Validate against ETH-denominated limits (e.g., `0.00005 ether`) or consistently use a single unit throughout the protocol.
