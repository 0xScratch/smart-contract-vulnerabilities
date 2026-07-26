# Hardcoded 2.5% Slippage Allows Repeated Sandwich Attacks to Drain Strategy Value

- **Severity:** Medium
- **Source:** [Code4rena](https://github.com/code-423n4/2023-07-tapioca-findings/issues/1430)
- **Affected Contract:** `LidoEthStrategy.sol` (Link ain't working)
- **Category:** Economic / Price Manipulation / Slippage Configuration

## Summary

`LidoEthStrategy` uses a **hardcoded 2.5% slippage tolerance** whenever it swaps **stETH → ETH** through Curve during withdrawals.

Although this was intended to prevent unnecessary transaction reverts, the tolerance is excessively large for the highly liquid Curve stETH/ETH pool. An attacker can temporarily manipulate the pool price via a sandwich attack so that the strategy executes its swap at nearly the full allowed 2.5% loss.

Because the attacker controls the temporary price movement, the value lost by the strategy becomes the attacker's profit. If the strategy manages a sufficiently large amount of assets, this attack remains profitable and can be repeated over many withdrawals, gradually draining a significant portion of the strategy's value.

## Simplified Explanation

Imagine a vending machine selling a bottle that normally costs **$100**.

The owner configures it with the following rule:

> "I'm willing to accept anything above **$97.50**."

A scammer realizes this.

Every time the owner comes to exchange bottles, the scammer temporarily manipulates the local market so that the bottle appears to be worth only **$97.50**.

The vending machine happily accepts the trade because it is still within its allowed tolerance.

Immediately afterwards, the scammer restores the market price back to $100 and pockets the $2.50 difference.

Nothing is technically broken.

The machine simply accepted a trade under an overly generous rule.

That is exactly what happens in this vulnerability.

## Intended Behavior

During withdrawals:

1. The strategy holds users' assets as **stETH**.
2. A user requests a withdrawal.
3. The strategy swaps the required **stETH** for **ETH** using Curve.
4. ETH is wrapped into WETH.
5. The user receives the withdrawal.

Because swap prices may fluctuate slightly, the strategy allows a small amount of slippage before reverting.

## Actual Behavior

Instead of allowing only a tiny amount of slippage, the strategy hardcodes a tolerance of **2.5%**.

This means the strategy willingly accepts receiving only **97.5 ETH** when swapping assets worth approximately **100 ETH**.

An attacker can intentionally manipulate the Curve pool just before the withdrawal so that the swap executes at this unfavorable price.

The transaction succeeds because it still satisfies the configured minimum output.

The strategy permanently loses value while the attacker captures the difference.

## Attack Walkthrough

Assume:

- The strategy holds a very large amount of stETH.
- Alice submits a withdrawal transaction.
- Mallory monitors the mempool.

### Step 1 — Withdrawal Appears

Alice submits a withdrawal that requires the strategy to swap stETH into ETH.

---

### Step 2 — Mallory Front-runs

Before Alice's transaction executes, Mallory performs a very large swap on the Curve stETH/ETH pool.

This temporarily pushes the exchange rate against the strategy.

As a result:

```
1 stETH < 1 ETH
```

for a short period.

---

### Step 3 — Strategy Executes

The strategy now performs its swap.

Instead of receiving approximately:

```
100 ETH
```

it receives:

```
97.5 ETH
```

The transaction does **not** revert because the result is still within the hardcoded 2.5% slippage limit.

---

### Step 4 — Mallory Back-runs

Immediately afterwards, Mallory performs the opposite trade.

The Curve pool returns close to its original state.

The temporary price manipulation disappears.

---

### Step 5 — Profit

The strategy permanently lost value during its swap.

Mallory captures most of that loss as profit.

This process can be repeated for future withdrawals.

## Why Existing Checks Fail

The only protection is the minimum acceptable output:

```solidity
uint256 minAmount =
    toWithdraw - (toWithdraw * 250) / 10_000;
```

which effectively says:

> "Accept anything above 97.5% of the expected amount."

Unfortunately, the attacker intentionally manipulates the pool so the swap returns almost exactly that minimum value.

Since the swap still satisfies the configured threshold, every validation passes and the transaction succeeds.

The protocol therefore treats an economically harmful trade as perfectly valid.

## Root Cause

The strategy assumes that allowing **2.5% slippage** on Curve's highly liquid stETH/ETH pool is safe.

This assumption is incorrect.

The slippage tolerance is large enough that temporarily manipulating the pool becomes cheaper than the value extracted from sufficiently large withdrawals.

In other words:

- **Manipulation cost** < **Value lost by the strategy**

Once this inequality holds, repeated sandwich attacks become profitable.

## Vulnerable Code Reference

```solidity
uint256 toWithdraw = amount - queued;

uint256 minAmount =
    toWithdraw - (toWithdraw * 250) / 10_000;

uint256 obtainedEth = curveStEthPool.exchange(
    1,
    0,
    toWithdraw,
    minAmount
);
```

The strategy hardcodes a **2.5%** acceptable loss during every withdrawal.

## Impact

For sufficiently large strategies:

- Withdrawals consistently execute at manipulated prices.
- Each withdrawal leaks protocol value.
- The attacker profits from every successful sandwich attack.
- Repeated withdrawals gradually drain the strategy's assets.
- The attack remains profitable until the remaining TVL becomes too small relative to the manipulation cost.

This is therefore **not** merely one profitable MEV opportunity, but a mechanism for repeatedly extracting value over time.

## Recommended Mitigation

The contest recommendation is to avoid exposing the strategy to manipulable single-sided swaps.

Possible approaches include:

- Denominating the strategy directly in **stETH**.
- Holding **Curve LP tokens** instead of repeatedly swapping back into ETH.
- Eliminating unnecessary stETH → ETH conversions whenever possible.

If swaps cannot be avoided:

- Use significantly tighter slippage limits.
- Consider dynamically calculating acceptable slippage based on current liquidity.
- Avoid relying solely on a manipulable spot price for large swaps.

## Pattern Recognition Notes

When auditing DeFi protocols, pay close attention whenever you see:

- Hardcoded slippage percentages.
- AMM swaps performed during deposits or withdrawals.
- Large protocol TVL interacting with publicly accessible liquidity pools.
- Transactions that can be observed and sandwiched from the mempool.
- Strategies assuming that a fixed slippage tolerance is always safe.

A common red flag is:

> **Large value + Public AMM + Loose slippage = Potential sandwich attack.**

## Quick Recall (TL;DR)

- **Bug**: Hardcoded **2.5% slippage** on Curve withdrawals.
- **Root Cause**: The strategy accepts an excessively poor exchange rate during stETH → ETH swaps.
- **Exploit**: An attacker temporarily manipulates the Curve pool before a withdrawal, causing the strategy to swap at the worst acceptable price, then restores the market afterwards.
- **Impact**: Each withdrawal leaks value to the attacker. If the strategy is sufficiently large, the attack can be repeated until the remaining TVL is no longer profitable to exploit.
- **Fix**: Avoid unnecessary single-sided swaps, or significantly tighten and dynamically calculate acceptable slippage limits.
