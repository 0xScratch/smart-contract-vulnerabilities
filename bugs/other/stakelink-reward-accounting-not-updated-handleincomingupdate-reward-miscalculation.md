# Incorrect Reward Distribution Due to Missing Reward Update in `handleIncomingUpdate`

* **Severity**: High
* **Source**: [CodeHawks](https://codehawks.cyfrin.io/c/2023-12-stake-link/s/162)
* **Affected Contract**: [SDLPoolPrimary.sol](https://github.com/Cyfrin/2023-12-stake-link/blob/main/contracts/core/sdlPool/SDLPoolPrimary.sol#L231)
* **Vulnerability Type**: Reward Accounting Error / State Update Order

## Summary

The `handleIncomingUpdate()` function in `SDLPoolPrimary` processes cross-chain updates that modify pool reward state. However, the function **updates reward-related variables without first updating the accumulated reward accounting**.

In reward distribution systems that use the **reward-per-token accounting model**, rewards must be **snapshotted before any stake or reward changes occur**. Because `handleIncomingUpdate()` fails to call the reward update logic prior to modifying state, new rewards are calculated using **incorrect stake snapshots**, leading to **misallocated rewards**.

As a result, users who change their stake around the time of the update may receive **more rewards than they should**, while earlier stakers may receive **less**, breaking the fairness and accuracy of the reward distribution mechanism.

## A Better Explanation (With Simplified Example)

### Intended Behavior

Reward distribution typically follows a **cumulative reward-per-token model**.

1. Rewards arrive to the pool.
2. The protocol updates a global variable:

    ```solidity
    rewardPerToken += rewards / totalStaked
    ```

3. Each user earns rewards based on their stake:

    ```solidity
    earned = userStake * rewardPerToken - userRewardDebt
    ```

Before any change in stake or rewards, the contract should first **update the global reward accounting** to ensure all past rewards are distributed according to the correct stake snapshot.

### What Actually Happens (Bug)

`handleIncomingUpdate()` applies reward or stake changes **without first updating accumulated rewards**.

Because of this:

* The protocol mixes **old rewards with new stake balances**.
* Rewards that should belong to earlier stakers get partially redistributed to users who joined later.

This leads to **incorrect reward calculations and unfair reward allocation**.

### Why This Matters

* Late stakers can capture rewards meant for earlier participants.
* Existing stakers receive **diluted rewards**.
* Reward accounting becomes inconsistent across cross-chain updates.
* The bug can potentially be exploited by **timing deposits around reward updates**.

While the issue does not directly allow theft of funds from the contract, it **breaks the economic correctness of reward distribution**, which is critical for staking systems.

## Concrete Walkthrough (Alice & Bob)

### Initial State

```yaml
Alice stake = 100 SDL
Bob stake   = 100 SDL
Total stake = 200
```

A reward update arrives:

```yaml
New rewards = 100 tokens
```

## Correct Behavior

Before any stake changes occur, rewards should be updated:

```yaml
rewardPerToken += 100 / 200
                = 0.5
```

Rewards earned:

```yaml
Alice = 100 * 0.5 = 50
Bob   = 100 * 0.5 = 50
```

## Buggy Behavior

Suppose `handleIncomingUpdate()` does **not update rewards first**.

### Step 1 — Reward arrives

```yaml
+100 rewards
```

But `rewardPerToken` is **not updated yet**.

### Step 2 — Bob increases stake

Bob deposits:

```yaml
+100 SDL
```

New state:

```yaml
Alice = 100
Bob   = 200
Total = 300
```

### Step 3 — Rewards calculated

```yaml
rewardPerToken += 100 / 300
                = 0.333
```

Rewards now become:

```yaml
Alice = 100 * 0.333 = 33
Bob   = 200 * 0.333 = 66
```

### Result

Correct rewards:

```yaml
Alice = 50
Bob   = 50
```

Actual rewards:

```yaml
Alice = 33
Bob   = 66
```

Bob receives **extra rewards** simply because the reward accounting was updated after his deposit.

> **Analogy:**
> Imagine a pizza meant to be shared by two people. Before slicing it, a third person joins the table. If the pizza is divided after the third person arrives, the original two people receive smaller slices even though the pizza was meant for them.

---

## Vulnerable Code Reference

### 1) Reward update missing before state changes

The function handling incoming updates modifies reward state without updating the reward snapshot.

```solidity
function handleIncomingUpdate(...) external {
    totalRewards += incomingRewards;
    totalStaked += incomingStake;
}
```

The missing step is the reward accounting update before these modifications.

### 2) Correct reward update pattern

Reward systems should follow this order:

```solidity
function handleIncomingUpdate(...) external {
    _updateRewards(); // snapshot rewards before modifying stake or rewards
    applyIncomingUpdate();
}
```

Without this step, the reward calculations use incorrect stake snapshots.

## Recommended Mitigation

### 1. Update rewards before state changes (Primary Fix)

Ensure that reward accounting is updated before modifying stake or reward variables.

```solidity
function handleIncomingUpdate(...) external {
    _updateRewards();
    applyIncomingUpdate();
}
```

### 2. Enforce consistent reward update patterns

Any function that modifies:

* total stake
* reward balance
* reward rate
* cross-chain reward state

must first call the reward update function.

### 3. Add invariant tests

Testing should ensure:

* Deposits during reward updates do not affect past reward distribution.
* Reward calculations remain consistent regardless of deposit timing.

### 4. Audit cross-chain update paths

Cross-chain handlers such as `handleIncomingUpdate()` should be reviewed carefully because they often bypass typical deposit/withdraw flows where reward updates normally occur.

## Pattern Recognition Notes

### **State Update Order Bugs**

Many reward distribution vulnerabilities occur because **state changes happen before reward accounting updates**.

Typical rule:

> **Always update rewards before modifying stake or reward variables.**

### **Reward Snapshot Dependence**

Reward systems depend on **correct historical snapshots of total stake**. If snapshots are taken at the wrong time, reward distribution becomes inaccurate.

### **Cross-Chain State Sync Risks**

Functions triggered by cross-chain messages may bypass common internal safeguards (like reward updates), making them high-risk locations for accounting bugs.

### **Deposit Timing Exploits**

Whenever reward accounting relies on `rewardPerToken`, check if users can:

* deposit
* withdraw
* rebalance

before the reward update occurs.

If yes, **reward capture attacks may be possible**.

## Quick Recall (TL;DR)

* **Bug**: `handleIncomingUpdate()` modifies reward state without updating accumulated rewards first.
* **Impact**: Rewards are distributed using incorrect stake snapshots, allowing late stakers to receive excess rewards.
* **Root Cause**: Incorrect order of operations in reward accounting.
* **Fix**: Call the reward update function before modifying stake or reward variables.
