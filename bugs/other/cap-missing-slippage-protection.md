# Missing Slippage Protection in Liquidation Allows Unexpected Collateral Loss

* **Severity**: Medium
* **Source**: [Sherlock](https://github.com/sherlock-audit/2025-07-cap-judging/issues/542)
* **Affected Contract**: [LiquidationLogic.sol](https://github.com/sherlock-audit/2025-07-cap/blob/main/cap-contracts/contracts/lendingPool/libraries/LiquidationLogic.sol)
* **Vulnerability Type**: Missing Slippage Protection / Value Mismatch in Liquidation

## Summary

The `liquidate()` function in `LiquidationLogic` lacks a **minimum acceptable collateral parameter**, exposing liquidators to potential losses.
When total slashable collateral (`totalSlashableCollateral`) is **less than the expected liquidation value**, the function silently **reduces the collateral payout** instead of reverting — forcing liquidators to accept less than expected, potentially below the repayment value.

## A Better Explanation (With Simplified Example)

### Intended Behavior

Liquidators should:

1. Repay part of a borrower's (agent's) debt.
2. Receive collateral of **equivalent or greater value** (plus liquidation bonus).
3. Be protected from receiving **less than expected** (slippage protection).

In a safe liquidation:

```solidity
expectedValue = (repaidAmount + bonus) * assetPrice
collateralToReceive = expectedValue
require(collateralToReceive >= minCollateralOut, "Slippage too high");
```

### What Actually Happens (Bug)

In `LiquidationLogic::liquidate()`:

```solidity
liquidatedValue =
    (liquidated + (liquidated * bonus / 1e27)) * assetPrice / (10 ** $.reservesData[params.asset].decimals);

if (totalSlashableCollateral < liquidatedValue)
    liquidatedValue = totalSlashableCollateral;
```

Here, if the agent's **available collateral** is lower than the calculated `liquidatedValue`, it's **clamped** down silently.
No mechanism exists for liquidators to **specify a minimum expected value**, so even if market conditions or prior liquidations reduce collateral, the transaction **still succeeds**, giving less collateral than anticipated.

### Why This Matters

* Liquidators base off-chain liquidation decisions on **expected reward math**.
* Between off-chain computation and on-chain execution:

  * The **epoch** (which affects `totalSlashableCollateral`) can change with time.
  * Another liquidation may **reduce available collateral** before this transaction executes.
* Without slippage protection, the liquidator's transaction executes successfully but yields **less collateral than expected**, resulting in **losses**.

### Concrete Walkthrough (Alice & Bob)

* **Setup**:
  Bob (liquidator) finds that Alice's (agent's) health factor is low and prepares a liquidation transaction.

* **Bob's off-chain calculation**:

  * He expects to receive `$100` worth of collateral after repaying `$95` in debt.

* **On-chain scenario**:

  * Between his calculation and transaction execution, another liquidator partially seizes Alice's collateral, lowering `totalSlashableCollateral` to `$60`.

* **Liquidation execution**:

  * The `liquidatedValue` is capped to `$60`.
  * The transaction **does not revert** — Bob receives only `$60` collateral for repaying `$95` of debt.

* **Result**:
  Bob loses `$35` due to missing slippage protection.

> **Analogy**: You agree to trade $100 for 1 gold coin, but by the time the trade executes, only half the coin is left. The system still completes the trade and gives you half a coin, even though you paid the full $100.

## Vulnerable Code Reference

```solidity
function liquidate(ILender.LenderStorage storage $, ILender.RepayParams memory params)
    external
    returns (uint256 liquidatedValue)
{
    // ... calculations for liquidated and bonus

    liquidatedValue =
        (liquidated + (liquidated * bonus / 1e27)) * assetPrice / (10 ** $.reservesData[params.asset].decimals);

    // Collateral capped silently — no revert or min check
    if (totalSlashableCollateral < liquidatedValue)
        liquidatedValue = totalSlashableCollateral;

    // No parameter like `minCollateralOut` or `minValueOut`
    if (liquidatedValue > 0)
        IDelegation($.delegation).slash(params.agent, params.caller, liquidatedValue);
}
```

## Recommended Mitigation

1. **Add slippage protection parameter**
   Introduce a `minCollateralOut` argument to the `liquidate()` function, ensuring liquidators can define a minimum acceptable collateral value:

   ```solidity
   function liquidate(
       ILender.LenderStorage storage $,
       ILender.RepayParams memory params,
       uint256 minCollateralOut
   ) external returns (uint256 liquidatedValue) {
       // ... existing logic
       if (totalSlashableCollateral < liquidatedValue)
           liquidatedValue = totalSlashableCollateral;

       require(liquidatedValue >= minCollateralOut, "Slippage too high");
   }
   ```

2. **Include slippage checks in front-end or SDK integrations**

   * Off-chain tools should automatically estimate `minCollateralOut` based on price oracles and apply a small safety margin.

3. **Emit event for shortfall**

   * Log whenever `totalSlashableCollateral < liquidatedValue`, aiding observability and analytics.

## Pattern Recognition Notes

* **Slippage Risk in Protocol-to-Protocol Interactions**:
  When expected vs. actual output value can diverge due to timing, network latency, or dynamic collateral updates.

* **Silent Value Clamping**:
  Assigning `x = min(x, available)` without informing or reverting leads to hidden loss paths.

* **Temporal Race Conditions**:
  Collateral amount depends on **epoch/time**, which can shift between calculation and execution.

* **Mitigation Pattern**:
  Always include `minOut` parameters for state-changing transactions dependent on dynamic on-chain values (e.g., swaps, redemptions, liquidations).

### Quick Recall (TL;DR)

* **Bug**: Liquidation lacks `minCollateralOut` parameter.
* **Impact**: Liquidators may receive less collateral than expected if total slashable collateral decreases before execution.
* **Fix**: Add `minCollateralOut` (slippage guard) and require it before finalizing liquidation.
