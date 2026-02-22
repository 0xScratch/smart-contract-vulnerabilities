# Rely On Balancer Oracle Which Is Not Updated Frequently

* **Severity**: Medium
* **Source**: [Sherlock](https://github.com/sherlock-audit/2022-09-notional-judging/issues/67)
* **Affected Contract**: [TwoTokenPoolUtils.sol](https://github.com/sherlock-audit/2022-09-notional/blob/main/leveraged-vaults/contracts/vaults/balancer/internal/pool/TwoTokenPoolUtils.sol#L66), [BalancerUtils.sol](https://github.com/sherlock-audit/2022-09-notional/blob/main/leveraged-vaults/contracts/vaults/balancer/internal/pool/BalancerUtils.sol#L21)
* **Vulnerability Type**: Oracle / Price Feed Risk / Stale Price Dependency

## Summary

The MetaStable2 Balancer leverage vault relies on a weighted price composed of **Balancer TWAP (60%)** and **Chainlink (40%)**. However, the Balancer oracle only updates when the underlying pool receives transactions (e.g., swaps).

Because the referenced stETH/ETH Balancer pool exhibits **very low activity (~1-3 transactions per day)**, the TWAP price can remain stale for long periods (~16 hours on average). This stale price is heavily weighted in the vault's pricing logic, causing the reported asset value to deviate from the true market price.

As a result, critical vault operations—including settlement, liquidation, and borrowing—may operate on inaccurate valuations.

## A Better Explanation (With Simplified Example)

### Intended Behavior

1. Vault fetches price from:

   * Balancer TWAP oracle
   * Chainlink oracle
2. Prices are combined via weighted average:

   ```
   finalPrice =
       balancerPrice * 60%
     + chainlinkPrice * 40%
   ```
3. Result is used across the vault for:

   * strategy valuation
   * collateral checks
   * liquidation logic

The design assumes both sources are reasonably fresh.

### What Actually Happens (Bug)

Balancer oracle updates **only when pool state changes**.

In the referenced pool:

* ~1-3 swaps per day
* sometimes only **one update daily**
* TWAP may lag real market by many hours

Therefore:

* Market price moves quickly
* Chainlink updates normally
* Balancer TWAP stays stale
* Weighted average becomes skewed (since Balancer has 60% weight)

### Why This Matters

Because the price feeds into nearly all vault logic:

* Users may **over-borrow** if assets are overvalued
* Users may be **prematurely liquidated** if undervalued
* Large settlements may amplify small pricing errors
* Risk accumulates silently over time

This is especially dangerous in leveraged vaults where precision is critical.

## Concrete Walkthrough (Alice & Market Move)

**Initial state**

```
Real stETH/ETH = 1.0
Balancer TWAP = 1.0
Chainlink = 1.0
Final = 1.0 ✅
```

**Market moves (no pool activity)**

```
Real stETH/ETH = 0.94
Chainlink = 0.94 ✅ updated
Balancer TWAP = 1.0 ❌ stale
```

Weighted result:

```
Final =
  1.0 × 60%
+ 0.94 × 40%
= 0.976
```

**Error:** ~3.8% overvaluation.

**Impact**

If Alice deposits stETH as collateral:

* protocol thinks her collateral is worth more
* she can borrow more than safe
* vault accumulates bad debt risk

> **Analogy**: Imagine averaging two thermometers where one updates every minute and the other updates once a day—but you trust the slow one more. Your final temperature reading will often be wrong.

## Vulnerable Code Reference

### 1) Balancer TWAP fetch

```solidity
uint256 balancerPrice = BalancerUtils._getTimeWeightedOraclePrice(
    address(poolContext.basePool.pool),
    IPriceOracle.Variable.PAIR_PRICE,
    oracleContext.oracleWindowInSeconds
);
```

### 2) Underlying oracle call

```solidity
return IPriceOracle(pool).getTimeWeightedAverage(queries)[0];
```

⚠️ Updates only when pool activity occurs.

### 3) Weighted price composition

```solidity
oraclePairPrice =
    (balancerWeightedPrice + chainlinkWeightedPrice) /
    BalancerConstants.BALANCER_ORACLE_WEIGHT_PRECISION;
```

With configuration:

```
Balancer weight = 60%
Chainlink weight = 40%
```

## Impact

### 1. Borrowing Risk

Overvalued collateral allows users to:

* borrow excessive funds
* increase protocol insolvency risk

### 2. Premature Liquidations

If TWAP lags downward:

* healthy positions may appear undercollateralized
* users may be deleveraged unfairly

### 3. Vault Settlement Distortion

This pricing feeds into:

```
_convertStrategyToUnderlying
```

Large vault positions amplify even small price deviations, potentially causing significant accounting inaccuracies.

## Recommended Mitigation

### 1. Use Chainlink as Primary Oracle (strongly recommended)

Make Chainlink the dominant source for base asset pricing:

```
Chainlink weight > Balancer weight
```

### 2. Avoid low-activity pools for oracle sourcing

Before trusting AMM TWAP:

* verify swap frequency
* verify liquidity depth
* verify update cadence

Low-volume pools should not be primary price sources.

### 3. Add freshness / heartbeat checks

Validate that the Balancer TWAP:

* updated recently
* or diverges minimally from Chainlink

Example defensive check:

```solidity
require(
    abs(balancerPrice - chainlinkPrice) < MAX_DEVIATION,
    "stale or divergent oracle"
);
```

### 4. Restrict Balancer oracle usage

Balancer TWAP is acceptable for:

✅ LP token pricing (pool is source of truth)

But risky for:

❌ base asset pricing (ETH, stETH, etc.)

## Pattern Recognition Notes

* **Low-Activity TWAP Risk**: AMM oracles that update only on swaps become stale in quiet markets.
* **Weighted Oracle Dominance**: Even with multiple sources, improper weighting can let the weakest oracle dominate.
* **Hidden Liveness Assumptions**: Systems often assume pools are active; always verify empirically.
* **Leverage Amplification**: Small oracle errors become large financial risks in leveraged systems.
* **Oracle Source Suitability**: AMM TWAPs are best when the pool itself is the price authority—not when pricing external assets.

## Quick Recall (TL;DR)

* **Bug**: Vault heavily trusts Balancer TWAP from a low-activity pool.

* **Root cause**: Oracle updates only on swaps → becomes stale.

* **Impact**: Mispricing affects borrowing, liquidation, and settlement.

* **Fix**: Prefer Chainlink as primary; add freshness and deviation checks.