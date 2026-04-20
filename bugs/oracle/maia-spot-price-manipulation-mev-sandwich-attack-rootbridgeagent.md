# Spot Price Manipulation via `slot0()` Leading to MEV Sandwich Losses in RootBridgeAgent

* **Severity**: High
* **Source**: [Code4rena](https://github.com/code-423n4/2023-05-maia-findings/issues/823)
* **Affected Contract**: [RootBridgeAgent.sol](https://github.com/code-423n4/2023-05-maia/blob/cfed0dfa3bebdac0993b1b42239b4944eb0b196c/src/ulysses-omnichain/RootBridgeAgent.sol)
* **Vulnerability Type**: MEV / Price Oracle Manipulation / Sandwich Attack

## Summary

`RootBridgeAgent.sol` relies on the **instantaneous spot price** from Uniswap V3 via `slot0()` (`sqrtPriceX96`) to compute swap price limits in `_gasSwapIn` and `_gasSwapOut`.

Because this value reflects the **latest pool state (single block)**, it can be **manipulated by attackers using flashloans and MEV sandwich attacks**. The manipulated price is then used as a trusted reference for swaps, causing the protocol to execute trades at **artificially unfavorable rates**, resulting in **direct loss of funds**.

## A Better Explanation (With Simplified Example)

### Intended Behavior

1. Fetch current pool price using:

   ```solidity
   (uint160 sqrtPriceX96,,,,,,) = IUniswapV3Pool(poolAddress).slot0();
   ```
2. Apply a small allowed price impact:

   ```solidity
   sqrtPriceLimitX96 = sqrtPriceX96 ± impact;
   ```
3. Perform swap within that limit:

   ```solidity
   pool.swap(...)
   ```

👉 Assumption:
The current price is fair, and small deviation is acceptable.

### What Actually Happens (Bug)

* `slot0()` returns the **spot price**, not a safe oracle price.
* This value can be **temporarily manipulated within a single transaction/block**.
* The contract **blindly trusts this manipulated price** to set swap bounds.

### Why This Matters

* The protocol becomes vulnerable to **MEV sandwich attacks**.
* Users/protocol swaps execute at **bad prices**.
* Attackers extract **risk-free profit**.
* Losses occur silently during normal operations.

### Concrete Walkthrough (Alice & Mallory)

#### Setup

* Alice triggers a swap via the bridge
* Contract will use `slot0()` price

### Step 1: Mallory (attacker) front-runs

* Takes a flashloan
* Trades heavily in the pool
* **Artificially pushes price up**

👉 Now `slot0()` returns a **fake inflated price**

### Step 2: Alice's transaction executes

* Contract reads manipulated `sqrtPriceX96`
* Calculates price limit based on fake price
* Executes swap

👉 Alice **overpays** (gets fewer tokens)

### Step 3: Mallory back-runs

* Reverses trade
* Restores price
* Keeps profit

### Result

* Alice / protocol → **loses value**
* Mallory → **profits risk-free**

> **Analogy**:
> It's like checking the price of gold from a shop owner who briefly inflates it just when you ask, sells to you at that fake price, and then immediately drops it back.

## Vulnerable Code Reference

### **1) Spot price fetched from `slot0()`**

```solidity
(uint160 sqrtPriceX96,,,,,,) = IUniswapV3Pool(poolAddress).slot0();
```

👉 Returns **latest price (manipulable within a block)**

### **2) Price limit derived from manipulable value**

```solidity
uint160 exactSqrtPriceImpact = (sqrtPriceX96 * (priceImpactPercentage / 2)) / GLOBAL_DIVISIONER;

uint160 sqrtPriceLimitX96 =
    zeroForOneOnInflow
        ? sqrtPriceX96 - exactSqrtPriceImpact
        : sqrtPriceX96 + exactSqrtPriceImpact;
```

👉 Entire protection logic depends on a **tainted input**

### **3) Swap executed using manipulated bounds**

```solidity
IUniswapV3Pool(poolAddress).swap(
    address(this),
    zeroForOneOnInflow,
    int256(_amount),
    sqrtPriceLimitX96,
    ...
);
```

👉 Swap executes at **attacker-influenced price**

## Recommended Mitigation

### 1. **Use TWAP instead of spot price (primary fix)**

Replace `slot0()` with a **Time-Weighted Average Price (TWAP)**:

```solidity
// Example using observe()
(uint32[] memory secondsAgos) = ...;
(int56[] memory tickCumulatives,) = pool.observe(secondsAgos);

// derive TWAP price from cumulative ticks
```

👉 TWAP resists short-term manipulation.

### 2. **Avoid using AMM spot price as oracle**

* Spot price = execution state
* Not safe for:

  * pricing logic
  * limits
  * validations

### 3. **Add slippage protections independent of pool state**

* Accept user-provided min/max outputs
* Enforce stricter bounds

### 4. **Consider oracle integration**

Use trusted oracle sources like:

* Chainlink
* Internal TWAP feeds

## Pattern Recognition Notes

### **1) Spot Price Oracle Misuse**

Using AMM spot price (`slot0`) as a trusted oracle → **high-risk anti-pattern**

### **2) MEV Sandwich Surface**

Any logic that:

* Reads price
* Immediately executes swap
  → is vulnerable to sandwiching

### **3) Flashloan Amplification**

Flashloans make manipulation:

* Cheap
* Atomic
* Risk-free

### **4) Derived Trust on Tainted Data**

Even if you add limits (like priceImpact %),
if base input is compromised → **entire protection collapses**

### **5) Lack of Time Dimension**

Secure pricing requires:

* Time averaging (TWAP)
* Not instantaneous snapshots

## Quick Recall (TL;DR)

* **Bug**: Uses Uniswap `slot0()` (spot price) as trusted input
* **Impact**: Attackers manipulate price → sandwich swaps → extract profit
* **Root Cause**: No protection against **single-block price manipulation**
* **Fix**: Use **TWAP** instead of spot price
