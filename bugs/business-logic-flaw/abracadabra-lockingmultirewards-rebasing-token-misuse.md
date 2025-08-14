# Improper Handling of Rebasing Tokens in Lending/Borrowing Logic

* **Severity**: Medium
* **Source**: [Code4rena](https://github.com/code-423n4/2024-03-abracadabra-money-findings/issues/48) / [One Bug Per Day](https://www.onebugperday.com/v1/1028)
* **Affected Contract**: [LockingMultiRewards.sol](https://github.com/code-423n4/2024-03-abracadabra-money/blob/1f4693fdbf33e9ad28132643e2d6f7635834c6c6/src/staking/LockingMultiRewards.sol)
* **Vulnerability Type**: Business Logic / Asset Type Assumption

## Original Bug Description

>## Lines of code
>
>[https://github.com/code-423n4/2024-03-abracadabra-money/blob/1f4693fdbf33e9ad28132643e2d6f7635834c6c6/src/staking/LockingMultiRewards.sol#L464-L465](https://github.com/code-423n4/2024-03-abracadabra-money/blob/1f4693fdbf33e9ad28132643e2d6f7635834c6c6/src/staking/LockingMultiRewards.sol#L464-L465)
>
>## Vulnerability details
>
>## Impact
>
>Loss of yield if USDB/WETH is used as reward token.
>
>## Proof of Concept
>
>From what I understood from the protocol team, they want to support any ERC20 tokens that are considered safe/well known. I believe USDB/WETH falls into this category.
>
>Both USDB and WETH yield mode are AUTOMATIC by default.    `LockingMultiRewards.sol` uses internal accounting to track all rewards that are accrued for users. For instance, when users stake `amount`, `stakingTokenBalance` will increase by `amount` and user's balance will increase by `amount` accordingly. Rewards that are distributed to users are based on these internal accounting values.
>
>The issue here is that,
>
>1. If USDB/WETH tokens are used as reward tokens, the accrued yield due to automatic rebasing are lost as they cannot be claimed.
>2. If protocol team has not used such tokens as reward tokens yet and becomes aware of this, then it means that they will not be able to use these tokens as reward tokens.
>
>## Tools Used
>
>Manual Review
>
>## Recommended Mitigation Steps
>
>Add the ability to set native RebasingERC20 token to `CLAIMABLE` and implement a way to claim the yields to the staking contract.
>
>## Assessed type
>
>Context

## Summary

The protocol allows the usage of **USDB/WETH** as deposit or reward assets without accounting for **rebasing behavior**.
USDB is a **rebasing token** — its balance in a user's wallet can increase or decrease automatically based on supply adjustments, without any transfer events.

If the protocol's logic assumes **static ERC20 balances**, it can miscalculate accrued rewards, debt positions, or collateral ratios, leading to **unintended over-rewarding, under-repayment, or liquidation bypass**.

## A Better Explanation (With Simplified Example)

### Intended Behavior

* The protocol expects ERC20 tokens to behave like USDC or WETH:

  * Balances only change via `transfer` or `transferFrom`.
  * If Alice deposits 100 tokens, her recorded balance stays 100 until she withdraws or earns rewards.

### What Actually Happens With USDB

* USDB is **rebasing**:
  If Alice holds 100 USDB and the token rebases at +5%, her wallet balance will read 105 USDB — **without any deposit action**.
* If the protocol only tracks raw balances from `balanceOf()` and doesn't account for the scaling factor:

  * Alice could get extra rewards or increased borrowing capacity **for free**.
  * Debt repayments could be miscalculated if rebasing goes negative.

#### Example

1. Alice deposits 100 USDB as collateral.
2. The protocol stores her balance as 100 and uses `balanceOf()` to check it later.
3. USDB rebases to +10% → `balanceOf(Alice)` now returns 110.
4. Protocol thinks Alice **deposited 110** and lets her borrow more, risking bad debt.

### Why This Matters

* Allows users to **inflate collateral value** without actually depositing more assets.
* Can cause **reward farming exploits** if rewards are calculated per token unit without normalizing rebases.
* In lending markets, may lead to **undercollateralized positions** and eventual insolvency.

## Vulnerable Code Reference

While the exact vulnerable code in `LockingMultiRewards.sol` depends on how USDB is integrated, the pattern usually appears in:

```solidity
uint256 balance = stakingToken.balanceOf(address(this));  
// If stakingToken = USDB (rebasing), this balance changes automatically
// without transfers, breaking accounting assumptions.
```

And when calculating rewards:

```solidity
rewards[account] += balanceOf(account) * rewardRate;
// `balanceOf(account)` here will include rebase effects unintentionally
```

## Recommended Mitigation

1. **Use Non-Rebasing Wrappers**

   * Wrap USDB into a non-rebasing ERC20 (like sUSDB) before accepting it.
   * Alternatively, use `staticBalance` tracking that ignores rebase changes.

2. **Track Deposits via Internal Accounting**

   * Maintain an internal ledger that only updates on explicit deposits/withdrawals.
   * Do not rely directly on `ERC20.balanceOf()` for rebasing tokens.

3. **Normalize Token Amounts**

   * For rebasing tokens, fetch the scaling index and convert balances to a fixed representation before calculations.

4. **Asset Whitelisting**

   * If protocol design does not support rebasing tokens, explicitly reject them in deposit functions.

## Pattern Recognition Notes

This vulnerability is an **asset type assumption bug** — the protocol assumes ERC20 tokens have static balances unless explicitly transferred, which fails for rebasing tokens.
Key tips to spot similar issues:

1. **Check Token Type Compatibility**

   * Always verify whether a token is static-supply or rebasing before integrating.
   * Read the token contract to see if balances are derived from a global scaling factor.

2. **Suspicious Reliance on `balanceOf()`**

   * If the protocol reads `balanceOf()` directly for accounting without tracking deposits, rebasing tokens can break it.

3. **Look for Collateral or Reward Calculations Without Normalization**

   * If calculations don't adjust for scaling factors, rebasing can cause over-crediting.

4. **Monitor Non-Transfer Balance Changes**

   * Test with rebasing tokens in simulations; if balances change without transfers, accounting logic is at risk.

5. **Use Wrappers for Complex Assets**

   * Wrapping rebasing tokens into static ERC20 equivalents is a common safe approach (e.g., Aave's aTokens vs. underlying).
