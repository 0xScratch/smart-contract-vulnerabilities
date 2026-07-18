# Modified `FullMath` Library Causes Permanent Fund Lock When Price Moves Outside the Liquidity Range

* **Severity:** High
* **Source:** [Code4rena](https://github.com/code-423n4/2023-08-goodentry-findings/issues/58)
* **Affected Contract:** [TokenisableRange.sol](https://github.com/code-423n4/2023-08-goodentry/blob/71c0c0eca8af957202ccdbf5ce2f2a514ffe2e24/contracts/TokenisableRange.sol), [FullMath.sol](https://github.com/code-423n4/2023-08-goodentry/blob/71c0c0eca8af957202ccdbf5ce2f2a514ffe2e24/contracts/lib/FullMath.sol#L2)
* **Vulnerability Type:** Modified Dependency / Arithmetic Underflow / Denial of Service (Fund Lock)

## Summary

`TokenisableRange` relies on Uniswap's `LiquidityAmounts.getAmountsForLiquidity()` helper to convert LP liquidity into the corresponding amounts of the underlying assets.

However, the imported `FullMath` library was modified to compile under Solidity **0.8.x**, changing the arithmetic behavior that the original Uniswap implementation intentionally relied upon.

As a result, under valid market conditions where the current pool price moves outside the configured liquidity range, `getAmountsForLiquidity()` can revert due to an arithmetic underflow.

Because both **deposit** and **withdraw** operations always execute `claimFee()`, and `claimFee()` depends on this calculation, the entire vault becomes unusable. Users can neither deposit nor withdraw, causing the underlying assets to become locked until prices return to a non-reverting state—or permanently if they never do.

## Simplified Explanation

Imagine a vault that owns a Uniswap V3 LP position.

Whenever someone wants to:

* deposit more liquidity, or
* withdraw their share,

the vault first asks:

> "Given today's market price, how many TOKEN0 and TOKEN1 does this LP position currently represent?"

To answer this question it calls:

```text
LiquidityAmounts.getAmountsForLiquidity()
```

Unfortunately, because the imported `FullMath` library no longer behaves like the original Uniswap implementation, this calculation can fail whenever the market price leaves the configured liquidity range in a certain direction.

Instead of returning token balances, the helper reverts.

Since this helper sits inside `claimFee()`, and `claimFee()` is executed before every deposit and withdrawal, every user operation begins reverting.

The vault still owns all assets, but nobody can access them.

## Intended vs Actual Behavior

### Intended Behavior

* The protocol should correctly calculate the underlying token balances for any valid Uniswap LP position.
* Deposits should continue working regardless of whether the market price is inside or outside the configured range.
* Withdrawals should always allow LP holders to redeem their proportional share.
* Moving outside the liquidity range should simply change the asset composition of the LP position, not make it unusable.

---

### Actual Behavior

When the current pool price satisfies:

```text
sqrtRatioX96 < sqrtRatioAX96
```

`LiquidityAmounts.getAmountsForLiquidity()` reverts because the modified `FullMath` implementation encounters an arithmetic underflow.

This causes:

```text
deposit()
    ↓
claimFee()
    ↓
returnExpectedBalanceWithoutFees()
    ↓
getAmountsForLiquidity()
    ↓
REVERT
```

The exact same execution path exists for `withdraw()`.

As a result, both deposits and withdrawals become impossible.

## Attack Walkthrough

Consider a liquidity range between **1700-1800 USDC/ETH**.

### Step 1

Alice and Bob deposit liquidity while ETH trades at **1750 USDC**.

Everything works normally.

---

### Step 2

Months later, ETH appreciates to **2300 USDC**.

The Uniswap position is now completely outside its original range.

This is perfectly valid behavior for Uniswap.

---

### Step 3

Alice decides to withdraw her share.

Execution proceeds as:

```text
withdraw()

↓

claimFee()

↓

returnExpectedBalanceWithoutFees()

↓

LiquidityAmounts.getAmountsForLiquidity()

↓

Arithmetic underflow

↓

REVERT
```

The withdrawal fails.

---

### Step 4

Bob also attempts to withdraw.

The same revert occurs.

---

### Step 5

A new user attempts to deposit additional liquidity.

Deposits also begin with `claimFee()`, so they fail for the same reason.

The vault becomes completely frozen.

---

### Step 6

If the market price never returns to a non-reverting range, every user's funds remain locked indefinitely.

## Root Cause

The root cause is **not simply an arithmetic underflow**.

The real issue is that a core Uniswap dependency (`FullMath`) was modified to compile under Solidity 0.8 without preserving the arithmetic behavior expected by the original algorithm.

The original library intentionally relies on intermediate arithmetic behavior ("phantom overflow") during its calculations.

After recompiling under Solidity 0.8, automatic overflow and underflow checks cause those intermediate calculations to revert instead of completing successfully.

Since `TokenisableRange` assumes `LiquidityAmounts.getAmountsForLiquidity()` always succeeds, this unexpected revert propagates into critical user operations.

In short:

* Modified dependency changes arithmetic semantics.
* Math helper unexpectedly reverts.
* Fee accounting cannot complete.
* Deposits and withdrawals become impossible.

## Vulnerable Code Reference

The vulnerable execution chain is:

```text
deposit()
        │
        ▼
claimFee()
        │
        ▼
returnExpectedBalanceWithoutFees()
        │
        ▼
LiquidityAmounts.getAmountsForLiquidity()
        │
        ▼
Modified FullMath
        │
        ▼
Arithmetic underflow
```

Relevant locations:

* `deposit()`
* `withdraw()`
* `claimFee()`
* `returnExpectedBalanceWithoutFees()`
* `LiquidityAmounts.getAmountsForLiquidity()`
* Modified `FullMath.sol`

## Why the Existing Checks Fail

The protocol assumes that Uniswap's liquidity calculation helper always returns valid token balances.

There are no fallback paths, defensive checks, or error handling around this dependency.

As soon as the helper reverts:

* fee accounting cannot complete,
* deposits cannot continue,
* withdrawals cannot continue.

The protocol therefore inherits the failure of a low-level dependency and escalates it into a protocol-wide denial of service.

## Recommended Mitigation

Restore the original `FullMath` implementation so that it preserves the arithmetic behavior expected by the Uniswap algorithm.

Alternatively, preserve the original arithmetic semantics (for example, by carefully using `unchecked` where appropriate), although restoring the upstream implementation is the safer and preferred approach.

The goal is to ensure that `LiquidityAmounts.getAmountsForLiquidity()` continues producing valid results for legitimate out-of-range liquidity positions instead of reverting.

## Pattern Recognition Notes

This vulnerability highlights several important audit patterns:

### 1. Modified Third-Party Libraries

Changing compiler versions or "upgrading" imported libraries can silently invalidate assumptions made by the original implementation.

---

### 2. Dependency Failures Propagate Upward

A failure inside a low-level math helper may eventually disable high-level protocol functionality if critical execution paths depend on it.

---

### 3. Core Accounting Must Never Depend on Fragile Calculations

Functions required for deposits or withdrawals should avoid relying on calculations that can unexpectedly revert under valid market conditions.

---

### 4. Out-of-Range Liquidity Is Normal

For Uniswap V3, positions moving outside their configured range are expected behavior.

Protocols integrating with Uniswap should correctly handle these situations rather than assuming prices always remain inside the chosen interval.

## Quick Recall (TL;DR)

* **Bug:** Modified `FullMath` changes arithmetic behavior expected by Uniswap.
* **Trigger:** Current pool price moves outside the liquidity range (`sqrtRatioX96 < sqrtRatioAX96`).
* **Effect:** `LiquidityAmounts.getAmountsForLiquidity()` reverts.
* **Propagation:** `claimFee()` fails.
* **Impact:** Both `deposit()` and `withdraw()` revert, locking all assets inside the vault.
* **Fix:** Restore the original `FullMath` implementation (or otherwise preserve its intended arithmetic semantics).
