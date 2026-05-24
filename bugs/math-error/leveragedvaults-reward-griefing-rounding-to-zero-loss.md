# Loss of Rewards via High-Frequency Griefing & Rounding-to-Zero on L2

* **Severity**: High
* **Source**: [Sherlock](https://github.com/sherlock-audit/2024-06-leveraged-vaults-judging/issues/61)
* **Affected Contract**: [VaultRewarderLib.sol](https://github.com/sherlock-audit/2024-06-leveraged-vaults/blob/main/leveraged-vaults-private/contracts/vaults/common/VaultRewarderLib.sol)
* **Vulnerability Type**: Precision Loss / Griefing Attack / Reward Accounting Failure / Rounding-to-Zero

## Summary

The reward distribution system relies on calculating:

```solidity
rewardPerVaultShare = tokensClaimed / totalVaultShares
```

using integer arithmetic.

Because reward claiming and reward accumulation functions are permissionless, malicious users on low-fee L2 environments such as Arbitrum can repeatedly trigger reward updates every block. This causes only tiny amounts of rewards to accumulate per update.

When the claimed/emitted rewards become too small relative to the total vault shares, integer division rounds the computed reward-per-share value down to zero.

As a result:

* rewards are successfully claimed or emitted,
* but vault accounting records `0` additional rewards for users,
* causing rewards to become effectively lost/stuck inside the system.

The issue becomes increasingly severe as vault TVL grows because larger total vault shares make rounding-to-zero easier to trigger.

## A Better Explanation (With Simplified Example)

### Intended Behavior

The vault distributes rewards proportionally across vault share holders.

Simplified flow:

1. Vault claims or emits reward tokens.
2. Protocol calculates how much reward belongs to each vault share:

```solidity
rewardPerShare = rewards / totalShares
```

3. `accumulatedRewardPerVaultShare` increases.
4. Users later claim rewards based on this accumulated value.

The system assumes reward updates happen infrequently enough that meaningful reward amounts accumulate before division occurs.

### What Actually Happens (Bug)

On L2 chains like Arbitrum:

* transactions are extremely cheap,
* anyone can repeatedly trigger reward accumulation functions every block.

This creates a griefing scenario where:

* each update processes only a microscopic reward amount,
* division against huge vault share supply rounds down to zero.

Example:

* `tokensClaimed = 9,999,999`
* `totalVaultSharesBefore = 10,000,000e8`

Calculation:

```solidity
(tokensClaimed * 1e8) / totalVaultSharesBefore
```

Effectively becomes:

```text
9999999 / 10000000 = 0.9999999
```

But Solidity integer math truncates decimals:

```text
0
```

So:

```solidity
accumulatedRewardPerVaultShare += 0;
```

The rewards were claimed successfully, but users receive no accounting credit for them.

### Why This Matters

* Users lose reward yield.
* Vault APY can be artificially suppressed.
* Attackers can continuously grief the reward system at low cost.
* Larger TVLs worsen the issue because reward-per-share becomes even smaller.
* Protocol growth ironically increases exploitability.

This is especially dangerous on L2 chains where attackers can cheaply automate block-by-block griefing.

## Concrete Walkthrough (Alice & Mallory)

### Setup

Vault state:

```text
totalVaultSharesBefore = 10,000,000e8
```

Reward updates are permissionless.

### Normal Expected Scenario

Suppose rewards accumulate for 1 hour before being claimed:

```text
tokensClaimed = 1000 tokens
```

Calculation:

```text
1000 / 10,000,000 = meaningful non-zero value
```

Users properly receive rewards.

### Mallory Griefing Attack

Mallory repeatedly calls:

```solidity
claimRewardTokens()
```

every block.

Now only tiny rewards accumulate between updates:

```text
tokensClaimed = extremely small
```

Eventually:

```solidity
(tokensClaimed * 1e8) / totalVaultSharesBefore
```

rounds to:

```text
0
```

So accounting becomes:

```solidity
accumulatedRewardPerVaultShare += 0;
```

Rewards were technically claimed by the vault but are never distributed to users.

### Result

* Reward accounting silently loses precision.
* Users lose rewards over time.
* Mallory continuously griefs the protocol at very low cost.
* Larger TVL increases the frequency of rounding-to-zero events.

> **Analogy**: Imagine distributing crumbs across millions of people. If the accountant rounds fractions down, eventually every crumb distribution becomes "0 per person," even though food was actually delivered.

## Vulnerable Code Reference

### 1) Tiny Claimed Rewards Can Round to Zero

```solidity
function _accumulateSecondaryRewardViaClaim(
    uint256 index,
    VaultRewardState memory state,
    uint256 tokensClaimed,
    uint256 totalVaultSharesBefore
) private {
    if (tokensClaimed == 0) return;

    state.accumulatedRewardPerVaultShare += (
        (tokensClaimed * uint256(Constants.INTERNAL_TOKEN_PRECISION))
            / totalVaultSharesBefore
    ).toUint128();

    VaultStorage.getVaultRewardState()[index] = state;
}
```

### Root Issue

If:

```text
tokensClaimed << totalVaultSharesBefore
```

then integer division truncates to zero.

### 2) Permissionless Reward Claiming Enables Griefing

```solidity
_executeClaim(rewardPool);

rewardPool.lastClaimTimestamp = uint32(block.timestamp);
```

Attackers can repeatedly trigger claims every block on low-cost L2 environments.

### 3) Emission-Based Rewards Suffer the Same Issue

```solidity
additionalIncentiveAccumulatedPerVaultShare =
    (timeSinceLastAccumulation
        * uint256(Constants.INTERNAL_TOKEN_PRECISION)
        * state.emissionRatePerYear)
    / (Constants.YEAR * totalVaultSharesBefore);
```

## Root Issue

Repeated updates every block make:

```text
timeSinceLastAccumulation
```

extremely small.

This again causes division rounding-to-zero.

### Instance 1 — Claimed Reward Loss

The vault claims rewards from external reward pools.

```solidity
tokensClaimed =
balanceAfter - balancesBefore[i];
```

Then distributes rewards per vault share.

Because attackers can force extremely frequent claims:

* `tokensClaimed` becomes tiny,
* division rounds to zero,
* rewards become undistributable.

### Instance 2 — Emission Reward Loss

The vault also supports streaming rewards over time.

Rewards depend on:

```solidity
timeSinceLastAccumulation
```

Attackers repeatedly trigger accumulation every block:

```text
timeSinceLastAccumulation ≈ 1 second
```

This produces microscopic reward increments which again round down to zero.

## Recommended Mitigation

### 1) Restrict Reward Claiming Frequency (Primary Fix)

Make reward claiming permissioned or rate-limited:

```solidity
onlyWhitelistedKeeper
```

or:

```solidity
require(
    block.timestamp >= lastClaim + MIN_CLAIM_INTERVAL
);
```

This ensures meaningful rewards accumulate before distribution.

### 2) Introduce Residual Accounting

Store remainder values instead of discarding precision:

```solidity
unallocatedRewards += remainder;
```

Then include residuals in future calculations.

This prevents permanent reward loss from truncation.

### 3) Increase Internal Precision

Use larger precision constants to reduce truncation risk:

```solidity
1e18 instead of 1e8
```

though this alone may not completely solve the issue.

### 4) Minimum Claim Thresholds

Only process reward updates once rewards exceed a minimum amount:

```solidity
if (tokensClaimed < MIN_REWARD_THRESHOLD) return;
```

This prevents microscopic updates.

### 5) Add Stress Tests for Large TVL

Add invariant/property tests ensuring:

* rewards never silently disappear,
* high-frequency updates cannot cause accounting drift,
* large TVLs do not break reward distribution.

## Pattern Recognition Notes

* **Rounding-to-Zero Bugs**: Integer division truncation becomes dangerous when small numerators are divided by massive denominators.
* **High-Frequency State Manipulation**: Permissionless update functions are vulnerable when attackers can force extremely frequent execution.
* **L2 Economic Assumption Failure**: Logic that is "safe on mainnet due to gas cost" may become exploitable on low-fee L2s.
* **Precision Loss in Reward Systems**: Reward-per-share accounting is highly sensitive to arithmetic precision and update frequency.
* **TVL-Scaling Vulnerabilities**: Bugs involving division by total shares often worsen as protocols grow larger.
* **Silent Economic Failures**: No revert occurs; rewards simply vanish from accounting invisibly over time.

## Quick Recall (TL;DR)

* **Bug**: Tiny reward updates divided by huge vault share supply round down to zero.
* **Attack**: Malicious users repeatedly trigger reward updates every block on cheap L2s.
* **Impact**: Rewards are claimed/emitted but never distributed to users.
* **Why L2 Matters**: Low fees make continuous griefing economically practical.
* **Fix**: Restrict update frequency, preserve residual precision, and avoid tiny reward distributions.
