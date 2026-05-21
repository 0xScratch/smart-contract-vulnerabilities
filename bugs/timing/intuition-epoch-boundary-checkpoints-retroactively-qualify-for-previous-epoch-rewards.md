# Epoch-Boundary Checkpoints Retroactively Qualify for Previous Epoch Rewards

* **Severity**: Medium
* **Source**: [Code4rena](https://code4rena.com/audits/2026-03-intuition/submissions/F-26)
* **Affected Contracts**: [CoreEmissionsController.sol](https://github.com/code-423n4/2026-03-intuition/blob/314b7d4/src/protocol/emissions/CoreEmissionsController.sol), [TrustBonding.sol](https://github.com/code-423n4/2026-03-intuition/blob/314b7d4/src/protocol/emissions/TrustBonding.sol)
* **Vulnerability Type**: Reward Accounting / Snapshot Boundary Bug / Temporal Accounting Error

## Summary

`TrustBonding` distributes epoch rewards using veTRUST balance snapshots taken at:

```solidity
_epochTimestampEnd(epoch)
```

However, the controller's epoch arithmetic defines that exact timestamp as:

```text
the FIRST moment of the NEXT epoch
```

At the same time, `VotingEscrow` historical lookups intentionally use an inclusive condition:

```solidity
checkpoint.ts <= snapshotTimestamp
```

This combination creates a boundary flaw:

* a checkpoint created at the **first second of epoch N+1**
* is still included in the reward snapshot for **epoch N**

As a result, attackers can:

* retroactively qualify for old rewards,
* dilute honest participants,
* or even capture an entire epoch's emissions,
  without actually bonding during that epoch.

## A Better Explanation (With Simplified Example)

### Intended Behavior

Suppose epochs work like this:

```text
Epoch 0: 0 → 100
Epoch 1: 100 → 200
```

Rewards for Epoch 0 should ONLY go to users who were bonded before Epoch 1 started.

Meaning:

```text
Joining at timestamp 100
should belong to Epoch 1 only.
```

That is the intended accounting model.

## How Rewards Are Calculated

Reward logic depends entirely on veTRUST snapshots:

```solidity
userBondedBalanceAtEpochEnd(account, epoch)
```

which internally does:

```solidity
_balanceOf(account, _epochTimestampEnd(epoch))
```

Similarly:

```solidity
totalBondedBalanceAtEpochEnd(epoch)
```

uses:

```solidity
_totalSupply(_epochTimestampEnd(epoch))
```

So the protocol asks:

> "What were balances at the END of this epoch?"

## The Critical Problem

The system defines:

```solidity
epochTimestampEnd(0) = 100
```

BUT:

```solidity
currentEpoch()
```

already returns:

```text
1
```

at timestamp `100`.

Because epoch arithmetic is:

```solidity
(timestamp - START) / EPOCH_LENGTH
```

So:

```text
timestamp 100
```

is simultaneously treated as:

```text
END of Epoch 0
AND
START of Epoch 1
```

This overlap becomes dangerous.

## What Actually Happens (Bug)

`VotingEscrow` historical lookups intentionally use inclusive matching:

```solidity
if (checkpoint.ts <= snapshotTime)
```

Meaning:

> "If checkpoint timestamp equals snapshot timestamp exactly, include it."

Now suppose an attacker creates a lock exactly at:

```text
timestamp = epochTimestampEnd(0)
```

which equals:

```text
100
```

The new checkpoint gets:

```text
checkpoint.ts = 100
```

When rewards for Epoch 0 are later computed:

```solidity
_balanceOf(user, 100)
```

the inclusive lookup sees:

```text
100 <= 100
```

TRUE.

So the attacker's brand-new Epoch-1 checkpoint gets counted inside Epoch-0 rewards.

## Why This Matters

This creates:

```text
retroactive reward eligibility
```

Attackers can:

* join AFTER an epoch ended,
* yet still earn that epoch's rewards.

This directly steals emissions from honest participants.

## Concrete Walkthrough (Alice & Mallory)

### Setup

Suppose:

```text
Epoch 0 = [0 → 100)
Epoch 1 = [100 → 200)
```

No users were bonded during Epoch 0.

### Boundary Timestamp Arrives

Time reaches:

```text
t = 100
```

At this exact second:

```text
currentEpoch() == 1
```

meaning Epoch 1 has already started.

### Mallory Joins Late

Mallory calls:

```solidity
create_lock(...)
```

This creates a checkpoint:

```text
checkpoint.ts = 100
```

### Reward Snapshot Executes

The protocol later calculates Epoch-0 rewards using:

```solidity
_balanceOf(mallory, 100)
```

Historical lookup performs:

```solidity
checkpoint.ts <= 100
```

Mallory's checkpoint satisfies:

```text
100 <= 100
```

so it gets included.

### Final Result

Mallory:

* joined at the FIRST second of Epoch 1,
* but receives rewards from Epoch 0.

If nobody else participated during Epoch 0:

```text
Mallory can steal 100% of Epoch-0 emissions
```

despite not being bonded during that epoch at all.

If honest users existed:

* Mallory still dilutes their rewards retroactively.

> **Analogy**: Imagine a school says:
>
> ```text
> "Attendance closes at exactly 5:00 PM."
> ```
>
> But students entering at:
>
> ```text
> 5:00:00 PM
> ```
>
> are still counted as having attended the earlier class session.
>
> Students effectively gain retroactive attendance credit after class already ended.

## Vulnerable Code Reference

### 1) Epoch End Equals Next Epoch Start

```solidity
function _calculateEpochTimestampEnd(uint256 epoch)
    internal
    view
    returns (uint256)
{
    return _START_TIMESTAMP
        + (epoch * _EPOCH_LENGTH)
        + _EPOCH_LENGTH;
}
```

This returns:

```text
the first timestamp of the next epoch
```

NOT the final in-epoch timestamp.

### 2) currentEpoch Advances At That Exact Timestamp

```solidity
return (timestamp - _START_TIMESTAMP) / _EPOCH_LENGTH;
```

At:

```text
timestamp == epochTimestampEnd(n)
```

the protocol already considers itself inside:

```text
epoch n+1
```

### 3) Reward Snapshots Use Boundary Timestamp Directly

```solidity
return _totalSupply(_epochTimestampEnd(epoch));
```

and:

```solidity
return _balanceOf(account, _epochTimestampEnd(epoch));
```

### 4) Historical Lookups Are Inclusive

```solidity
if (point_history[_mid].ts <= _ts)
```

and:

```solidity
if (user_point_history[addr][_mid].ts <= _ts)
```

Meaning:

```text
boundary-second checkpoints are included
```

inside the older epoch snapshot.

## Root Cause

The vulnerability comes from the interaction of TWO independently reasonable decisions:

### Decision 1 — Epoch Arithmetic

The protocol defines:

```text
epochTimestampEnd(n)
```

as:

```text
the first second of epoch n+1
```

### Decision 2 — Inclusive Historical Snapshots

Historical lookups intentionally include checkpoints satisfying:

```text
ts <= snapshotTime
```

### Dangerous Combination

Together, these create:

```text
cross-epoch snapshot contamination
```

where:

* Epoch N snapshots accidentally include
* checkpoints created during Epoch N+1.

## Impact

This vulnerability enables:

* retroactive reward qualification,
* reward dilution,
* emissions theft,
* unfair veTRUST accounting.

The exploit is especially severe because:

* attacker capital is only temporarily locked,
* no privileged role is required,
* timing alone enables extraction.

The cleanest attack occurs at the Epoch-0 → Epoch-1 boundary:

* attacker joins only after Epoch 1 starts,
* but still captures Epoch-0 rewards.

## Recommended Mitigation

### 1) Make Reward Snapshots Exclusive

Instead of:

```solidity
_epochTimestampEnd(epoch)
```

snapshot at:

```solidity
_epochTimestampEnd(epoch) - 1
```

This ensures:

```text
Epoch N snapshot
strictly excludes Epoch N+1 checkpoints
```

### 2) Maintain Consistent Epoch Semantics

Ensure:

* `currentEpoch()`
* `epochTimestampEnd()`
* `_balanceOf()`
* `_totalSupply()`
* historical checkpoint lookups

all share the SAME inclusive/exclusive boundary convention.

### 3) Add Boundary Regression Tests

Specifically test:

```text
create_lock exactly at epoch boundary
increase_amount exactly at boundary
increase_unlock_time exactly at boundary
immediate reward claims after boundary
```

## Pattern Recognition Notes

* **Temporal Boundary Bugs**: Small differences between `<` and `<=` can create major financial vulnerabilities in reward systems.
* **Snapshot Contamination**: Systems using historical balance snapshots must clearly define whether boundaries are inclusive or exclusive.
* **Epoch Arithmetic Consistency**: If one component treats a timestamp as "new epoch" while another treats it as "old epoch," accounting corruption becomes possible.
* **Retroactive Eligibility**: Any system allowing users to qualify for past rewards after the qualifying window ended is vulnerable to reward theft.
* **Fix Tradeoffs**: The inclusive lookup (`<=`) was introduced to fix a previous underflow bug, but unintentionally introduced this retroactive accounting flaw.
* **Time Is A Security Boundary**: In staking/emission systems, timestamps are authorization boundaries just like signatures or access control.

## Quick Recall (TL;DR)

* **Bug**: Reward snapshots used a boundary timestamp that was ALSO the first second of the next epoch.
* **Issue**: Historical checkpoint lookup used inclusive `<=` logic.
* **Impact**: Checkpoints created at the first second of Epoch N+1 were counted inside Epoch N rewards.
* **Result**: Attackers could retroactively earn previous-epoch emissions without participating during that epoch.
* **Fix**: Make snapshots exclusive (`epochEnd - 1`) or standardize epoch-boundary semantics consistently across the system.
