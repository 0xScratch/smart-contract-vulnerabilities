# Front-Run Pool Initialization & Forced Mispriced Liquidity Deposit in `getOrCreatePoolAndAddLiquidity`

* **Severity**: Medium
* **Source**: [Solodit (Code4rena)](https://solodit.cyfrin.io/issues/m-01-routergetorcreatepoolandaddliquidity-can-be-frontrunned-which-leads-to-price-manipulation-code4rena-maverick-maverick-git)
* **Affected Contract**: `Router.sol` (`getOrCreatePoolAndAddLiquidity()`) - Link wasn't working
* **Vulnerability Type**: Front-Running / Initialization Manipulation / Liquidity Mispricing

## Summary

The Router exposes a convenience method `getOrCreatePoolAndAddLiquidity()` that allows a user to:

1. **Create a new pool** with a chosen initial `activeTick`, and
2. **Add liquidity** to that pool,
   **in the same transaction**.

Because the function first tries to "get *or* create" the pool, an attacker can **front-run** a liquidity provider's transaction by **creating the pool first**, but with a **different initial active tick** (i.e., a different price).

When the user's tx finally executes, the router sees that *the pool already exists*, skips creation, and adds the user's liquidity into the attacker-initialized price range. The attacker then performs an immediate swap to extract value from the mispriced deposit, causing **losses to the first liquidity provider**.

Although a user *can* protect themselves by setting `isDelta = false` in `AddLiquidityParams`, this protection is **optional**, non-obvious, and many users assume they are adding liquidity relative to the tick they intended to initialize with. Therefore the issue is real, but severity is **downgraded to Medium**.

## A Better Explanation (With Simplified Example)

### Intended Behavior

A user calling `getOrCreatePoolAndAddLiquidity()` wants to:

* Create a pool for `tokenA:tokenB` *if it doesn't exist*, using `poolParams.activeTick` as the starting price.
* Add liquidity according to `AddLiquidityParams`, which may specify:

  * **Absolute bins** (`isDelta = false`)
  * **Relative bins** (`isDelta = true`) → relative to the **current active tick** of the pool

So the user is assuming:
"**I create the pool → its active tick is the one I provided → my relative bin instruction is based on that tick.**"

### What Actually Happens (Bug)

Because the mempool is public:

1. The attacker sees the user's tx with the intended active tick, e.g., `tick = 1000`.
2. The attacker **front-runs** and creates the same pool first with **a different tick**, e.g., `tick = 500`.
3. The user's transaction executes afterward:

   * Pool **already exists**, so router **does not create it**
   * Router simply **adds user liquidity** at the attacker's initial tick
4. If the user used `isDelta = true` (very common in these router patterns):

   * Their bins shift relative to the attacker's tick
   * They unknowingly deposit liquidity at a **manipulated price**
5. Attacker immediately swaps against the user's liquidity and extracts value.

### Concrete Example (Alice & Mallory)

*Alice:*
Wants to create a pool at **1 A = 1 B**, so she sets `activeTick = 1000` and calls `getOrCreatePoolAndAddLiquidity()` with `isDelta = true`.

*Mallory (attacker):*
Front-runs and creates the pool with `activeTick = 500` (corresponds to **1 A = 2 B**, where A is overpriced).

*Result:*
Alice's bins are placed relative to **500** not **1000**, so she deposits liquidity at a **worse price**.
Mallory immediately swaps at the artificially advantageous rate and profits.
Alice loses part of her deposited tokens.

> **Analogy**: Alice is trying to open a market stall and post her own price list.
> Mallory rushes ahead, opens the stall *first*, posts a **fake price list**, and when Alice puts goods on the shelves, she is forced to follow Mallory's prices… and Mallory buys them cheap.

## Code Reference

### Router function allowing the vulnerability

```solidity
function getOrCreatePoolAndAddLiquidity(
    PoolParams calldata poolParams,
    uint256 tokenId,
    IPool.AddLiquidityParams[] calldata addParams,
    uint256 minTokenAAmount,
    uint256 minTokenBAmount,
    uint256 deadline
) external payable checkDeadline(deadline)
  returns (uint256 receivingTokenId, uint256 tokenAAmount, uint256 tokenBAmount, IPool.BinDelta[] memory binDeltas)
{
    IPool pool = getOrCreatePool(poolParams); // <-- can be front-run
    return addLiquidity(pool, tokenId, addParams, minTokenAAmount, minTokenBAmount);
}
```

### Root Cause

* Pool creation is **not guaranteed** even if the user intends to create it.
* No check such as:

  ```solidity
  require(pool.didNotExistBefore, "Pool already created");
  ```

* No validation that:

  ```solidity
  pool.activeTick() == poolParams.activeTick;
  ```

* `isDelta = true` makes user liquidity **depend on the current active tick**, which the attacker can set.

## Why This Matters

* Creates **unexpected price exposure** for first LPs.
* Allows extraction of value purely from **transaction ordering**, no economic cost to attacker.
* The optional safety (`isDelta = false`) is **not obvious**, and most first LPs naturally use relative bins expecting their tick to be the one used.
* A common front-running vector typical in router convenience functions.

## Concrete Walkthrough of the Exploit

1. **Pool doesn't exist**.
2. User calls router with:

   * `poolParams.activeTick = T_user`
   * `isDelta = true`
3. Attacker sees the tx → creates the pool at tick `T_attack`.
4. User's tx: pool exists → addLiquidity() uses **T_attack**.
5. User's liquidity is placed at **wrong bins**.
6. Attacker performs a swap to extract arbitrage → **user loses funds**.

## Recommended Mitigation

### 1. Split actions into two transactions (best UX-safe fix)

* Tx1: `createPool(poolParams)`
* Tx2: `addLiquidity(addParams)`
  This removes the front-running vector entirely.

### 2. Or enforce creation expectations

Add a `bool expectNewPool` param:

```solidity
if (expectNewPool && pool.exists()) revert("Pool already exists");
```

### 3. Or enforce tick correctness

Before adding liquidity:

```solidity
require(pool.activeTick() == poolParams.activeTick, "Unexpected activeTick");
```

### 4. User-side mitigation (the sponsor's note)

* Set **`isDelta = false`** when calling this function for a pool you believe does not exist.
  This ensures the attacker cannot shift your bins relative to the active tick.

### 5. Optional: block single-tx initialization + LP addition entirely

Disallow the combined action by design since it's inherently raceable.

## Pattern Recognition Notes

* **Front-Run Initialization**: Any time pool creation + liquidity add occur in one tx, MEV bots can create the pool first at a manipulated price.
* **Assumption Mismatch**: User assumes they control initial tick; router logic allows external influence.
* **State-Dependent Liquidity Placement**: `isDelta = true` means positions depend on the current tick — making the system sensitive to attacker-manipulated state.
* **Router Convenience Hazards**: "get or create" patterns often expose silent edge cases where "get" is taken even though caller expected "create".

### Quick Recall (TL;DR)

* **Bug**: Pool creation + LP deposit in one tx can be front-run.
* **Impact**: LP deposits at attacker-controlled initial tick → loses value.
* **Why Medium**: Optional protection exists (`isDelta=false`), but not obvious; realistic user error scenario.
* **Fix**: Split the actions, or enforce tick expectations, or require user to explicitly acknowledge pool existence.
