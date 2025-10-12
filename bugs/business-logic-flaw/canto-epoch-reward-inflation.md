# Epoch Boundary Reward Inflation via Misaligned `nextEpoch` Calculation in `update_market`

* **Severity**: High
* **Source**: [Code4rena](https://github.com/code-423n4/2024-01-canto-findings/issues/9#issuecomment-1920975054)
* **Affected Contract**: [LendingLedger.sol](https://github.com/code-423n4/2024-01-canto/blob/5e0d6f1f981993f83d0db862bcf1b2a49bb6ff50/src/LendingLedger.sol)
* **Vulnerability Type**: Logic Error / Reward Inflation / Epoch Arithmetic Mistake

## Summary

In Canto's lending reward distribution logic, the function `update_market()` calculates accrued rewards between the last update and the current block.
However, it incorrectly computes the boundary of reward epochs using:

```solidity
uint256 nextEpoch = i + BLOCK_EPOCH;
```

instead of

```solidity
uint256 nextEpoch = epoch + BLOCK_EPOCH;
```

This subtle arithmetic mistake causes the function to **overcount blocks when crossing epoch boundaries**, resulting in **inflated rewards** for lenders and faster depletion of the reward pool.

## A Better Explanation (With Simplified Example)

### Intended Behavior

1. The system divides time into epochs of fixed length (`BLOCK_EPOCH = 100,000` blocks).
2. Each epoch can have a unique reward rate `cantoPerBlock[epoch]`.
3. When `update_market()` runs, it should:

   * Identify which epochs have passed since the last update.
   * For each, multiply the number of blocks in that epoch by its reward rate.
   * Update `accCantoPerShare` proportionally to the accrued rewards.

### What Actually Happens (Bug)

Because `nextEpoch` is derived from the **loop variable** `i` (current block), instead of the **epoch start**, the function misjudges where the current epoch ends.

That causes:

* Too many blocks to be counted under the current epoch's reward rate.
* Extra rewards to be distributed when transitioning from one epoch to another.

### Concrete Walkthrough (Example)

| Variable                | Value   |
| ----------------------- | ------- |
| `BLOCK_EPOCH`           | 100,000 |
| `cantoPerBlock[100000]` | 100     |
| `cantoPerBlock[200000]` | 0       |
| `lastRewardBlock`       | 199,999 |
| `current block`         | 200,002 |

**Expected logic:**

* Only 1 block (199,999 → 200,000) should get 100 CANTO/block.
* After that, rewards drop to 0.

✅ Correct total = `100 * 1 + 0 * 2 = 100`.

**Actual logic (bugged line):**

```solidity
nextEpoch = i + 100000; // instead of epoch + 100000
```

So:

```solidity
blockDelta = min(299,999, 200,002) - 199,999 = 3
reward = 3 * 100 = 300
```

❌ Inflated reward = 300 CANTO (3× higher than expected).

### Why This Matters

* Reward emissions are **over-distributed** near epoch transitions.
* Early lenders may receive **more tokens than they should**, draining the emission pool faster.
* Later users may earn less, breaking the fairness of the reward system.
* The bug only triggers occasionally — making it harder to detect without careful testing.

### Analogy

Imagine you pay employees weekly, but you accidentally overlap a few days when switching weeks. Every time Sunday hits, you pay three days’ salary twice. The books look fine short-term, but the company runs out of money earlier than expected.

## Vulnerable Code Reference

### 1) Wrong epoch boundary calculation

```solidity
uint256 nextEpoch = i + BLOCK_EPOCH; // ❌ uses loop index instead of epoch start
...
uint256 blockDelta = min(nextEpoch, block.number) - i;
accCantoPerShare += (blockDelta * cantoPerBlock[epoch]) / totalSupply;
```

### 2) Correct boundary logic

```solidity
uint256 nextEpoch = epoch + BLOCK_EPOCH; // ✅ aligns with true epoch limit
uint256 blockDelta = min(nextEpoch, block.number) - i;
accCantoPerShare += (blockDelta * cantoPerBlock[epoch]) / totalSupply;
```

## Recommended Mitigation

1. **Fix boundary computation**

   ```solidity
   uint256 nextEpoch = epoch + BLOCK_EPOCH;
   ```

   This ensures the loop correctly ends at the true epoch boundary rather than overshooting it.

2. **Add tests for epoch transitions**

   * Simulate reward updates just before and after each epoch change.
   * Verify that reward deltas remain consistent and no overcounting occurs.

3. **Add internal assertions**

   ```solidity
   assert(nextEpoch % BLOCK_EPOCH == 0);
   ```

   Prevents future arithmetic regressions if code is refactored.

4. **Review other time/epoch-based loops**
   Similar arithmetic patterns elsewhere (e.g., staking, emission schedules) may suffer from the same off-by-epoch issue.

## Pattern Recognition Notes

* **Epoch Boundary Misalignment**: Common in reward or emission logic where boundary arithmetic (`+ BLOCK_EPOCH`) depends on wrong base variable.
* **Silent Reward Inflation**: Bug doesn't revert — it overpays silently, making it financially impactful yet operationally invisible.
* **Transition-Triggered Errors**: Only occurs when moving between reward epochs; normal testing across static epochs may miss it.
* **Preventive Principle**: Always derive boundaries from **epoch identifiers**, not **current loop indices** or **block numbers**.

### Quick Recall (TL;DR)

* **Bug**: `nextEpoch = i + BLOCK_EPOCH` uses block number instead of epoch base.
* **Impact**: Over-accrues rewards when crossing epoch boundaries.
* **Fix**: `nextEpoch = epoch + BLOCK_EPOCH;` and add transition tests.
