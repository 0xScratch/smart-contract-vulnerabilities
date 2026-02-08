# Partial Order Fulfillment Discount via Low-Decimal ERC20 in `BasicOrderFulfiller`

* **Severity**: Medium
* **Source**: [Code4rena](https://github.com/code-423n4/2022-05-opensea-seaport-findings/issues/129)
* **Affected Contract**: [BasicOrderFulfiller.sol](https://github.com/code-423n4/2022-05-opensea-seaport/blob/main/contracts/lib/BasicOrderFulfiller.sol)
* **Vulnerability Type**: Precision Loss / Partial Payment Exploit

## Summary

In OpenSea's **Seaport** protocol, `BasicOrderFulfiller` enables simple trades between buyers and sellers. However, for ERC20 tokens with **low decimals** (e.g., WBTC — 8 decimals), the protocol's **rounding and encoding behavior** can be exploited.
Under certain conditions, an attacker can **fulfill an order without paying the full amount owed**, effectively buying NFTs at a *slight discount*.

This occurs because the protocol's calculations for ERC20 token transfers assume full-precision decimals (like 18), but tokens such as WBTC or USDC have fewer. Combined with multiple consideration items and calldata limits, small rounding gaps arise — letting attackers omit a minimal fraction of payment.

## A Better Explanation (With Simplified Example)

### Intended Behavior

When fulfilling an order:

1. Buyer specifies payment amount (e.g., 1 WBTC = 100,000,000 satoshis).
2. Contract validates and transfers the total amount of tokens listed in `considerationItems`.
3. The seller receives the entire payment expected.

### What Actually Happens (Bug)

For ERC20 tokens with **less than 18 decimals**, Seaport encodes the payment data such that the least significant digits can be **lost due to precision rounding**.
If an order contains **multiple `considerationItems`**, the rounding differences accumulate — allowing the buyer to **send slightly less** than the total intended amount.

For example:

```solidity
// Pseudo example
// Expected total: 1.00000000 WBTC (100,000,000 units)
// Actual paid:    0.99999998 WBTC (99,999,998 units)
```

The missing value (2 satoshis) seems negligible, but in principle, it's a *discounted trade* the buyer shouldn't get.

### Why This Matters

Although the monetary loss per trade is tiny, the exploit breaks the **economic integrity** of Seaport's order matching.
In systems processing millions of orders, the discrepancy could be abused for **micro-arbitrage** or **sandwich-style precision games**.

However, the exploit:

* Requires a token with **low decimals** (e.g., WBTC - 8 decimals).
* Works only when there are **multiple consideration items** in the order.
* Is **limited by Ethereum's calldata size and gas limits** — attackers can't make the discrepancy arbitrarily large.

### Concrete Walkthrough (Alice & Mallory)

* **Setup**: NFT is priced at `1.00000000 WBTC`. The order has 3 consideration items (e.g., royalties, platform fee, seller).
* **Mallory (attacker)**: Constructs a `BasicOrder` that exploits rounding during encoding:

  * Due to WBTC's 8 decimals, total transfers add up to `0.99999998 WBTC`.
  * Order still passes all validation checks.
* **Outcome**: Mallory gets the NFT while paying slightly less than the intended amount.
  Economic integrity is violated, though the deviation is small.

> **Analogy**: Like paying $99.999 when the actual price is $100 — the system accepts it because it can't handle paise precision correctly.

## Vulnerable Code Reference

Vulnerability resides in **calculation and validation logic** of ERC20 consideration transfers in `BasicOrderFulfiller.sol`, particularly where:

```solidity
// Pseudo representation
uint256 totalConsiderationAmount = sum(considerationItems.amount);
_safeTransferERC20(token, from, to, totalConsiderationAmount);
```

The `amount` values come from user input and can underflow by a few units when decimals are < 18, leading to a **slight underpayment**.

## Judge's Key Notes (from Audit Resolution)

* Attack possible **only for ERC20 tokens with <18 decimals**, primarily **WBTC (8 decimals)**.
* Requires multiple consideration items.
* With Ethereum's current calldata gas cost (~16 gas/byte) and block limit (30M gas), the maximum extractable value is about **$375** (assuming WBTC at $20,000).
* The **transaction gas cost** to perform this attack would likely **exceed the gain**.
* Therefore, **severity = Medium**, as assets aren't at direct risk but value can leak under niche conditions.

## Recommended Mitigation

1. **Normalize token amounts**
   Convert ERC20 token amounts to a consistent internal precision (e.g., 18 decimals) before performing calculations.

2. **Rounding safeguards**
   Always round **upward** when distributing ERC20 consideration amounts to avoid underpayment.

3. **Explicit decimal validation**
   Validate that token decimals are within supported precision before accepting the order:

   ```solidity
   require(IERC20Metadata(token).decimals() >= 18, "Unsupported decimal token");
   ```

4. **Fuzz testing with low-decimal tokens**
   Include fuzz tests for tokens like WBTC (8 decimals) and USDC (6 decimals) to ensure rounding errors don't accumulate.

## Pattern Recognition Notes

* **Decimal Precision Risk**: Solidity's integer math combined with low-decimal tokens often leads to rounding/underflow issues.
* **Order Aggregation Vulnerability**: When orders involve multiple recipients, small rounding gaps multiply.
* **Economic Exploit, Not Functional Breakage**: The protocol still works, but at a *discounted price path*.
* **Mitigation via Normalization**: Always scale token amounts to a fixed precision before arithmetic.

### Quick Recall (TL;DR)

* **Bug**: Low-decimal ERC20 tokens cause rounding gaps in multi-item orders.
* **Impact**: Attacker fulfills an order while paying slightly less than owed.
* **Limitations**: Requires low-decimal token (like WBTC) and specific order structure.
* **Fix**: Normalize token precision and round up during distribution.
* **Severity**: **Medium**, as exploitability is niche and value leakage minimal.
