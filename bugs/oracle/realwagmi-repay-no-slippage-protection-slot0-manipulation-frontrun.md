# No Slippage Protection During Repayment via slot0() Price Manipulation

* **Severity**: High
* **Source**: [Sherlock](https://github.com/sherlock-audit/2023-10-real-wagmi-judging/issues/109)
* **Affected Contract**: [LiquidityBorrowingManager.sol](https://github.com/sherlock-audit/2023-10-real-wagmi/blob/b33752757fd6a9f404b8577c1eae6c5774b3a0db/wagmi-leverage/contracts/abstract/LiquidityManager.sol#L265)
* **Vulnerability Type**: Frontrunning / MEV / Slippage Protection Failure / Oracle Manipulation

## Summary

The `repay()` function in **Real Wagmi** lacks proper slippage protection. It relies on **Uniswap V3 `slot0()`** to determine swap amounts during repayment. Since `slot0()` returns the **current pool price**, it can be **temporarily manipulated** via frontrunning swaps.

Additionally, the contract calculates **slippage dynamically** using values derived from this manipulated price. Because both the expected output and minimum output are derived from the same manipulated value, **the slippage protection becomes ineffective**.

This allows attackers (or MEV bots) to **sandwich repayment transactions**, reducing the profit of repayers without causing transaction failure.

## A Better Explanation (With Simplified Example)

### Intended Behavior

During repayment:

1. User calls `repay()`
2. Protocol calculates how much token to swap
3. Swap executes with slippage protection
4. Liquidity is restored
5. Remaining tokens returned as profit to repayer

The slippage protection should **prevent unfavorable swaps**.

### What Actually Happens (Bug)

Instead of using a **fixed slippage threshold**, the protocol:

1. Reads **current price** from Uniswap `slot0()`
2. Calculates expected swap output
3. Dynamically computes slippage from this manipulated value
4. Executes swap

Because the slippage is calculated **after price manipulation**, the swap **always succeeds**, even under unfavorable conditions.

## Step-by-Step Attack Flow

### Step 1 — Repayer submits transaction

Bob calls:

```solidity
repay()
```

This transaction enters the mempool.

### Step 2 — Attacker frontruns

Attacker swaps tokens:

```yaml
WBTC → WETH
```

This temporarily **moves Uniswap price**.

### Step 3 — Repay reads manipulated price

Protocol reads:

```solidity
(sqrtPriceX96, , , , , , ) = IUniswapV3Pool(poolAddress).slot0();
```

Now it uses **fake price**.

### Step 4 — Dynamic slippage calculated

```solidity
amountOutMinimum =
(saleTokenAmountOut * params.slippageBP1000) / Constants.BPS;
```

Since `saleTokenAmountOut` is already manipulated:

* Minimum output also reduced
* Swap still succeeds

### Step 5 — Repayer loses profit

Repayer receives fewer tokens.

Attacker backruns to restore price and capture profit.

## Concrete Walkthrough (Alice & Bob)

### Without Attack

Bob repays:

```text
Expected Output: 100 WETH
Received: 100 WETH
```

### With Attack

Alice frontruns:

```text
Price manipulated
Expected Output becomes: 95 WETH
MinOut becomes: 94 WETH
```

Swap executes:

```text
Bob receives: 95 WETH
```

Bob loses **5 WETH worth of value**.

## Proof of Concept Result

From provided PoC:

### Without Manipulation

```text
WETH Balance:
99005137946252426108
```

### With Manipulation

```text
WETH Balance:
99000233164653177505
```

### Loss

```text
4904781599248603 wei
≈ $8 loss
```

Although small in this case, **loss scales with liquidity size**.

## Why This Matters

* Repay transactions can be **reliably sandwiched**
* MEV bots can exploit automatically
* Profit continuously drained from repayers
* No protection mechanism exists

## Vulnerable Code Reference

### 1) Price from slot0() (Manipulatable)

```solidity
function _getCurrentSqrtPriceX96(
    bool zeroForA,
    address tokenA,
    address tokenB,
    uint24 fee
) private view returns (uint160 sqrtPriceX96) {
    if (!zeroForA) {
        (tokenA, tokenB) = (tokenB, tokenA);
    }
    address poolAddress = computePoolAddress(tokenA, tokenB, fee);
    (sqrtPriceX96, , , , , , ) = IUniswapV3Pool(poolAddress).slot0();
}
```

### 2) Dynamic swap calculation

```solidity
(uint256 holdTokenAmountIn, uint256 amount0, uint256 amount1) =
_getHoldTokenAmountIn(...)
```

### 3) Dynamic slippage protection

```solidity
amountOutMinimum =
(saleTokenAmountOut * params.slippageBP1000) / Constants.BPS;
```

This creates **fake slippage protection**.

## Root Cause

Two issues combined:

### 1. Use of manipulatable price source

`slot0()` can be manipulated via flash swaps.

### 2. Dynamic slippage calculation

Minimum output derived from manipulated value.

## Impact

* Repayer profit reduced
* Sandwich attacks possible
* Continuous MEV extraction
* No transaction failure

Severity considered **High** because:

* Exploitable by anyone
* No permission required
* Repeatable attack
* Affects all users

## Recommended Mitigation

### 1. Use TWAP instead of slot0()

Instead of:

```solidity
slot0()
```

Use:

* Uniswap TWAP
* Oracle-based pricing

### 2. Fixed slippage protection

Allow users to set:

```solidity
amountOutMinimum
```

Instead of dynamic calculation.

### 3. Add pre-swap validation

Use cached price before swap execution.

## Pattern Recognition Notes

### Dynamic Slippage Anti-Pattern

If slippage calculated from manipulated value:

→ Slippage protection becomes meaningless

### slot0() Oracle Misuse

Using:

```yaml
Uniswap slot0()
```

Without TWAP:

→ Vulnerable to price manipulation

### Sandwichable Swap Logic

When:

* Swap happens inside function
* No fixed slippage

→ Repay function becomes sandwichable

## Quick Recall (TL;DR)

* **Bug**: Repay uses manipulatable `slot0()` price and dynamic slippage
* **Impact**: Repayers can be sandwiched and lose profit
* **Fix**: Use TWAP and fixed slippage protection
