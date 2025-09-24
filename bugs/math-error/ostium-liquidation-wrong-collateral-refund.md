# Wrong Collateral Refund in Liquidation (`liqPrice == priceAfterImpact`)

* **Severity**: Medium (High Impact, Low Likelihood)
* **Source**: [Pashov Audit Group Report](https://github.com/pashov/audits/blob/master/team/md/Ostium-security-review_2025-04-06.md#m-01-wrong-collateral-refund-in-liquidation-when-liqprice--priceafterimpact)
* **Affected Contract**: Unknown (likely liquidation / trade settlement module)
* **Vulnerability Type**: Liquidation Logic Error / Rounding Edge-Case / Legacy Refund Path

## Summary

During liquidation, if the oracle price and the execution price after market impact are **exactly equal** (`liqPrice == priceAfterImpact`), inconsistent math inside `getTradeValuePure()` causes the system to incorrectly compute a positive refund amount (`usdcSentToTrader`).

Because legacy refund logic still executes, the liquidated trader can **wrongly receive back collateral**, approximately equal to the liquidation fee, even though the position should have been fully closed with **zero refund**.

## A Better Explanation (With Simplified Example)

### Intended Behavior

1. Trader opens a leveraged position with \$100 collateral.
2. Price drops to the liquidation threshold.
3. On liquidation:

   * \$5 goes to the liquidator (fee).
   * \$95 goes to the vault.
   * \$0 goes to the trader (all collateral lost).

### What Actually Happens (Bug)

* In the edge case `liqPrice == priceAfterImpact`, the formulas for `value` and `liqMarginValue` diverge slightly.
* This makes the system think there's a small leftover (≈ \$5).
* Legacy code sends that to the trader as `usdcSentToTrader`.
* Result:

  * \$5 → liquidation fee (correct).
  * \$90 → vault (incorrect, shortfall).
  * \$5 → trader (incorrect refund).

### Why This Matters

* A liquidated trader walks away with money they shouldn't receive.
* The vault receives less than it should, distorting protocol accounting.
* While the exact equality case is rare, it can be **manipulated** or engineered by a trader if oracle values and trade sizes are chosen carefully.

## Concrete Walkthrough (Alice the Trader)

* **Setup**: Alice posts \$100 collateral, opens a long position.
* **Liquidation Trigger**: Oracle sets `liqPrice = $2,970`. Execution price with slippage = `priceAfterImpact = $2,970`.
* **Calculation**:

  * `value = 100.01` (rounding up slightly).
  * `liqMarginValue = 100.00`.
  * Difference = `0.01` → system interprets this as "leftover collateral."
* **Refund Flow**:

  * Legacy logic sends this "leftover" back to Alice.
  * Net effect: Alice unfairly gets back ≈ liquidation fee.

## Vulnerable Code Reference

From audit excerpt:

```solidity
uint256 usdcSentToVault = usdcLeftInStorage - usdcSentToTrader;
storageT.transferUsdc(address(storageT), address(this), usdcSentToVault);
vault.receiveAssets(usdcSentToVault, trade.trader);

if (usdcSentToTrader > 0) 
    storageT.transferUsdc(address(storageT), trade.trader, usdcSentToTrader);
```

* `usdcSentToTrader` is incorrectly positive due to mismatched `value` vs `liqMarginValue`.
* Refund executes even during liquidation, which should always be zero.

## Recommended Mitigation

1. **Force refund = 0 during liquidation** (quick safe fix):

    ```solidity
    if (liquidationFee > 0) {
        storageT.transferUsdc(address(storageT), address(this), liquidationFee);
        vault.distributeReward(liquidationFee);
        emit VaultLiqFeeCharged(orderId, tradeId, trade.trader, liquidationFee);

        usdcSentToTrader = 0; // critical override
    }
    ```

2. **Unify calculation logic**: Ensure `value` and `liqMarginValue` are derived from identical formulas so equality never produces divergence.
3. **Remove / restrict legacy refund logic**: Refunds should not run inside liquidation branches.
4. **Add tests & assertions**:

   * Fuzzing for equality cases.
   * `assert(usdcSentToTrader == 0)` whenever `isLiquidation == true`.

## Pattern Recognition Notes

* **Rounding Divergence**: Two formulas meant to yield the same result can differ at equality boundaries, producing unintended positive outputs.
* **Legacy Code Hazard**: Old refund logic that no longer applies to new liquidation rules introduces attack surfaces.
* **Invariant Violation**: Liquidations should *always* leave the trader with zero collateral. Any positive refund is a red flag.
* **Guard Rails**: Critical state machines should enforce invariants explicitly, not just rely on math aligning.

### Quick Recall (TL;DR)

* **Bug**: At `liqPrice == priceAfterImpact`, math mismatch makes `usdcSentToTrader` positive.
* **Impact**: Trader wrongly refunded ≈ liquidation fee, vault underfunded.
* **Fix**: Force refund = 0 in liquidation, align formulas, remove legacy refund path.
