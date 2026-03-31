# Sandwich Attack on Premium Accrual in ProtectionPool

* **Severity**: High
* **Source**: [Sherlock](https://github.com/sherlock-audit/2023-02-carapace-judging/issues/26)
* **Affected Contract**: [ProtectionPool.sol](https://github.com/sherlock-audit/2023-02-carapace/blob/main/contracts/core/pool/ProtectionPool.sol#L279-L354)
* **Vulnerability Type**: Economic Attack / Sandwich Attack / Share Inflation

## Summary

The `accruePremiumAndExpireProtections()` function increases `totalSTokenUnderlying` when premium is accrued, **without minting new shares**. This increases the **exchange rate** of the pool.

A malicious user can **front-run** premium accrual by depositing funds before the premium is added and **withdraw immediately after**, capturing a **disproportionate share of premium**.

This allows attackers to **steal premium meant for long-term liquidity providers**, violating the **fair distribution** principle described in the Carapace whitepaper.

## A Better Explanation (With Simplified Example)

### Intended Behavior

Premiums collected from protection buyers should be distributed fairly among **existing liquidity providers**.

1. Sellers deposit capital and receive **sTokens**
2. Buyers pay premium
3. Premium increases pool capital
4. Existing sellers earn yield proportionally

### What Actually Happens (Bug)

The premium accrual logic:

```solidity
if (_totalPremiumAccrued > 0) {
    totalPremiumAccrued += _totalPremiumAccrued;
    totalSTokenUnderlying += _totalPremiumAccrued;
}
```

This **adds capital but does not mint shares**.

This increases:

```solidity
exchangeRate = totalCapital / totalShares
```

This allows attackers to:

1. Deposit before premium accrual
2. Wait for exchange rate increase
3. Withdraw immediately
4. Capture profit

## Why This Matters

This allows:

* Risk-free profit
* Premium stealing
* Unfair distribution
* MEV / sandwich exploitation
* Repeatable economic drain

This is especially dangerous because:

* `accruePremiumAndExpireProtections()` is public
* Premium accrual is predictable
* No lockup required

## Concrete Walkthrough (Alice & Bob)

### Initial State

```text
totalSTokenUnderlying = 1,000,000
totalShares = 1,000,000
exchangeRate = 1
```

Bob holds:

```text
BobShares = 100,000
```

### Step 1 — Premium Accrual Incoming

Suppose premium:

```text
+100,000
```

Bob sees this and prepares attack.

### Step 2 — Bob Front-Runs

Bob deposits:

```text
100,000 underlying
```

After deposit:

```text
capital = 1,100,000
shares = 1,100,000
exchangeRate = 1
```

Bob now owns:

```text
200,000 shares
```

### Step 3 — Premium Accrual Happens

Premium added:

```text
+100,000
```

New state:

```text
capital = 1,200,000
shares = 1,100,000
exchangeRate = 1.0909
```

### Step 4 — Bob Withdraws

Bob withdraws:

```text
100,000 shares
```

Withdrawal:

```
100,000 × 1.0909 = 109,090
```

### Final Profit

```text
Deposit: 100,000 
Withdraw: 109,090
Profit: 9,090
```

This is **risk-free profit via sandwich attack**.

## Vulnerable Code Reference

### Premium Accrual Increases Capital

```solidity
if (_totalPremiumAccrued > 0) {
    totalPremiumAccrued += _totalPremiumAccrued;
    totalSTokenUnderlying += _totalPremiumAccrued;
}
```

This increases capital without minting shares.

### Deposit Mints Shares Based on Current Exchange Rate

```solidity
uint256 _sTokenShares = convertToSToken(_underlyingAmount);
```

Attacker deposits before premium accrual.

### Withdraw Uses Updated Exchange Rate

```solidity
uint256 _underlyingAmountToTransfer = convertToUnderlying(
  _sTokenWithdrawalAmount
);
```

Attacker withdraws after exchange rate increases.

## Root Cause

The protocol:

* Adds premium instantly
* Does not distribute premium gradually
* Does not prevent short-term deposits

This allows **instant profit extraction**.

## Recommended Mitigation

### 1. Stream Premium Over Time (Best Fix)

Instead of:

```text
instant premium addition
```

Use:

```text
premiumPerSecond
```

This prevents instant profit.

### 2. Deposit Limits

Limit sudden large deposits:

```text
maxDepositPerCycle
```

### 3. Withdrawal Delay

Increase lock period after deposit.

### 4. Snapshot-Based Distribution

Distribute premium to:

```text
snapshot holders only
```

## Pattern Recognition Notes

### Share Inflation Attacks

If:

```text
capital increases
shares remain constant
```

→ exchange rate manipulation possible

### Sandwich Attacks

Pattern:

```text
Deposit
Yield event
Withdraw
```

### MEV Exploitable Functions

Functions like:

```solidity
accruePremium()
distributeRewards()
rebalance()
```

Are vulnerable.

### Fair Distribution Violations

When:

* Instant rewards
* No lockups
* Public distribution trigger

→ High risk

## Quick Recall (TL;DR)

* **Bug**: Premium added instantly without minting shares
* **Attack**: Deposit → premium accrual → withdraw
* **Impact**: Risk-free profit
* **Fix**: Stream premium or lock deposits
