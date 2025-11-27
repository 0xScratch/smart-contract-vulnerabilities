# Protocol Reserve Leakage via Unrestricted Borrowing in LToken Vault

* **Severity**: Medium
* **Source**: [Sherlock Audits](https://github.com/sherlock-audit/2022-08-sentiment-judging/blob/main/122-M/1-report.md)
* **Affected Contract**: `LToken.sol` (Link in the report doesn't work)
* **Vulnerability Type**: Broken Invariant / Reserve Leakage / Economic Integrity Failure

## Summary

The LToken vault maintains a **protocol reserve** — a portion of assets set aside to compensate the protocol or act as a liquidity backstop. These reserves are **deliberately excluded** from LP withdrawals using:

```solidity
totalAssets() = balance + borrows - reserves
```

However, the lending flow (`lendTo`) does **not** enforce reserve preservation. As long as a borrower provides sufficient collateral, they can borrow **all assets held by the LToken**, including the protocol reserve itself.

This breaks the core invariant that reserves must remain untouched, making it possible for the vault to end up with **zero liquidity** and **no reserve buffer**, leaving LPs and the protocol exposed to insolvency risk.

## A Better Explanation (With Simplified Example)

### Intended Behavior

The system's architecture treats the protocol reserve as **non-withdrawable and non-lendable**:

1. **LP withdrawals** use `totalAssets()` where reserves are *subtracted*.
   This ensures LPs cannot withdraw reserve funds.
2. **Protocol safety** depends on this reserve staying intact during lending.
   It acts as insurance in case borrowers fail to repay.

### What Actually Happens (Bug)

While withdrawals respect the reserve, **borrowing does not**:

```solidity
asset.safeTransfer(account, amt);  // sends underlying tokens to borrower
```

There is **no check** that lending must leave `asset.balanceOf(this) ≥ reserve`.

Thus:

* If the vault contains `LP deposits + protocol reserve`,
* A borrower can request an amount **equal to the total vault balance**,
* And the contract will happily send it as long as the borrower has collateral.

This means the protocol reserve — which should be locked — becomes freely lendable.

### Why This Matters

Once the reserve is lent out:

* The vault may become **fully drained**.
* There is **no liquidity** left to buffer losses.
* A borrower default becomes **catastrophic** for LPs and the protocol.
* The protocol loses its ability to backstop shortfalls.
* LP withdrawals may revert due to lack of funds.

A feature intended as an insurance layer turns into a **borrowable, depletive balance**, eliminating a core safety invariant.

## Concrete Walkthrough (Alice & Mallory)

*Setup:*
The LToken vault holds:

* **900 ETH** from depositors
* **100 ETH** as protocol reserve
* **Total balance = 1000 ETH**

### Mallory borrows

Mallory has enough collateral to maintain a "healthy" account. He asks to borrow:

```text
borrow 1000 ETH
```

The `lendTo` function:

```solidity
convertAssetToBorrowShares(1000);
borrows += 1000;
asset.safeTransfer(mallory, 1000);
```

There is **no reserve check**, so the borrow succeeds.

Vault balance becomes:

```text
0 ETH remaining
reserve = functionally 0 (it has already been lent out)
```

### Alice wants to withdraw

Alice tries to redeem her LP tokens. The logic believes reserves are intact, but the vault is **empty**, so her withdrawal will revert or underflow unless Paused/Emergency measures kick in.

## Vulnerable Code Reference

**1) `totalAssets()` excludes reserves — correctly — for LP withdrawals:**

```solidity
function totalAssets() public view override returns (uint) {
    return asset.balanceOf(address(this)) + getBorrows() - getReserves();
}
```

**2) `lendTo()` does not preserve reserves:**

```solidity
function lendTo(address account, uint amt) external {
    updateState();
    ...
    asset.safeTransfer(account, amt);  // drains vault without reserve protection
}
```

There is no:

```solidity
require(asset.balanceOf(address(this)) >= getReserves());
```

**3) Reserve logic exists, but is not enforced during borrowing:**

```solidity
function getReserves() public view returns (uint) {
    return reserves + borrows.mulWadUp(getRateFactor()).mulWadUp(reserveFactor);
}
```

Reserves are calculated but **not protected**.

## Recommended Mitigation

### 1. Enforce reserve preservation inside `lendTo`

```solidity
require(
    asset.balanceOf(address(this)) - amt >= getReserves(),
    "Insufficient liquidity: reserves must remain untouched"
);
```

This ensures lending **never dips into** required reserve levels.

### 2. Pause lending when vault liquidity is below reserve

If the remaining free liquidity (non-reserve) is zero or too low, lending should be disabled entirely.

### 3. Add invariant tests

Unit tests should assert:

* `lendTo()` cannot change `reserve ≥ getReserves()`
* Borrowers cannot drain vault balance below reserve level

## Pattern Recognition Notes

* **Unprotected Invariant**: A "special" balance (reserves) is only partially enforced.
* **Asymmetric Safety**: Withdraw path protects reserves; borrow path ignores them.
* **Economic Integrity Failure**: Insurance funds become borrowable, nullifying risk buffers.
* **Collateral-Based Blind Spot**: Relying only on health checks without liquidity checks can lead to vault drainage.
* **Missing Strategic Require**: A single missing reserve threshold check enables total fund depletion.

### Quick Recall (TL;DR)

* **Bug**: Borrowers can drain *protocol reserve* because `lendTo()` never checks that reserves must remain untouched.
* **Impact**: Vault can reach **0 liquidity**, **0 reserve**, and become insolvent.
* **Fix**: Add reserve-protection require; restrict lending when free liquidity is depleted.
