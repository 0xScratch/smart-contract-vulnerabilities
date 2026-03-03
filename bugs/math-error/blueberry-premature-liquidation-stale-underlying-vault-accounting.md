# Premature Liquidation via Stale Underlying Vault Accounting

* **Severity**: High
* **Source**: [Sherlock](https://github.com/sherlock-audit/2023-02-blueberry-judging/issues/126)
* **Affected Contract**: [BlueBerryBank.sol](https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol), [CoreOracle.sol](https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/oracle/CoreOracle.sol#L182-L189)
* **Vulnerability Type**: Accounting Error / Collateral Undervaluation / Premature Liquidation

## Summary

Blueberry calculates liquidation risk using `pos.underlyingAmount`, which only tracks the **initial ERC20 deposit amount** and does not account for yield earned inside vaults.

Since isolated collateral is deposited into yield-generating vaults, its real value increases over time. However, the protocol continues valuing the position using the original deposit amount, causing the system to **underestimate collateral value**.

As a result, positions can become liquidatable even though they are economically healthy, leading to **premature liquidation** and unfair loss of funds.

## A Better Explanation (With Simplified Example)

### Intended Behavior

1. User deposits 100 USDC via `lend()`.
2. Deposit goes into a vault and user receives vault shares.
3. Vault earns yield over time.
4. Collateral value should increase accordingly.
5. Liquidation logic should use the *current vault value* when calculating risk.

### What Actually Happens (Bug)

When liquidation risk is computed inside `getPositionRisk()`:

```solidity
uint256 cv = oracle.getUnderlyingValue(
    pos.underlyingToken,
    pos.underlyingAmount
);
```

The system uses:

```solidity
pos.underlyingAmount
```

This value:

* Increases only when `lend()` is called
* Never increases from vault yield
* Remains static forever

Meanwhile, the actual collateral grows inside the vault via:

```solidity
pos.underlyingVaultShare
```

But liquidation ignores it.

### Why This Matters

Vault shares increase in value over time.

If:

* User deposited 100 tokens
* Vault grows 10%

Real collateral value = 110
Stored collateral value = 100

Risk calculation uses 100 instead of 110.

This artificially inflates risk and may trigger liquidation early.

## Concrete Walkthrough (Alice & Time)

### Step 1 — Initial Position

Alice:

* Deposits 100 USDC
* Borrows against it

Vault gives her:

```yaml
100 vault shares
```

System stores:

```yaml
underlyingAmount = 100
underlyingVaultShare = 100
```

### Step 2 — Vault Earns 10%

After yield:

```yaml
Each share = 1.1 USDC
Total real value = 110 USDC
```

But system still sees:

```yaml
underlyingAmount = 100
```

### Step 3 — Market Moves Slightly

Assume:

```yaml
BorrowValue = 95
CollateralValue (ERC1155) = 0
LiqThreshold = 90%
```

### Real Risk

```yaml
95 / 110 = 86%
```

Safe.

### System's Risk

```yaml
95 / 100 = 95%
```

Liquidatable.

Alice gets liquidated even though her vault grew.

## Why It Happens (Root Cause)

Two different accounting variables exist:

| Variable               | Purpose                    |
| ---------------------- | -------------------------- |
| `underlyingAmount`     | Raw deposited token amount |
| `underlyingVaultShare` | Yield-bearing vault shares |

Liquidation uses the raw amount, not the dynamic share value.

This disconnect causes systematic undervaluation.

## Vulnerable Code Reference

### 1) Risk Calculation in `BlueBerryBank.sol`

```solidity
uint256 cv = oracle.getUnderlyingValue(
    pos.underlyingToken,
    pos.underlyingAmount
);
```

This uses static deposit amount.

### 2) Deposit Logic in `lend()`

```solidity
pos.underlyingAmount += amount;
pos.underlyingVaultShare += ISoftVault(bank.softVault).deposit(amount);
```

Vault shares are stored but never used in liquidation math.

### 3) Oracle Valuation in `CoreOracle.sol`

```solidity
function getUnderlyingValue(address token, uint256 amount)
```

This simply multiplies price by amount — it does not account for vault growth.

## Impact

* Users can be liquidated earlier than intended.
* Liquidators can seize collateral unfairly.
* No insolvency risk, but economic unfairness.
* System consistently undervalues isolated collateral.

This is a **directional error**:

* It never overestimates collateral.
* It always underestimates.

## Recommended Mitigation

### Primary Fix

Calculate underlying collateral value based on vault shares:

```solidity
currentValue = vault.convertToUnderlying(pos.underlyingVaultShare);
```

Or:

```solidity
currentValue = pos.underlyingVaultShare * vaultSharePrice;
```

Then pass this dynamic value into risk calculation.

### Alternative Improvement

Remove `underlyingAmount` entirely and derive collateral value solely from vault share accounting.

This prevents future drift between:

* Raw deposit amount
* Real vault value

### Defensive Recommendation

Add invariant:

```text
underlyingAmount should not be relied upon for health checks
```

All yield-bearing assets must be dynamically valued.

## Pattern Recognition Notes

### 📌 1. Static vs Dynamic Accounting Drift

If you store both:

* Deposit amount
* Share representation

Always derive health metrics from the dynamic source.

### 📌 2. Yield-Bearing Collateral Must Be Marked-to-Market

If collateral earns yield, liquidation must reflect current value, not historical input.

### 📌 3. Accounting Layer Mismatch

Common DeFi bug:

* Strategy layer grows value
* Risk engine ignores growth

Always check consistency between:

* Vault accounting
* Liquidation engine

### 📌 4. Economic Bugs ≠ Arithmetic Bugs

No overflow.
No reentrancy.
No oracle failure.

This is purely economic misvaluation.

Those are subtle and easy to miss.

## Quick Recall (TL;DR)

* **Bug**: Liquidation uses static deposit amount instead of dynamic vault value.
* **Impact**: Collateral undervalued → premature liquidation.
* **Root Cause**: Accounting mismatch between `underlyingAmount` and `underlyingVaultShare`.
* **Fix**: Derive collateral value from vault shares, not stored deposit amount.
