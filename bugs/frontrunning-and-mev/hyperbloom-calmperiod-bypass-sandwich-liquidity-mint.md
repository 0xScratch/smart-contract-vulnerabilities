# Sandwich-Driven Liquidity Mint Manipulation via Calm-Period Bypass in Passive Strategy Manager

* **Severity**: Medium
* **Source**: [Pashov Audit Group - Hyperbloom Report](https://github.com/pashov/audits/blob/master/team/md/Hyperbloom-security-review_2025-06-24.md)
* **Affected Contracts**: `StrategyPassiveManagerHyperswap.sol`, `StrategyPassiveManagerKittenswap.sol`
* **Vulnerability Type**: Price Manipulation / Missing Validation / Sandwich Attack Surface

## Summary

The withdraw flow in HyperBloom's passive strategy managers conditionally calls `_onlyCalmPeriods()` **only** when:

```solidity
block.timestamp == lastDeposit && !calmAction
```

This condition is rarely true.

Because `_addLiquidity()` (executed during withdrawal) **also skips** `_onlyCalmPeriods()` whenever:

```solidity
liquidity > 0 && amountsOk
```

an attacker can **freely force the strategy to mint new V3-style liquidity based on a manipulated pool price**.

### Core issue

A malicious user can:

1. Make a tiny deposit (sets `lastDeposit`),
2. Wait one block (bypasses timestamp condition),
3. Temporarily manipulate the pool price by swapping,
4. Call `withdraw()` to trigger `_addLiquidity()`,
5. Strategy mints liquidity at the manipulated price because **no calm-period check** was enforced.

This results in **sandwich-style re-pricing**, causing user-fund loss, skewed liquidity distribution, and exploitation opportunities.

## A Better Explanation (With Simplified Example)

### Intended Behavior

The strategy is designed to:

1. **Withdraw** tokens back to the vault.
2. **Re-add liquidity** into predefined ticks.
3. **Use `_onlyCalmPeriods()`** to ensure minting only happens when prices are stable (to avoid adding liquidity at manipulated prices).
4. Skip `_addLiquidity()` when the system is paused.

### What Actually Happens (Bug)

There are two bypasses:

#### 1. Withdraw-level bypass

`withdraw()` *only* calls `_onlyCalmPeriods()` when:

```solidity
if (block.timestamp == lastDeposit && !calmAction)
    _onlyCalmPeriods();
```

This means:

* If attacker **waits even one block**, condition becomes false.
* Withdraw proceeds **without calm checks**.

#### 2. Mint-level bypass

Inside `_addLiquidity()`, the key check is:

```solidity
if (liquidity > 0 && amountsOk) {
    minting = true;
    pool.mint(...);
} else {
    if (!calmAction) _onlyCalmPeriods();
}
```

If attacker manipulates the pool price **but keeps it inside the strategy's tick range**, then:

* `liquidity > 0` → true
* `amountsOk` → true

→ **Calm-period check is skipped entirely**.
→ Mint occurs at manipulated price.

### Why This Matters

The strategy **trusts the spot price** when minting liquidity.
A price manipulation lasting only a few seconds can cause:

* The strategy to add liquidity in an **unfavorable ratio**,
* The attacker to arbitrage the manipulated state,
* Permanent portfolio skew or direct value loss.

Because minting is expensive and sensitive to price, the absence of calm checks exposes the vault to *repeatable, low-cost sandwich attacks*.

## Concrete Walkthrough (Alice & Mallory)

### Setup

* Strategy has tokens waiting to be redeployed.
* `lastDeposit` is set when someone deposits (Mallory will abuse this).
* `_onlyCalmPeriods()` is supposed to prevent liquidity minting during volatility.

### Attack flow

#### 1. Mallory prepares

Mallory deposits **1 wei** (dust) → sets `lastDeposit = now`.

#### 2. Mallory waits

She waits **one block**.
Now:

```solidity
block.timestamp != lastDeposit
```

→ withdraw() will **not** check `_onlyCalmPeriods()`.

#### 3. Mallory manipulates price

She performs a large swap in the underlying Kittenswap/Hyperswap pool to push the price temporarily.

#### 4. Mallory calls `withdraw()`

This triggers `_addLiquidity()`.

Inside `_addLiquidity()`:

* `sqrtPrice()` is manipulated.
* `liquidity > 0 && amountsOk` evaluates **true** as long as price stays within tick range.

→ The strategy **mints liquidity at manipulated ratio**.

#### 5. Mallory reverses the swap

Price returns to normal.
The strategy is left holding skewed assets → Mallory profits from arbitrage.

> **Analogy**: A shop only checks the weather before selling ice cream *on the exact minute of a restock*. If you buy a popsicle one minute earlier, the shop stops checking the weather — you can turn on a massive fan heater, and they'll refill the freezer believing it's hot outside.

## Vulnerable Code Reference

### 1) Withdraw-level conditional calm check

```solidity
if (block.timestamp == lastDeposit && !calmAction)
    _onlyCalmPeriods();
```

### 2) Mint-level calm check bypass in `_addLiquidity()`

```solidity
if (liquidity > 0 && amountsOk) {
    minting = true;
    pool.mint(...);  // executes even during manipulated price
} else {
    if (!calmAction) _onlyCalmPeriods();
}
```

### 3) No TWAP / oracle check

`sqrtPrice()` comes directly from the pool.
No sanity check → spot price can be manipulated cheaply.

## Recommended Mitigation

### **1. Enforce `_onlyCalmPeriods()` unconditionally in `_addLiquidity()`**

Add at the top:

```solidity
if (!calmAction) _onlyCalmPeriods();
```

This ensures **all price-sensitive liquidity minting** requires calm conditions.

---

### **2. Replace spot-price checks with TWAP**

Instead of using pool's `sqrtPrice()`, use:

* A 30s or 60s TWAP
* Or a max drift check: `abs(spot - twap) < threshold`

This prevents flash/manipulated prices from being used.

### **3. Reject dust deposits or implement a deposit cooldown**

Prevents attacker from cheaply resetting `lastDeposit`.

### **4. Add tests covering sandwich manipulation**

* Simulate price push → withdraw → mint → price revert.
* Ensure patched version rejects mint during manipulated state.

## Pattern Recognition Notes

* **Conditional safety checks that depend on timestamps are dangerous** when attackers can control timing.
* **Liquidity minting must never rely on spot prices without TWAP/volatility protection.**
* **Branch-based validations create bypasses**: checks must be applied at all sensitive mint paths.
* **Dust attacks** are common when logic ties security to a shared "last action" variable.
* **Sandwich surfaces appear when user operations trigger liquidity changes inside the same block as a price-moving swap.**

## TL;DR

*Bug*: Calm-period protections only run in rare branches; attacker can bypass them by waiting one block, manipulating spot price, and triggering `_addLiquidity()`.

*Impact*: Strategy mints liquidity at a manipulated price → arbitrage profit for attacker, losses for vault depositors.

*Fix*: Enforce `_onlyCalmPeriods()` directly inside `_addLiquidity()`; add TWAP checks; avoid timestamp-based gating.
