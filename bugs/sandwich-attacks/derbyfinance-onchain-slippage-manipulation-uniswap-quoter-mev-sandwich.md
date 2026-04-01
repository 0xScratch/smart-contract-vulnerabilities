# On-Chain Slippage Manipulation via Uniswap Quoter in Derby Finance

* **Severity**: High
* **Source**: [Sherlock Audit](https://github.com/sherlock-audit/2023-01-derby-judging/issues/310)
* **Affected Contract**:
  * [Vault.sol](https://github.com/sherlock-audit/2023-01-derby/blob/main/derby-yield-optimiser/contracts/Vault.sol)
  * [Swap.sol](https://github.com/sherlock-audit/2023-01-derby/blob/main/derby-yield-optimiser/contracts/libraries/Swap.sol)
  * [QuoterV2.sol](https://github.com/Uniswap/v3-periphery/blob/main/contracts/lens/QuoterV2.sol)
* **Vulnerability Type**: Price Manipulation / MEV Sandwich Exploit / On-Chain Oracle Misuse

## Summary

The Derby Finance vault periodically sells reward tokens (like `COMP`) into underlying assets (e.g. `USDC`) using Uniswap.
However, it **calculates slippage tolerance on-chain** via Uniswap's **Quoter contract**, which actually performs a swap simulation directly on the pool — meaning its price query can be **manipulated by MEV bots**.

An attacker can sandwich the `claimTokens()` call to **temporarily distort the pool price**, trick the slippage calculation, and force the vault to sell reward tokens at **unrealistically low prices**, stealing value from depositors.

## A Better Explanation (With Simplified Example)

### Intended Behavior

1. **Vault deposits** user funds in yield strategies like Aave or Compound.

2. **Rewards (e.g. COMP)** are periodically harvested via `claimTokens()`.

3. These rewards are sold for the vault's base asset (e.g. USDC) using:

   ```solidity
   Swap.swapTokensMulti(
       Swap.SwapInOut(tokenBalance, govToken, vaultCurrency),
       controller.getUniswapParams(),
       false
   );
   ```

4. `swapTokensMulti()` queries Uniswap's `Quoter` to determine the expected output amount:

   ```solidity
   uint256 amountOutMinimum = IQuoter(quoter).quoteExactInput(...);
   ```

5. That value is used as the **minimum acceptable output** during the swap, protecting from slippage.

### What Actually Happens (Bug)

The **Quoter contract** doesn't just return an estimate — it **executes a simulated swap** on-chain to compute the output.
This means the Quoter is **not resistant to manipulation**: anyone can change the pool's spot price right before the call.

Attack flow:

1. Attacker observes a transaction calling `claimTokens()` in the mempool.
2. Attacker front-runs by swapping a small amount in the reward token pool → pushes the price down.
3. When `swapTokensMulti()` calls `quoteExactInput()`, the Quoter now reports a **lower expected output**.
4. Vault uses that as `amountOutMinimum`, effectively saying "I'm okay selling even this cheap."
5. Vault executes the sale — at the manipulated low price.
6. Attacker then **back-runs** the transaction, restoring the pool price and profiting from the cheap tokens they bought.

Result: the vault (and thus all depositors) lose yield value, while the attacker captures the spread.

### Why This Matters

* The vulnerability enables **reward theft** — attacker profits from underpriced swaps.
* Since reward tokens (like COMP) often represent a large share of total yield, this **significantly reduces depositor returns**.
* Exploitability is **recurring** — any time `claimTokens()` or `rebalance()` is called, it can be sandwiched again.
* The root cause is **trusting an on-chain price query** instead of off-chain pre-calculated slippage.

### Concrete Walkthrough (Alice & Mallory)

* **Setup**: Alice deposits USDC → vault allocates to Compound → earns COMP rewards.
* **Mallory (attacker)** monitors mempool for vault calls to `claimTokens()`.
* Just before the claim executes, she sends a **front-running swap** that dumps COMP to USDC → drops the pool price.
* Vault calls the Uniswap `Quoter` and sees a low `amountOutMinimum`.
* Vault executes the sell at that manipulated rate, receiving fewer USDC.
* Immediately after, Mallory **back-runs** to buy back COMP at a lower price and restore the pool's original price.
* Result: Mallory gains USDC from vault's loss.

> **Analogy**: Imagine checking a "live" currency exchange rate right after someone briefly crashes the market. You lock in that bad rate — and when prices bounce back, you've already sold cheap.

## Vulnerable Code Reference

### 1) Slippage calculated via manipulable Quoter

```solidity
uint256 amountOutMinimum = IQuoter(_uniswap.quoter).quoteExactInput(
  abi.encodePacked(_swap.tokenIn, _uniswap.poolFee, WETH, _uniswap.poolFee, _swap.tokenOut),
  _swap.amount
);
```

### 2) Value directly used as `amountOutMinimum` in the Uniswap swap

```solidity
ISwapRouter.ExactInputParams memory params = ISwapRouter.ExactInputParams({
  path: ...,
  recipient: address(this),
  deadline: block.timestamp,
  amountIn: _swap.amount,
  amountOutMinimum: amountOutMinimum
});
```

### 3) `claimTokens()` triggers these swaps whenever reward tokens are harvested

```solidity
if (claim) {
  Swap.swapTokensMulti(...);
}
```

## Recommended Mitigation

1. **Move slippage calculation off-chain**
   Let the caller (e.g. a backend keeper) compute `amountOutMinimum` off-chain based on safe TWAP or oracle pricing.
   Then pass it into the transaction as a parameter.

2. **Restrict public access**
   Make `Vault.claimTokens()` callable only by trusted roles (e.g. keeper, DAO, or vault manager), not public users.

3. **Optional: Add TWAP-based validation**
   Before swapping, verify that the current on-chain price doesn't deviate from TWAP beyond an acceptable threshold.

4. **Add MEV protection**
   Use private transactions (e.g. Flashbots) to prevent front-running.

## Pattern Recognition Notes

* **On-Chain Oracle Trap**: Never use AMM price queries (like Uniswap's Quoter) inside a transaction that performs swaps — both occur in the same block and are manipulable.
* **Sandwich Attack Vector**: Any contract that performs predictable swaps with on-chain pricing or public access is sandwichable.
* **Slippage Delegation Risk**: Slippage should be provided externally by the initiator, not derived internally from manipulable sources.
* **Reward Token Liquidation Sensitivity**: Yield strategies that auto-sell rewards should treat liquidation prices as critical points of attack.

### Quick Recall (TL;DR)

* **Bug**: Slippage tolerance derived from on-chain Quoter → manipulable by MEV bots.
* **Impact**: Reward tokens sold below market; depositors lose yield.
* **Fix**: Calculate slippage off-chain and restrict `claimTokens()` to trusted callers.
