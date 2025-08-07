# Incorrect Share-to-Fraction Calculation Due to Inconsistent Rounding in MagnetarHelper

* **Severity**: Medium
* **Source**: [Code4rena](https://github.com/code-423n4/2024-02-tapioca-findings/issues/94)
* **Affected Contract**: [MagnetarHelper.sol](https://github.com/Tapioca-DAO/tapioca-periph/blob/2ddbcb1cde03b548e13421b2dba66435d2ac8eb5/contracts/Magnetar/MagnetarHelper.sol)
* **Vulnerability Type**: Business Logic / Math Error

## Original Bug Description

>## Lines of code
>
>[https://github.com/Tapioca-DAO/tapioca-periph/blob/2ddbcb1cde03b548e13421b2dba66435d2ac8eb5/contracts/Magnetar/MagnetarHelper.sol#L208-L219](https://github.com/Tapioca-DAO/tapioca-periph/blob/2ddbcb1cde03b548e13421b2dba66435d2ac8eb5/contracts/Magnetar/MagnetarHelper.sol#L208-L219)
>
>## Vulnerability details
>
>## Impact
>
>`MagnetarHelper.getFractionForAmount` is used to determine the `fraction` (shares) that will be received when an `amount` is deposited.
>
>The code needs to compute the totalShares and then determine what the fraction is going to be.
>
>The logic is used in `Singularity._removeAsset` and since it's tied to a withdrwal, the rounding will be `down`
>
>However, in `MagnetarHelper` the round is `up`
>
>[https://github.com/Tapioca-DAO/tapioca-periph/blob/2ddbcb1cde03b548e13421b2dba66435d2ac8eb5/contracts/Magnetar/MagnetarHelper.sol#L208-L219](https://github.com/Tapioca-DAO/tapioca-periph/blob/2ddbcb1cde03b548e13421b2dba66435d2ac8eb5/contracts/Magnetar/MagnetarHelper.sol#L208-L219)
>
>```solidity
>    function getFractionForAmount(ISingularity singularity, uint256 amount) external view returns (uint256 fraction) {
>        (uint128 totalAssetShare, uint128 totalAssetBase) = singularity.totalAsset();
>        (uint128 totalBorrowElastic,) = singularity.totalBorrow();
>        uint256 assetId = singularity.assetId();
>
>        IYieldBox yieldBox = IYieldBox(singularity.yieldBox());
>
>        uint256 share = yieldBox.toShare(assetId, amount, false);
>        uint256 allShare = totalAssetShare + yieldBox.toShare(assetId, totalBorrowElastic, true); /// @audit Round UP for Debt
>
>        fraction = allShare == 0 ? share : (share * totalAssetBase) / allShare;
>    }
>```
>
>See the actual implementation in `Singularity`
>
>[https://github.com/Tapioca-DAO/Tapioca-bar/blob/c2031ac2e2667ac8f9ac48eaedae3dd52abef559/contracts/markets/singularity/SGLCommon.sol#L199-L216](https://github.com/Tapioca-DAO/Tapioca-bar/blob/c2031ac2e2667ac8f9ac48eaedae3dd52abef559/contracts/markets/singularity/SGLCommon.sol#L199-L216)
>
>```solidity
>    function _removeAsset(address from, address to, uint256 fraction) internal returns (uint256 share) {
>        if (totalAsset.base == 0) {
>            return 0;
>        }
>        Rebase memory _totalAsset = totalAsset;
>        uint256 allShare = _totalAsset.elastic + yieldBox.toShare(assetId, totalBorrow.elastic, false);
>        share = (fraction * allShare) / _totalAsset.base;
>    }
>```
>
>This will cause issues when calculating how much fraction to withdraw based on the amount required by the withdrawer
>
>## Mitigation
>
>Change the rounding direction
>
>```solidity
>        uint256 allShare = totalAssetShare + yieldBox.toShare(assetId, totalBorrowElastic, down);
>```
>
>## Assessed type
>
>Math

## Summary

`MagnetarHelper.getFractionForAmount()` incorrectly uses **rounding up** when converting borrowed asset amounts into "shares" while computing user fractions. However, the core Singularity contract (`_removeAsset`) rounds **down** during the actual share-to-fraction redemption.

This inconsistency creates an off-by-one bug that can cause users to underestimate the fraction they need to withdraw specific amounts, which may result in transaction reverts or misalignments in expected behavior.

## A Better Explanation (With Simplified Example)

To understand this bug, let's break it down step by step, using simpler terms.

### What's Going On?

In Tapioca, the protocol needs to figure out **how much of your "share" of the system** (called a "fraction") it should take when you want to **withdraw a specific dollar amount** of tokens — like \$15 worth of USDC.

This is done using a helper function in `MagnetarHelper`.

But here's the catch: the helper function rounds numbers **up**, while the actual withdrawal logic later (inside the core contract) rounds **down**.

That small difference in rounding can cause big problems when the two don't match — your withdrawal might fail or you may not get what you expected.

### Understanding the Terms

Let's decode the confusing variable names first:

| Term                   | What It Means                                                         |
| ---------------------- | --------------------------------------------------------------------- |
| `totalAsset.base`      | The total amount of "fractions" in the system (like slices of a pie). |
| `totalAsset.elastic`   | The total value (in tokens) backing those fractions.                  |
| `totalBorrow.elastic`  | The total borrowed tokens (i.e., debt in the system).                 |
| `share`                | A representation of value inside a wrapper called YieldBox.           |
| `toShare()`            | A function that converts dollar/token amounts into "shares".          |
| `roundUp` (true/false) | Whether to round up or down during conversion.                        |

So essentially, when you try to withdraw \$15:

* The protocol converts that \$15 into **"shares"**
* Then it converts those shares into a **fraction**, which is what the system actually tracks
* Later, it reverses the process to actually give you tokens

If the **fraction** was calculated based on a **rounded-up** number of shares, but the withdrawal is executed based on **rounded-down** math, the actual value retrieved can be **less than expected or can even fail**.

### Step-by-Step Example

Let's say:

* The system has 1000 fractions (`totalAsset.base`)
* Those fractions are backed by 1200 tokens (`totalAsset.elastic`)
* There are 300 borrowed tokens (`totalBorrow.elastic`)
* The asset is USDC

You want to **withdraw \$15 worth of USDC**

Now the helper (`MagnetarHelper`) tries to figure out **what fraction you need** to withdraw \$15.

To do that, it does:

```solidity
yieldBox.toShare(assetId, 15, false)  → returns 15 (round down)
yieldBox.toShare(assetId, 300, true)  → returns 301 (round up)
```

> So MagnetarHelper thinks the system has:
>
> ```solidity
> allShares = totalAssetShare + 301
> ```

But the **real withdrawal function** (in Singularity) does this:

```solidity
yieldBox.toShare(assetId, 300, false)  → returns 300 (round down)
```

> So Singularity thinks the system has:
>
> ```solidity
> allShares = totalAssetShare + 300
> ```

That 1-share mismatch leads to a **wrong calculation** of how many fractions you need. When the user submits the withdrawal, the math won't line up — it could result in **withdrawal failure** or **incorrect asset amounts**.

## Vulnerable Code Reference

```solidity
// MagnetarHelper.sol
function getFractionForAmount(ISingularity singularity, uint256 amount) external view returns (uint256 fraction) {
    (uint128 totalAssetShare, uint128 totalAssetBase) = singularity.totalAsset();
    (uint128 totalBorrowElastic,) = singularity.totalBorrow();
    uint256 assetId = singularity.assetId();

    IYieldBox yieldBox = IYieldBox(singularity.yieldBox());

    uint256 share = yieldBox.toShare(assetId, amount, false);
    uint256 allShare = totalAssetShare + yieldBox.toShare(assetId, totalBorrowElastic, true);

    fraction = allShare == 0 ? share : (share * totalAssetBase) / allShare;
}
```

```solidity
// Singularity (actual withdrawal logic)
function _removeAsset(address from, address to, uint256 fraction) internal returns (uint256 share) {
    if (totalAsset.base == 0) {
        return 0;
    }
    Rebase memory _totalAsset = totalAsset;
    uint256 allShare = _totalAsset.elastic + yieldBox.toShare(assetId, totalBorrow.elastic, false); // Rounds DOWN
    share = (fraction * allShare) / _totalAsset.base;
}
```

## Recommended Mitigation

1. **Align the rounding direction in `MagnetarHelper`**:

   * Replace this:

     ```solidity
     yieldBox.toShare(assetId, totalBorrowElastic, true)
     ```

   * With this:

     ```solidity
     yieldBox.toShare(assetId, totalBorrowElastic, false)
     ```

2. **Add Integration Tests**:

   * Test scenarios where users withdraw exact amounts (e.g., \$15 USDC)
   * Ensure the calculated `fraction` truly redeems that amount

3. **Document Consistency**:

   * Make clear throughout the codebase what rounding direction is expected when calculating fractions or shares

## Pattern Recognition Notes

This vulnerability highlights a class of **precision and rounding inconsistencies** that often arise when converting between *different representations* of value in DeFi protocols. To identify similar issues in other codebases, watch for the following patterns:

### 1. **Inconsistent Rounding Direction Across Related Functions**

* Many DeFi systems convert between tokens, shares, and fractions.
* These conversions often involve rounding. If **one part rounds up** and another **rounds down**, it can lead to **off-by-one** or **value misalignment bugs**.
* 🔍 **Red flag**: Same conversion function (e.g., `toShare`, `toAmount`) is called with different rounding parameters (`true` vs `false`) in related logic (e.g., deposit vs withdraw).

### 2. **Silent Overflows or Underflows via Integer Division**

* Solidity truncates towards zero when dividing integers — this can silently cut off small values and cause miscalculations.
* If a protocol uses formulas like `fraction = (amount * total) / base`, and `base` is slightly off (due to rounding), the result may deviate significantly, especially at small scales.
* 🔍 **Red flag**: Look for division-based formulas that don't normalize precision first, or where rounding behavior isn't clearly documented.

### 3. **Rebase Structures Require Extra Caution**

* Structures like `Rebase` (common in protocols like BentoBox, Tapioca, or YieldBox) involve both a `.base` and `.elastic` value.
* These two fields must stay in sync, or all share/fraction math becomes unreliable.
* 🔍 **Red flag**: When you see rebase math (`elastic + borrowedShares` / `base`), verify whether the borrow side is rounded differently than the asset side.

### 4. **Mixed Representations: Amounts vs Shares vs Fractions**

* Bugs often occur when the code mixes units without clean conversions. Example units:

  * `amount`: Raw token value (e.g., 15 USDC)
  * `share`: Internal representation inside YieldBox/BentoBox
  * `fraction`: Portion of protocol-level rebase (e.g., stake in the pool)
* 🔍 **Red flag**: If a function compares or operates on different unit types without converting them to a common basis, it's a potential bug.

### 5. **Assumptions That "Rounding Won't Matter"**

* Developers often assume small rounding errors are negligible. But when these errors control access to funds (like withdrawals), they **absolutely matter**.
* Rounding can mean the difference between allowing a withdrawal or rejecting it, or over-crediting someone by 1 unit repeatedly.
* 🔍 **Red flag**: Critical logic paths (e.g., deposits, withdrawals, fee collection) that involve rounding should be reviewed rigorously, even if the difference seems small.

### 6. **Cross-Module Dependencies That Don't Align**

* Here, one module (e.g., `MagnetarHelper`) uses an **external approximation** to estimate something that another module (e.g., `Singularity`) handles authoritatively.
* This causes inconsistencies when logic in helper contracts tries to replicate or guess internal state behavior.
* 🔍 **Red flag**: If helper or utility contracts mimic calculations found in core contracts, verify the implementation matches exactly — including rounding rules.
