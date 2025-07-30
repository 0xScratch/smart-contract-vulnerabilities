# Ineffective Deadline Usage in Particle's LiquidityPosition Library

- **Severity**: Medium
- **Source**: [Code4rena](https://github.com/code-423n4/2023-12-particle-findings/issues/59) / [One Bug Per Day](https://www.onebugperday.com/v1/802)
- **Affected Contract**: [LiquidityPosition.sol](https://github.com/code-423n4/2023-12-particle/blob/a3af40839b24aa13f5764d4f84933dbfa8bc8134/contracts/libraries/LiquidityPosition.sol)
- **Vulnerability Type**: Business Logic Flaw — Missing/Improper Timeout/Expiry Control

## Original Bug Description

>## Lines of code
>
>[https://github.com/code-423n4/2023-12-particle/blob/a3af40839b24aa13f5764d4f84933dbfa8bc8134/contracts/libraries/LiquidityPosition.sol#L144](https://github.com/code-423n4/2023-12-particle/blob/a3af40839b24aa13f5764d4f84933dbfa8bc8134/contracts/libraries/LiquidityPosition.sol#L144)
>[https://github.com/code-423n4/2023-12-particle/blob/a3af40839b24aa13f5764d4f84933dbfa8bc8134/contracts/libraries/LiquidityPosition.sol#L197](https://github.com/code-423n4/2023-12-particle/blob/a3af40839b24aa13f5764d4f84933dbfa8bc8134/contracts/libraries/LiquidityPosition.sol#L197)
>[https://github.com/code-423n4/2023-12-particle/blob/a3af40839b24aa13f5764d4f84933dbfa8bc8134/contracts/libraries/LiquidityPosition.sol#L260](https://github.com/code-423n4/2023-12-particle/blob/a3af40839b24aa13f5764d4f84933dbfa8bc8134/contracts/libraries/LiquidityPosition.sol#L260)
>
>## Vulnerability details
>
>## Summary
>
>The protocol is using `block.timestamp` as the deadline argument while interacting with the Uniswap NFT Position Manager, which completely defeats the purpose of using a deadline.
>
>## Impact
>
>Actions in the Uniswap NonfungiblePositionManager contract are protected by a `deadline` parameter to limit the execution of pending transactions. Functions that modify the liquidity of the pool check this parameter against the current block timestamp in order to discard expired actions.
>
>These interactions with the Uniswap position are present in the LiquidityPosition library. The functions `mint()`, `increaseLiquidity()` and `decreaseLiquidity()` call their corresponding functions in the Uniswap Position Manager, providing `block.timestamp` as the argument for the `deadline` parameter:
>
>[https://github.com/code-423n4/2023-12-particle/blob/a3af40839b24aa13f5764d4f84933dbfa8bc8134/contracts/libraries/LiquidityPosition.sol#L131-L146](https://github.com/code-423n4/2023-12-particle/blob/a3af40839b24aa13f5764d4f84933dbfa8bc8134/contracts/libraries/LiquidityPosition.sol#L131-L146)
>
>```solidity
>131:         // mint the position
>132:         (tokenId, liquidity, amount0Minted, amount1Minted) = Base.UNI_POSITION_MANAGER.mint(
>133:             INonfungiblePositionManager.MintParams({
>134:                 token0: params.token0,
>135:                 token1: params.token1,
>136:                 fee: params.fee,
>137:                 tickLower: params.tickLower,
>138:                 tickUpper: params.tickUpper,
>139:                 amount0Desired: params.amount0ToMint,
>140:                 amount1Desired: params.amount1ToMint,
>141:                 amount0Min: params.amount0Min,
>142:                 amount1Min: params.amount1Min,
>143:                 recipient: address(this),
>144:                 deadline: block.timestamp
>145:             })
>146:         );
>```
>
>[https://github.com/code-423n4/2023-12-particle/blob/a3af40839b24aa13f5764d4f84933dbfa8bc8134/contracts/libraries/LiquidityPosition.sol#L189-L199](https://github.com/code-423n4/2023-12-particle/blob/a3af40839b24aa13f5764d4f84933dbfa8bc8134/contracts/libraries/LiquidityPosition.sol#L189-L199)
>
>```solidity
>189:         // increase liquidity via position manager
>190:         (liquidity, amount0Added, amount1Added) = Base.UNI_POSITION_MANAGER.increaseLiquidity(
>191:             INonfungiblePositionManager.IncreaseLiquidityParams({
>192:                 tokenId: tokenId,
>193:                 amount0Desired: amount0,
>194:                 amount1Desired: amount1,
>195:                 amount0Min: 0,
>196:                 amount1Min: 0,
>197:                 deadline: block.timestamp
>198:             })
>199:         );
>```
>
>[https://github.com/code-423n4/2023-12-particle/blob/a3af40839b24aa13f5764d4f84933dbfa8bc8134/contracts/libraries/LiquidityPosition.sol#L254-L262](https://github.com/code-423n4/2023-12-particle/blob/a3af40839b24aa13f5764d4f84933dbfa8bc8134/contracts/libraries/LiquidityPosition.sol#L254-L262)
>
>```solidity
>254:         (amount0, amount1) = Base.UNI_POSITION_MANAGER.decreaseLiquidity(
>255:             INonfungiblePositionManager.DecreaseLiquidityParams({
>256:                 tokenId: tokenId,
>257:                 liquidity: liquidity,
>258:                 amount0Min: 0,
>259:                 amount1Min: 0,
>260:                 deadline: block.timestamp
>261:             })
>262:         );
>```
>
>Using `block.timestamp` as the deadline is effectively a no-operation that has no effect nor protection. Since `block.timestamp` will take the timestamp value when the transaction gets mined, the check will end up comparing `block.timestamp` against the same value, i.e. `block.timestamp <= block.timestamp` (see [https://github.com/Uniswap/v3-periphery/blob/697c2474757ea89fec12a4e6db16a574fe259610/contracts/base/PeripheryValidation.sol#L7](https://github.com/Uniswap/v3-periphery/blob/697c2474757ea89fec12a4e6db16a574fe259610/contracts/base/PeripheryValidation.sol#L7)).
>
>Failure to provide a proper deadline value enables pending transactions to be maliciously executed at a later point. Transactions that provide an insufficient amount of gas such that they are not mined within a reasonable amount of time, can be picked by malicious actors or MEV bots and executed later in detriment of the submitter.
>
>See [this issue](https://github.com/code-423n4/2022-12-backed-findings/issues/64) for an excellent reference on the topic (the author runs a MEV bot).
>
>## Recommendation
>
>Add a deadline parameter to each of the functions that are used to manage the liquidity position, `ParticlePositionManager.mint()`, `ParticlePositionManager.increaseLiquidity()` and `ParticlePositionManager.decreaseLiquidity()`. Forward this parameter to the corresponding underlying calls to the Uniswap NonfungiblePositionManager contract.
>
>## Assessed type
>
>Other

## Overview

Particle's `LiquidityPosition` library calls Uniswap V3's `mint`, `increaseLiquidity`, and `decreaseLiquidity` functions but passes `block.timestamp` as the `deadline` parameter. Because Uniswap's periphery validation checks

```solidity
require(block.timestamp <= params.deadline);
```

and `params.deadline == block.timestamp` at execution time, these require-statements always pass and do not prevent stale, long-pending transactions from executing when mined. This defeats the very purpose of a deadline and exposes users to unexpected execution and MEV risks.

## A Better Explanation (With Simplified Example)

### 1. Intended Deadline Behavior

- **User sets** a fixed future deadline off-chain (e.g., current time + 15 minutes), baked into the transaction call data.
- When a miner includes the transaction in a later block, Uniswap's router validates:

    ```solidity
    require(block.timestamp <= deadlineOffChain);
    ```

- If too much time has passed, the call reverts, protecting users from stale mempool execution.

### 2. What Particle Does Today

In `LiquidityPosition.sol`, each call uses:

```solidity
// mint(), increaseLiquidity(), decreaseLiquidity()...
deadline: block.timestamp
```

- `block.timestamp` here is the timestamp **of the block** when the transaction is mined.
- Passing it as the deadline means Uniswap checks:

    ```solidity
    require(block.timestamp <= block.timestamp);
    ```

- This condition is always true, so **no real deadline** exists.

### 3. Why This Fails to Protect Users

| Risk Scenario | With Proper Deadline | With `deadline = block.timestamp` |
| :-- | :-- | :-- |
| Transaction pending for hours | Reverts if mined after cutoff | Always executes regardless of delay |
| Price swings heavily | Swap reverts rather than executes at old price | Swap executes at outdated rates |
| MEV sandwich attacks | Narrow execution window limits sandwich feasibility | Unlimited window maximizes MEV risk |

### 4. Vulnerable Code Reference

```solidity
// LiquidityPosition.sol
(tokenId, liquidity, amount0Minted, amount1Minted) = Base.UNI_POSITION_MANAGER.mint(
    INonfungiblePositionManager.MintParams({
        /* … */
        recipient: address(this),
        deadline: block.timestamp      // ❌ wrong—always "now"
    })
);

(liquidity, amount0Added, amount1Added) = Base.UNI_POSITION_MANAGER.increaseLiquidity(
    INonfungiblePositionManager.IncreaseLiquidityParams({
        /* … */
        deadline: block.timestamp      // ❌ wrong—always "now"
    })
);

(amount0, amount1) = Base.UNI_POSITION_MANAGER.decreaseLiquidity(
    INonfungiblePositionManager.DecreaseLiquidityParams({
        /* … */
        deadline: block.timestamp      // ❌ wrong—always "now"
    })
);
```

### 5. Recommended Mitigation

1. **Add a `uint256 deadline` parameter** to all public mint/increase/decrease functions in the ParticlePositionManager API.
2. **Require** `block.timestamp <= deadline` at the start of each function:

    ```solidity
    require(block.timestamp <= deadline, "Deadline passed");
    ```

3. **Forward** this user-supplied deadline directly into the Uniswap Position Manager call:

    ```solidity
    deadline: deadline
    ```

4. **Remove any on-chain `block.timestamp` calculations** for deadline—deadlines must be determined off-chain before signing.

## Pattern Recognition Notes

- **On-Chain vs Off-Chain Deadline Evaluation**
Never compute deadlines on-chain at execution time. Deadlines must be computed by the caller (wallet/frontend) before signing and remain immutable in calldata.
- **Atomic vs Multi-Block Flows**
For any multi-step protocol interaction that relies on time-bounds, ensure each call accepts externally provided time constraints rather than computing them within the transaction.
- **MEV \& Mempool Risks**
Absent or ineffective deadlines leave transactions exposed in the mempool indefinitely, enabling price-slippage attacks and sandwich MEV with no time-based barrier.
- **Dual Protection: Slippage + Deadline**
Always combine slippage checks (`amountMin` or price limits) with genuine deadlines to protect users against both economic (price) and temporal (timing) risks.

## Bonus

Here's a really good article/blog on DeFi slippage by Dacian: [DeFi Slippage Attacks](https://dacian.me/defi-slippage-attacks)
