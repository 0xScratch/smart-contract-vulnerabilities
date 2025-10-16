# Denial of Service via Rounding Edge Case in `WstEth.withdraw`

- **Severity**: Medium
- **Source**: [Code4rena](https://github.com/code-423n4/2023-05-asymmetry-mitigation-findings/issues/70) / [One Bug Per Day](https://www.onebugperday.com/v1/89)
- **Affected Contract**: [WstEth.sol](https://github.com/asymmetryfinance/smart-contracts/blob/ec582149ae9733eed6b11089cd92ca72ee5425d6/contracts/SafEth/derivatives/WstEth.sol#L71-L72)
- **Vulnerability Type**: Denial of Service (DoS) - Edge-Case Precision Error

## Original Bug Description

>## Description
>
>The changes in [`WstEth.withdraw()`](https://github.com/asymmetryfinance/smart-contracts/blob/ec582149ae9733eed6b11089cd92ca72ee5425d6/contracts/SafEth/derivatives/WstEth.sol#L69C33-L84) has introduced a new issue exactly parallel to the one present in `SfrxEth.withdraw()` which was reported in [M-02: sFrxEth may revert on redeeming non-zero amount](https://github.com/code-423n4/2023-03-asymmetry-findings/issues/1049), i.e. `WstEth.withdraw(_amount)` may revert when `_amount > 0`. For why this is an issue please refer to M-02. The mitigation of M-02 was to enable/disable derivatives. See my mitigation review of M-02 for how that issue is not resolved and why I think the mitigation may be insufficient. What is said there equally apply, mutatis mutandis, to this new issue.
>
>## Proof of Concept
>
>`WstEth.withdraw()` now begins
>
>```solidity
>uint256 stEthAmount = IWStETH(WST_ETH).unwrap(_amount);
>require(stEthAmount > 0, "No stETH to unwrap");
>```
>
>We therefore have the same problem as in M-02 if `IWStETH(WST_ETH).unwrap(1) == 0`.
>
>`WstEth.unwrap()` is
>
>```solidity
>function unwrap(uint256 _wstETHAmount) external returns (uint256) {
>    require(_wstETHAmount > 0, "wstETH: zero amount unwrap not allowed");
>    uint256 stETHAmount = stETH.getPooledEthByShares(_wstETHAmount);
>    _burn(msg.sender, _wstETHAmount);
>    stETH.transfer(msg.sender, stETHAmount);
>    return stETHAmount;
>}
>```
>
>We then ask whether `stETH.getPooledEthByShares(1) == 0`. [`StETH.getPooledEthByShares()`](https://github.com/lidofinance/lido-dao/blob/df95e563445821988baf9869fde64d86c36be55f/contracts/0.4.24/StETH.sol#L315C14-L324) is:
>
>```solidity
>function getPooledEthByShares(uint256 _sharesAmount) public view returns (uint256) {
>    uint256 totalShares = _getTotalShares();
>    if (totalShares == 0) {
>        return 0;
>    } else {
>        return _sharesAmount
>            .mul(_getTotalPooledEther())
>            .div(totalShares);
>    }
>}
>```
>
>So just like in M-02, if `_getTotalPooledEther() < totalShares` then `IWStETH(WST_ETH).unwrap(1) == 0` and `WstEth.withdraw(1)` reverts.
>
>## Recommended Mitigation Steps
>
>Replace `require(stEthAmount > 0, "No stETH to unwrap");` with `if (stEthAmount > 0) return;`

## Summary

A rounding-to-zero edge case in Lido's wstETH conversion logic can cause `WstEth.withdraw(_amount)` to revert when `_amount` is as low as 1 wei and the global Lido pooled ETH is less than total wstETH shares. Because `safETH.unstake()` batches proportional withdrawals across all derivatives, a failure in this single derivative blocks the entire unstaking operation for all users.

## A Better Explanation (With Simplified Example)

To understand this bug, let's break it down in simple steps — starting from what the protocol is doing, how wstETH math works under the hood, and when things can go wrong.

### 🧠 What's Happening Under the Hood

`safETH` is a protocol that diversifies your ETH among several **liquid staking tokens** (also called *derivatives*), such as:

- Lido's **wstETH**
- Rocket Pool's **rETH**
- Frax's **sfrxETH**

This means:

- When you stake ETH into `safETH`, the protocol **splits** your ETH into different pools.
- When you **unstake**, it needs to **pull back** ETH from each derivative according to its remaining balance.

So if a user is withdrawing 50% of safETH supply, the protocol will try to withdraw 50% of whatever's in each derivative.

Now: suppose wstETH has only **2 wei** left — the unstake function will attempt to withdraw **1 wei** from it.

That's where things go wrong.

### 🧮 What Does `WstEth.withdraw()` Actually Do?

Inside the wstETH wrapper, the `withdraw(_amount)` function tries to **unwrap** `wstETH` into `stETH` — Lido's main staking token.

The line that begins unwrapping:

```solidity
uint256 stEthAmount = IWStETH(WST_ETH).unwrap(_amount);
require(stEthAmount > 0, "No stETH to unwrap");
```

Let's unpack what's behind `unwrap()`.

### 🧪 How wstETH Unwrap Works (Zooming In)

When you call `unwrap(1)`, the unwrap function internally converts shares using this formula:

```solidity
amount = shares × (totalPooledEther / totalShares)
```

- `shares` = amount of wstETH (in this case, just **1 wei**)
- `totalPooledEther` = the total ETH sitting in the Lido staking validators
- `totalShares` = all shares issued so far for people staking in Lido

So you get back:

```solidity
1 × (totalPooledEther / totalShares)
```

### 🔥 When Can This Be a Problem?

Here's the key:

**If `totalPooledEther < totalShares`, then the ratio becomes less than 1.**

And since Solidity operates using **integer division without decimals**, the calculation will round **down to 0**.

This makes `unwrap(1) = 0`, and that triggers the `require(...)` check:

```solidity
require(stEthAmount > 0, "No stETH to unwrap"); // ❌ Reverts
```

Once any one `withdraw()` fails, the `safETH.unstake()` call **aborts for everyone** — nobody can withdraw even if they never staked in wstETH.

### 🔍 Why Could `totalPooledEther < totalShares`?

Some misunderstandings claim this can "never happen." That's not true.

Here are a few *real conditions* where this can happen:

| Case | Why It Happens |
| :-- | :-- |
| **Slashing** | If Lido validators get punished (slashed), total ETH backing stETH (i.e. totalPooledEther) shrinks, but shares stay the same. |
| **Rounding during minting** | Someone deposits ETH and gets more shares than ETH (due to rounding/truncation). |
| **Reward delay** | Lido may delay updating totalPooledEther until validators report. So shares may pre-exist ETH. |
| **Operational bug in Lido or ETH withdrawal logic** | ETH burned, not yet accounted — imbalance is created indirectly. |

So yes — even if rare, this is **absolutely possible** in real-world DeFi.

### 🧩 How `_amount == 1` Happens in the First Place?

This part is also easily misunderstood. You might ask:
> "Why would anyone try to unwrap just 1 wei?"

Here's why:

`safETH.unstake()` calls `withdraw()` for *every* derivative — regardless of how much is left in that derivative.

Let's say:

- safETH has 1,000 ETH staked
- 1 ETH is remaining in the wstETH derivative (maybe its weight was set to 0 before)
- A user tries to unstake **half** the total safETH supply

Then:

```solidity
→ 0.5 × 1 ETH from wstETH = 0.5 ETH → call withdraw(0.5 ETH)
```

This results in the protocol calling:

```solidity
unwrap(1) // or whatever amount equals the 0.5 derivative balance
```

And **if** wstETH's underlying exchange rate makes that unwrap equal to **0 stETH**, then the call fails.

Boom, there goes everyone's withdrawal 😬

✓ Even better? You didn't even stake in wstETH!
But `safETH` **still calls** wstETH.withdraw() for you, and it fails on a dust-value.

### 🍕 Real Life Analogy: The Shared Pizza Problem

Imagine this:

- You and 10 friends put money into a shared pizza fund.
- The fund buys pizza from 3 restaurants.
- Months later, you're hungry — you want to eat **half** your share.

But the system is set up so that **you must take half of what remains from every restaurant**...

One restaurant only has **1 pepperoni crumb** left.

Now the pizza cutter (smart contract math) tries to give you **half** of that crumb — but it's so tiny, the knife can't slice it.

Instead of skipping the crumb, the whole order fails and nobody gets any pizza.

### ✅ Final Summary

| Step | What Happens |
| :-- | :-- |
| 1 | User calls `safETH.unstake()` |
| 2 | Protocol calculates a proportional withdrawal from **all derivatives**, including wstETH |
| 3 | wstETH receives a withdrawal request with a very small `_amount` (like `1 wei`) |
| 4 | `unwrap(1)` ≈ 1 × (`totalPooledEther` / `totalShares`) = **0** due to rounding |
| 5 | `require(stEthAmount > 0)` fails, causing **revert** |
| 6 | Whole unstake fails — users are locked out, even if they never interacted with wstETH |

## Vulnerable Code Reference

```solidity
function withdraw(uint256 _amount) external onlyOwner {
    underlyingBalance = underlyingBalance - _amount;
    uint256 stEthAmount = IWStETH(WST_ETH).unwrap(_amount);
    require(stEthAmount > 0, "No stETH to unwrap");
    IERC20(STETH_TOKEN).approve(LIDO_CRV_POOL, stEthAmount);
    ...
}
```

## Impact

- **DoS**: Users cannot unstake any ETH because a single wei in one derivative can block the entire batch operation.
- **Protocol-Wide Liveness Risk**: Even users without wstETH exposure are affected.
- **Reputation Damage**: A live "can't withdraw" event undermines trust and may force emergency admin interventions.

## Mitigation

Replace the fatal `require` with a soft-fail guard:

```diff
- require(stEthAmount > 0, "No stETH to unwrap");
+ if (stEthAmount == 0) {
+     return; // Skip zero-value unwrap without reverting
+ }
```

This early return allows other derivatives to complete withdrawals while leaving negligible dust untouched.

## Pattern Recognition Notes

- **Denial of Service via Batch Reverts**: Iterating through multiple adapters where a single revert halts the entire process.
- **Edge-Case Precision Errors**: Integer division rounding down when assets < shares.
- **Dust Management**: Always anticipate "1 wei" scenarios; skip or burn dust rather than revert.
- **Graceful Degradation**: Favor `if (x == 0) return;` over `require(x > 0)` in multi-step, multi-adapter functions.
- **Circuit Breakers**: Provide means to disable or skip failing adapters without halting core user flows.
