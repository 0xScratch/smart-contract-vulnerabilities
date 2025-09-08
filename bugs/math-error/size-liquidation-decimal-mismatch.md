# Liquidation Profit Underflow via Decimal Mismatch in Collateral-Debt Conversion

* **Severity**: High
* **Source**: [Code4rena](https://github.com/code-423n4/2024-06-size-findings/issues/21) / [One Bug Per Day](https://www.onebugperday.com/v1/1353)
* **Affected Contract**: [Liquidate.sol](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/src/libraries/actions/Liquidate.sol)
* **Vulnerability Type**: Precision Error / Decimal Mismatch / Incentive Failure

## Original Bug Description

>## Lines of code
>
>[https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/src/libraries/actions/Liquidate.sol#L90-L100](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/src/libraries/actions/Liquidate.sol#L90-L100)
>
>## Vulnerability details
>
>## Impact
>
>`Liquidation` is crucial, and all `liquidatable` `positions` must be `liquidated` to maintain the protocol's `solvency`.
>However, because the `liquidator's profit` is calculated incorrectly, users are not `incentivized` to `liquidate` `unhealthy positions`.
>This issue has a significant `impact` on the protocol.
>
>## Proof of Concept
>
>When the `collateral ratio` of a user falls below the `specified threshold`, the user's `debt positions` become `liquidatable`.
>The `liquidation` process involves the following steps:
>
>```solidity
>function executeLiquidate(State storage state, LiquidateParams calldata params)
>    external
>    returns (uint256 liquidatorProfitCollateralToken)
>{
>    DebtPosition storage debtPosition = state.getDebtPosition(params.debtPositionId);
>    LoanStatus loanStatus = state.getLoanStatus(params.debtPositionId);
>    uint256 collateralRatio = state.collateralRatio(debtPosition.borrower);
>
>    uint256 collateralProtocolPercent = state.isUserUnderwater(debtPosition.borrower)
>        ? state.feeConfig.collateralProtocolPercent
>        : state.feeConfig.overdueCollateralProtocolPercent;
>
>    uint256 assignedCollateral = state.getDebtPositionAssignedCollateral(debtPosition);  // @audit, step 1
>    uint256 debtInCollateralToken = state.debtTokenAmountToCollateralTokenAmount(debtPosition.futureValue);  // @audit, step 2
>    uint256 protocolProfitCollateralToken = 0;
>
>    if (assignedCollateral > debtInCollateralToken) {
>        uint256 liquidatorReward = Math.min(
>            assignedCollateral - debtInCollateralToken,
>            Math.mulDivUp(debtPosition.futureValue, state.feeConfig.liquidationRewardPercent, PERCENT)
>        );  // @audit, step 3
>        liquidatorProfitCollateralToken = debtInCollateralToken + liquidatorReward;
>
>        ...
>    } else {
>        liquidatorProfitCollateralToken = assignedCollateral;
>    }
>}
>```
>
>`step 1`: Calculate the `collateral` assigned to the `debt`:
>For example, if the `collateral` is `100 ETH` and the `total debt` is `1000 USDC`, and the `liquidated position's debt` is `200 USDC`, then `20 ETH` is assigned to this `debt`.
>
>`step 2`: Convert `debt` tokens to `collateral` tokens:
>For instance, if `1 ETH` equals `1000 USDC` and the `liquidated position's debt` is `200 USDC`, then this value converts to `0.2 ETH`.
>
>`step 3`: Calculate the `liquidator's profit` if the `assigned collateral` to this `position` is larger than the `debt value`:
>
>It's crucial to note that `assignedCollateral` and `debtInCollateralToken` values are denominated in `ETH's decimal` (`18`), whereas `debtPosition.futureValue` is denominated in `USDC's decimal` (`6`).
>
>As a result, the second value becomes extremely small, which fails to fully `incentivize` `liquidators`.
>
>This discrepancy is because the calculated `profit` is equal to the `exact profit` divided by `1e12`.
>
>As you can see in the log below, the profit is calculated to be extremely small.
>
>```text
>future value in usdc        ==>  82814071
>future value in WETH        ==>  82814071000000000000
>assigned collateral in WETH ==>  100000000000000000000
>********************
>liquidator profit in WETH   ==>  4140704
>exact profit in WETH        ==>  4140703550000000000
>```
>
>Please add below test to the `test/local/actions/Liquidate.t.sol`:
>
>```solidity
>function test_liquidate_profit_of_liquidator() public {
>    uint256 rewardPercent = size.feeConfig().liquidationRewardPercent;
>
>    // current reward percent is 5%
>    assertEq(rewardPercent, 50000000000000000);
>
>    _deposit(alice, weth, 100e18);
>    _deposit(alice, usdc, 100e6);
>    _deposit(bob, weth, 100e18);
>    _deposit(liquidator, weth, 100e18);
>    _deposit(liquidator, usdc, 100e6);
>
>    _buyCreditLimit(alice, block.timestamp + 365 days, YieldCurveHelper.pointCurve(365 days, 0.03e18));
>    uint256 debtPositionId = _sellCreditMarket(bob, alice, RESERVED_ID, 80e6, 365 days, false);
>
>    _setPrice(1e18);
>    assertTrue(size.isDebtPositionLiquidatable(debtPositionId));
>
>    uint256 futureValue = size.getDebtPosition(debtPositionId).futureValue;
>    uint256 assignedCollateral = size.getDebtPositionAssignedCollateral(debtPositionId);
>    uint256 debtInCollateralToken =
>        size.debtTokenAmountToCollateralTokenAmount(futureValue);
>    
>    console2.log('future value in usdc        ==> ', futureValue);
>    console2.log('future value in WETH        ==> ', debtInCollateralToken);
>    console2.log('assigned collateral in WETH ==> ', assignedCollateral);
>
>    console2.log('********************');
>
>    uint256 liquidatorProfit = _liquidate(liquidator, debtPositionId, 0);
>    console2.log('liquidator profit in WETH   ==> ', liquidatorProfit - debtInCollateralToken);
>
>    uint256 exactProfit = Math.min(
>        assignedCollateral - debtInCollateralToken,
>        Math.mulDivUp(debtInCollateralToken, rewardPercent, PERCENT)    
>    );
>    console2.log('exact profit in WETH        ==> ', exactProfit);
>}
>```
>
>## Tools Used
>
>## Recommended Mitigation Steps
>
>Use the converted value to `ETH`.
>
>```diff
>function executeLiquidate(State storage state, LiquidateParams calldata params)
>    external
>    returns (uint256 liquidatorProfitCollateralToken)
>{
>    DebtPosition storage debtPosition = state.getDebtPosition(params.debtPositionId);
>    LoanStatus loanStatus = state.getLoanStatus(params.debtPositionId);
>    uint256 collateralRatio = state.collateralRatio(debtPosition.borrower);
>
>    uint256 collateralProtocolPercent = state.isUserUnderwater(debtPosition.borrower)
>        ? state.feeConfig.collateralProtocolPercent
>        : state.feeConfig.overdueCollateralProtocolPercent;
>
>    uint256 assignedCollateral = state.getDebtPositionAssignedCollateral(debtPosition);
>    uint256 debtInCollateralToken = state.debtTokenAmountToCollateralTokenAmount(debtPosition.futureValue); 
>    uint256 protocolProfitCollateralToken = 0;
>
>    if (assignedCollateral > debtInCollateralToken) {
>        uint256 liquidatorReward = Math.min(
>            assignedCollateral - debtInCollateralToken,
>-            Math.mulDivUp(debtPosition.futureValue, state.feeConfig.liquidationRewardPercent, PERCENT)
>+            Math.mulDivUp(debtInCollateralToken, state.feeConfig.liquidationRewardPercent, PERCENT)
>        ); 
>        liquidatorProfitCollateralToken = debtInCollateralToken + liquidatorReward;
>
>        ...
>    } else {
>        liquidatorProfitCollateralToken = assignedCollateral;
>    }
>}
>```
>
>## Assessed type
>
>Math

## Summary

In Size, liquidations work by comparing a debt position's **assigned collateral** against the debt's value (converted to collateral units). If the collateral exceeds the debt, the liquidator claims the debt plus a profit margin.

However, the liquidation code mistakenly mixes **USDC (6 decimals)** with **ETH (18 decimals)**. Specifically, it uses `debtPosition.futureValue` (in USDC, 6 decimals) against ETH-denominated values. This mismatch causes the computed liquidator reward to shrink by **1e12**, effectively eliminating incentives.

As a result, liquidators won't act, unhealthy positions remain unliquidated, and the protocol risks bad debt accumulation.

## A Better Explanation (With Simplified Example)

### Intended Behavior

1. **Assign collateral to debt**:

   * Example: User deposits **100 ETH** as collateral.
   * Borrows **1000 USDC**.
   * One debt position = **200 USDC**.
   * Assigned collateral = `(200 / 1000) * 100 ETH = 20 ETH`.

2. **Convert debt into collateral terms**:

   * Assume **1 ETH = 1000 USDC**.
   * 200 USDC = `200 / 1000 = 0.2 ETH`.

3. **Liquidator reward**:

   * If assigned collateral (20 ETH) > debt (0.2 ETH), liquidator gets 0.2 ETH + bonus (e.g., 5%).
   * Correct reward = `0.2 ETH * 5% = 0.01 ETH`.

### What Actually Happens (Bug)

* The liquidation logic uses `debtPosition.futureValue` (200 USDC with **6 decimals**) against ETH values (18 decimals).
* Instead of comparing **0.2 ETH**, the code effectively compares **200e6 (USDC, 6 decimals)** directly in ETH math.
* The result: reward = `0.01 ETH / 1e12 = 0.00000000001 ETH`.

Liquidators receive near-zero profit, making liquidations irrational.

### Why This Matters

* **Liquidators stop participating**: No one spends gas for near-zero returns.
* **Bad debt accumulates**: Risky borrowers can remain under-collateralized.
* **Protocol insolvency**: Collateral backing fails if liquidations stall.
* **Silent failure**: The protocol appears functional but is economically broken.

### Concrete Walkthrough (Alice & Mallory)

* **Setup**:

  * Alice deposits **100 ETH**, borrows **1000 USDC**.
  * Assigned collateral for 200 USDC debt = **20 ETH**.

* **Liquidation event**:

  * Debt converted properly = **0.2 ETH**.
  * Correct liquidator reward = 0.01 ETH.

* **Buggy code path**:

  * Uses `200e6` (6 decimals USDC) as if it were 18 decimals.
  * Reward shrinks by factor of 1e12 → 0.00000000001 ETH.

* **Impact**: Mallory (a liquidator) sees zero gain and refuses to liquidate. Alice's position lingers, harming protocol solvency.

> **Analogy**: Imagine a cashier mis-reading \$200.00 as \$0.000000200. The refund you get is essentially dust, so no cashier bothers processing returns.

## Vulnerable Code Reference

### 1) Assigned collateral calculation (denominated in ETH, 18 decimals)**

```solidity
uint256 assignedCollateral = (collateral * debtPosition.futureValue) / totalDebt;
```

### 2) Debt value conversion uses `futureValue` in USDC (6 decimals)**

```solidity
// futureValue in USDC (6 decimals) is treated as if it were in 18 decimals
uint256 debtInCollateral = debtPosition.futureValue * PRICE_CONVERSION_RATE; 
```

### 3) Profit calculation shrinks reward by 1e12**

```solidity
uint256 profit = assignedCollateral - debtInCollateral; 
// profit is 1e12 times smaller due to decimal mismatch
```

## Recommended Mitigation

1. **Unify decimal precision**:

   * Normalize all debt values into **18-decimal precision** before comparing with ETH collateral.

   ```solidity
   uint256 normalizedDebt = debtPosition.futureValue * 1e12; // USDC (6) → 18
   ```

2. **Introduce shared math library**:

   * Centralize conversion logic to avoid scattered mismatches.

3. **Add invariant tests**:

   * Assert that liquidation profit is non-zero for under-collateralized positions.

4. **Audit liquidation economics**:

   * Run scenario tests where collateral/debt ratios vary to ensure liquidators are always incentivized.

## Pattern Recognition Notes

* **Decimal Mismatch Risk**: Mixing tokens with different decimals (6 vs 18) without normalization often creates silent underflows or overflows.
* **Incentive Fragility**: Even if logic "works," dust-sized payouts kill economic incentives.
* **Cross-asset Systems**: Protocols with multiple asset types must unify decimals at ingress, not ad-hoc.
* **Silent Failure Mode**: Unlike reverts/DoS, this bug keeps the system "running" but economically broken.

### Quick Recall (TL;DR)

* **Bug**: USDC (6 decimals) debt used directly in ETH (18 decimals) math.
* **Impact**: Liquidator rewards shrink by 1e12 → no one liquidates.
* **Fix**: Normalize USDC debt values to 18 decimals before profit calculation.
