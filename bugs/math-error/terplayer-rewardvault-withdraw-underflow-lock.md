# Withdrawal Underflow via Self-Delegation and Ceiling Division in BVT Reward Vault

* **Severity**: Critical
* **Source**: [Shieldify Audit Report](https://github.com/shieldify-security/audits-portfolio-md/blob/main/Terplayer-BVT-Staking&Distribution-Security-Review.md#c-01-withdrawal-calculation-causes-underflow-locking-all-user-funds)
* **Affected Contract**: [BvtRewardVault.sol](https://github.com/batoshidao/berabtc-vault-token/blob/c68f412b3c7dfd99d3f6302a42bdf772ededb2a3/src/BvtRewardVault.sol#L155) (Somehow this link led to nowhere, don't know why?!)
* **Vulnerability Type**: Arithmetic Underflow / Double Counting / DoS (withdraw freeze)

## Summary

The `withdraw` function of `BvtRewardVault` incorrectly **includes the withdrawing user in their own delegation loop**. This causes their stake to be **double-counted**. On top of that, the calculation of delegated portions uses **ceiling division**, which consistently rounds withdrawals *upward*.

Together, these issues inflate `totalDelegatedAmount` beyond the actual withdrawal `amount`. When the contract attempts to compute `remainingAmount = amount - totalDelegatedAmount`, it underflows and reverts.

**Impact**: No user can withdraw — all funds in the vault become permanently locked.

## A Better Explanation (With Simplified Example)

### Intended Behavior

1. User requests to withdraw `amount`.
2. Contract loops over all delegators (other users who delegated stake to this user) and calculates how much to take from each.
3. After processing delegations, it calculates the **remaining portion** of the withdrawal to be covered by the user's **own stake**.
4. User receives the total `amount`.

### What Actually Happens (Bug)

**Issue 1: Self-Delegation Double Counting**:

* The withdrawal loop iterates over `users[]`, which includes the caller (`msg.sender`).
* This causes the user's own stake to be incorrectly treated as a delegated stake.
* As a result, part of the withdrawal is already "charged" against their own funds during delegation.

**Issue 2: Ceiling Division Overestimation**:

* Delegated withdrawal portions are computed with ceiling division:

  ```solidity
  withdrawAmount = (delegatedAmount * amount + stakes[msg.sender] - 1) / stakes[msg.sender];
  ```

* This formula rounds fractional results **up** instead of down.
* When multiple delegations exist, each is slightly over-allocated, inflating the total.

**Combined Effect**:
`totalDelegatedAmount > amount` → then `remainingAmount = amount - totalDelegatedAmount` underflows and reverts.

### Why This Matters

* **Global Freeze**: Any withdrawal attempt fails. Once triggered, no user can exit their funds.
* **Permanent Locking**: Since this is a core math bug, it cannot self-heal. Funds remain trapped until contract upgrade or migration.
* **Critical Risk**: Loss of access to all staked funds undermines the entire vault's functionality.

### Concrete Walkthrough (Alice & Bob)

* **Setup**: Alice has `100 BVT` staked. Bob has delegated `20 BVT`.
* **Alice withdraws 50 BVT**.

Loop:

* Iteration 1 (Alice included as a "delegator"):

  * Withdraw portion from herself counted prematurely.
  * Say this adds **30** to `totalDelegatedAmount`.
* Iteration 2 (Bob as delegator):

  * Formula with ceiling division calculates **11** instead of 10.
  * Now `totalDelegatedAmount = 41`.

Final step:

```solidity
remainingAmount = 50 - 41 = 9   ✅ (still positive in this toy example)
```

But with more delegators or larger rounding errors:

```solidity
remainingAmount = 50 - 60;  // underflow → revert
```

Alice's withdrawal fails, and so will everyone else's.

> **Analogy**: Imagine you're splitting a \$50 bill between friends. You accidentally count yourself as a friend who owes money back to you, and you also round up each person's share to the next dollar. Very quickly, you "owe" more than \$50 total, leaving nothing for yourself — the math collapses.

## Vulnerable Code Reference

### 1) Inclusion of `msg.sender` in delegation loop

```solidity
for (uint256 i = 0; i < users.length; i++) {
    address user = users[i];
    uint256 delegatedAmount = delegatedStakes[msg.sender][user];
    if (delegatedAmount > 0) {
        uint256 withdrawAmount =
            (delegatedAmount * amount + stakes[msg.sender] - 1) / stakes[msg.sender];
        totalDelegatedAmount += withdrawAmount;
        _delegateWithdraw(msg.sender, user, withdrawAmount);
    }
}
```

### 2) Remaining amount subtraction prone to underflow

```solidity
uint256 remainingAmount = amount - totalDelegatedAmount; // underflow risk
```

## Recommended Mitigation

1. **Exclude self from delegation loop**

   ```solidity
   if (user == msg.sender) continue;
   ```

2. **Use floor division for delegated amounts**

   ```solidity
   uint256 withdrawAmount = (delegatedAmount * amount) / stakes[msg.sender];
   ```

   Handle the remainder separately if exactness is required.

3. **Add invariant checks**
   Before subtraction, enforce:

   ```solidity
   require(totalDelegatedAmount <= amount, "Delegated exceeds amount");
   ```

4. **Add dedicated tests**
   Unit tests where a user withdraws with multiple delegators should confirm no over-allocation or underflow occurs.

## Pattern Recognition Notes

* **Self-Inclusion Bugs**: Iterating over all users without filtering can create "ghost" cases where an entity double-counts itself.
* **Ceiling Division Pitfalls**: Rounding up in proportional calculations causes silent inflation when repeated across many entries.
* **Arithmetic Safety**: Even with Solidity 0.8 checks, unchecked over/underflows can still cause DoS by reverting all calls.
* **Withdraw Path Fragility**: If a withdraw function is not resilient to rounding errors, it risks bricking the entire system.
* **Best Practice**: Keep delegation logic and self-stake handling strictly separate, and prefer floor division + remainder accounting for proportional math.

### Quick Recall (TL;DR)

* **Bug**: Self included in delegation loop + ceiling division inflates delegated withdrawals.
* **Impact**: `totalDelegatedAmount > amount` → underflow → all withdrawals revert → **funds locked forever**.
* **Fix**: Exclude self in loop, use floor division, and enforce upper-bound checks.
