# Consensus Stall via Strict Equality in StaderOracle Submissions

* **Severity**: Medium
* **Source**: [Code4rena](https://github.com/code-423n4/2023-06-stader-findings/issues/321) / [One Bug Per Day](https://www.onebugperday.com/v1/273)
* **Affected Contract**: [StaderOracle.sol](https://github.com/code-423n4/2023-06-stader/blob/main/contracts/StaderOracle.sol#L148)
* **Vulnerability Type**: Logic Error / Consensus Liveness Failure

## Summary

`StaderOracle` aggregates submissions from trusted nodes and finalizes data (exchange rates, SD token price, rewards roots, penalties, etc.) once **majority consensus** is reached.

The contract currently checks consensus using **strict equality (`==`)** against a dynamic majority threshold derived from `trustedNodesCount`.

Because `trustedNodesCount` can change mid-round (e.g., nodes added/removed), the exact equality may **never be satisfied**, leaving consensus permanently "stuck." This means important oracle updates may fail to finalize, stalling core protocol operations.

## A Better Explanation (With Simplified Example)

### Intended Behavior

1. A group of **trusted nodes** submit data (e.g., exchange rate).
2. Once more than half (or 2/3 for SD price) agree, consensus is reached.
3. The oracle finalizes the update and clears submissions.

### What Actually Happens (Bug)

* Consensus condition uses `==` instead of `>=`.

```solidity
// Example for exchange rate
if (submissionCount == trustedNodesCount/2 + 1) {
    updateWithInLimitER(...);
}
```

* If the number of trusted nodes changes mid-round:

  * The majority threshold shifts downward.
  * `submissionCount` may already exceed the new threshold.
  * Since equality can't be reached (`6 == 5` fails), consensus is never finalized.

* For SD price submissions, stale data isn't cleared if equality is never reached, causing contamination of the next batch.

### Why This Matters

* **Exchange rate stalls**: ETH/ETHx ratio may not update for an epoch (mild impact).
* **SD price contamination**: Old submissions remain in `sdPrices`, corrupting the next median calculation.
* **Rewards/penalties stall**: Failure to finalize rewards or penalties can block protocol accounting, potentially freezing payouts or missed-slash reporting.

### Concrete Walkthrough (Alice & Mallory)

* **Setup**: `trustedNodesCount = 10`; majority needed = `10/2 + 1 = 6`.
* **Alice & peers submit**: 5 nodes have submitted so far → `submissionCount = 5`.
* **Node removed**: `trustedNodesCount = 9`; new majority = `9/2 + 1 = 5`.
* **Mallory submits**: Another submission arrives → `submissionCount = 6`.
* **Check**:

  ```solidity
  if (6 == 5) // fails
  ```

  Even though > majority, consensus never finalizes.

> **Analogy**: Imagine a committee vote where the rule is: *exactly 6 votes needed*. If membership changes and the new rule requires 5 votes, the fact that 6 people already voted "yes" doesn't count — the system insists on *exactly 5*. The decision is never passed.

## Vulnerable Code Reference

### 1) Exchange Rate (line 148)

```solidity
if (
    submissionCount == trustedNodesCount / 2 + 1 &&
    _exchangeRate.reportingBlockNumber > exchangeRate.reportingBlockNumber
) {
    updateWithInLimitER(...);
}
```

### 2) SD Price (line 290)

```solidity
if ((submissionCount == (2 * trustedNodesCount) / 3 + 1)) {
    lastReportedSDPriceData = _sdPriceData;
    lastReportedSDPriceData.sdPriceInETH = getMedianValue(sdPrices);
    delete sdPrices; // not reached if equality fails
}
```

## Recommended Mitigation

1. **Use `>=` instead of `==`** for consensus checks.

   ```solidity
   if (submissionCount >= trustedNodesCount/2 + 1) { ... }
   ```

   ```solidity
   if (submissionCount >= (2 * trustedNodesCount)/3 + 1) { ... }
   ```

2. **Reset state defensively**: Ensure `sdPrices` (and other submission buffers) are cleared after each reporting round, even if thresholds change.

3. **Optional safeguard**: Add cooldown or restrict adding/removing trusted nodes mid-round to avoid shifting thresholds.

## Pattern Recognition Notes

* **Strict Equality in Dynamic Thresholds**: When quorum thresholds are derived from live counts, using `==` instead of `>=` risks missing consensus.
* **State Leakage**: Failure to clear temporary storage (`sdPrices`) contaminates future consensus.
* **Governance Liveness Risks**: Dynamic membership systems must be resilient to mid-round changes or enforce fixed membership per round.
* **Consensus Rule Design**: Always allow "at least" majority, not "exactly majority," unless there's a strong invariant that prevents overshoot.

### Quick Recall (TL;DR)

* **Bug**: Consensus checks require exact equality (`==`) against a moving threshold.
* **Impact**: Oracle updates may stall; SD price contamination can break next round; rewards/penalties may freeze.
* **Fix**: Replace `==` with `>=` and reset submission buffers reliably.

Would you like me to also add a **"severity reasoning"** section (like why this is Medium vs High), similar to the judge's comment, so it becomes more realistic for training?
