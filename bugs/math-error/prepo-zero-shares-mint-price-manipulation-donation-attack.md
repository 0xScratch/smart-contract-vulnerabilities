# Zero-Share Mint via Total Asset Inflation in `Collateral.sol`

* **Severity**: High
* **Source**: [Code4rena](https://github.com/code-423n4/2022-03-prepo-findings/issues/27)
* **Affected Contract**: [`Collateral.sol`](https://github.com/code-423n4/2022-03-prepo/blob/main/contracts/core/Collateral.sol)
* **Vulnerability Type**: Share Price Manipulation / Incorrect Share Issuance / Economic Exploit

## Summary

`Collateral.sol` mints vault shares (preCT) based on:

```solidity
shares = depositValue * totalSupply / totalAssets
```

However, *totalAssets* is derived from the **raw balance** of the `StrategyController`:

```solidity
totalAssets = baseToken.balanceOf(collateral) + strategyController.totalValue()
```

Because `totalValue()` includes *all tokens held by the strategy controller*, **any user can artificially inflate vault assets** by donating tokens directly to the strategy controller.

If an attacker **mints the first share** with a tiny deposit, then massively increases `totalAssets` with a large donation, the share price skyrockets. Subsequent depositors receive:

```solidity
shares ≈ 0
```

meaning **zero shares minted**, yet their deposits are still accepted → funds are effectively absorbed by the attacker's single share.

This locks honest users out of receiving fair ownership and breaks the vault's economic model.

## A Better Explanation (With Simplified Example)

### Intended Behavior

1. **User deposits baseToken** → contract takes small fee → deposits into strategy.

2. Contract calculates how many vault shares to mint:

   * If no supply exists:

     ```solidity
     shares = depositValue
     ```

   * Otherwise:

     ```solidity
     shares = depositValue * totalSupply / totalAssets
     ```

3. Shares represent proportional claim on aggregated assets.

This model assumes **totalAssets cannot be arbitrarily inflated**.

### What Actually Happens (Bug)

Because `totalAssets` includes **every token held by the strategy controller**, regardless of source, an attacker can:

1. Become the **first depositor** and mint a trivial amount of shares (e.g., 1 share).
2. **Send an enormous amount of tokens** directly to the strategy controller **without using `deposit()`**.
3. This makes the vault *think* it controls massive assets and drives the share price extremely high.
4. Honest users deposit normal amounts but receive:

   ```solidity
   shares = smallDeposit * tinySupply / hugeTotalAssets = 0
   ```

Their deposits succeed → funds go into the vault → but they get **zero shares**.

The attacker's single share remains the sole claim on the entire vault.

### Why This Matters

* Users lose their deposits entirely — they receive **0 shares**.
* Attacker becomes the **sole owner** of all vault assets.
* Protocol TVL becomes a meaningless, attacker-inflated number.
* No mechanism exists to unwind or prevent the manipulation once it happens.

This is economically severe and blocks honest participation in the protocol.

## Concrete Walkthrough (Bob & Mallory)

### Step 1 — Mallory's trivial initial deposit

Mallory deposits:

```solidity
2 wei
```

After 1 wei fee → **1 wei deposited** → totalSupply becomes:

```solidity
1 share
```

Mallory now owns **100% of vault**.

### Step 2 — Mallory inflates strategy assets

Mallory directly transfers:

```solidity
1,000,000 base tokens
```

to the `StrategyController`.

No deposit function is triggered → but `totalValue()` now reports ~1,000,000 assets.

The vault now believes:

```solidity
1 share = 1,000,000 base tokens
```

### Step 3 — Bob deposits legitimate amount

Bob deposits:

```solidity
10,000 tokens
```

Share calculation:

```solidity
shares = 10,000 * 1 / 1,000,000 = 0.01 shares → truncated to 0
```

So:

* Bob receives **0 shares**
* Bob permanently loses **10,000 tokens**
* Mallory still owns **1 share**, and thus all vault assets.

### Final Effect

> **A single malicious initial deposit + donation manipulates share price such that all future deposits mint 0 shares.**

This is economically identical to a permanent freeze-out attack.

## Vulnerable Code Reference

### 1) Share minting logic vulnerable to asset inflation

```solidity
if (totalSupply() == 0) {
    _shares = _amountToDeposit;
} else {
    _shares = (_amountToDeposit * totalSupply()) / (valueBefore);
}
```

**`valueBefore` is manipulatable** because it includes arbitrary donated funds in the strategy controller.

### 2) `totalAssets()` trusts external balances

```solidity
function totalAssets() public view override returns (uint256) {
    return baseToken.balanceOf(address(this))
        + strategyController.totalValue();
}
```

`totalValue()` includes any externally-transferred tokens.

### 3) StrategyController side-effects make donation trivial

StrategyController deposits **its entire balance** during calls, so donated funds inflate perceived asset value seamlessly.

## Recommended Mitigation

### 1. Follow Uniswap V2: Mint initial "minimum liquidity" to `address(0)`

Uniswap mints **1,000 LP tokens** to the zero address to prevent initial supply manipulation.

Apply same logic:

```solidity
if (totalSupply() == 0) {
    _mint(address(0), MIN_LIQUIDITY);
    _shares = _amountToDeposit - MIN_LIQUIDITY;
}
```

This prevents a malicious user from owning 100% of supply at inception.

### 2. Reject zero share mints

Ensure deposits never produce zero shares:

```solidity
require(_shares > 0, "zero shares minted");
```

### 3. Seed vault during initialization

Deployers can call an initialization-phase deposit so early users cannot manipulate first-share price.

### 4. Normalize donations or disallow external transfers

Any method that stops external manipulation of `totalAssets`, such as:

* Only count strategy *principal* + yield
* Maintain internal accounting for deposits only
* Require strategyController.totalValue to ignore raw token balances

## Pattern Recognition Notes

* **First Depositor Manipulation**: Initial liquidity minted 1:1 to shares allows someone to claim ownership of the entire vault.
* **Donation Attacks**: Systems that rely on `balanceOf()` for pricing can be manipulated by arbitrary "donations" that distort valuation.
* **Integer Truncation**: Small-supply / huge-assets ratios cause minting to round down to zero.
* **LP Token Price Oracles**: Without a protected initialization step, any AMM-style share formula can be hijacked.
* **Economic Freeze-Out**: Instead of DoS by reverts, the attack creates a DoS by making participation economically impossible.

## Quick Recall (TL;DR)

* **Bug**: Attacker inflates `totalAssets` using donations after minting initial shares.
* **Impact**: All future deposits mint **0 shares**, causing total loss of user funds.
* **Fix**: Burn minimum initial liquidity, disallow zero-share mints, sanitize asset calculations.
