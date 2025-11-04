# Rounding Error Interest Loss via Day-Truncation in Interest Calculation

* **Severity**: High
* **Source**: [OpenCoreCH - Rigor Report](https://github.com/OpenCoreCH/smart-contract-audits/blob/main/reports/c4/rigor.md#high-significant-rounding-errors-for-interest-calculation)
* **Affected Contract**: [Community.sol](https://github.com/code-423n4/2022-08-rigor/blob/e35f5f61be9ff4b8dc5153e313419ac42964d1fd/contracts/Community.sol)
* **Vulnerability Type**: Arithmetic Precision / Time Granularity Error / Financial Logic Fault

## Summary

`Community.sol` accrues lender interest using **integer-based day counting**, truncating fractional days:

```solidity
uint256 _noOfDays = (block.timestamp - _communityProject.lastTimestamp) / 86400;
```

Since Solidity integer division rounds down, any elapsed time **below a full 24-hour block** is treated as **zero days**, producing **no interest** for that period.

If `lastTimestamp` is frequently updated — such as through `lendToProject()`, `claimInterest()`, or `reduceDebt()` — **interest accumulation resets prematurely**, effectively deleting accrued but uncredited interest.

Over many cycles, this causes **partial or total loss of expected yield**, especially when these functions are called frequently or just before the 24-hour mark.

## A Better Explanation (With Simplified Example)

### Intended Behavior

Interest is supposed to grow continuously over time based on APR (Annual Percentage Rate):

```text
interest = (principal × APR × elapsed_time) / (365 days)
```

Example:
If Alice lends $1,000 at 10% APR, after 1 day she should earn:

```text
(1000 × 10 × 1) / 365000 = 0.027 rTokens
```

So in 30 days, she earns about $0.82 worth of interest.

### What Actually Happens (Bug)

`returnToLender()` truncates fractional days when computing `_noOfDays`:

```solidity
uint256 _noOfDays = (block.timestamp - _communityProject.lastTimestamp) / 86400;
```

Thus:

| Actual Time Elapsed | Computed `_noOfDays` | Interest Accrued |
| ------------------- | -------------------- | ---------------- |
| 0.9 days (23h 59m)  | 0                    | ❌ 0 interest     |
| 1.5 days            | 1                    | ✅ 1-day interest |
| 1.99 days           | 1                    | ⚠️ ~50% interest |

When a user calls `lendToProject()` or `reduceDebt()` just before a full day completes,
`claimInterest()` gets called internally, resetting `lastTimestamp` without accruing any interest.

Repeating this repeatedly ensures **no daily interest is ever recognized**.

### Concrete Walkthrough (Alice & Bob)

* **Setup**: Alice lends $5,000 to Bob's project at 6% APR.

#### 1️⃣ Expected

After one day (86400s), she should get:

```text
(5000 × 6 × 1) / 365000 = 0.82 tokens interest
```

#### 2️⃣ Actual

If Alice (or the protocol) calls `lendToProject()` again after **86390 seconds** (23h 59m 50s):

* `_noOfDays = 0`
* `claimInterest()` does nothing
* `lastTimestamp` resets to the current block time

Interest for that day is **wiped out**.

If this repeats daily, Alice accrues **no interest at all** despite continuous lending.

#### 3️⃣ Partial Example

If actions occur every 1.99 days, `_noOfDays = 1` — only **50% of the expected interest** is accrued. Over time, this results in a **permanent yield shortfall**.

## Why This Matters

This bug silently erodes the lender's yield — the most critical metric in a lending protocol.

* **Financial Impact**:
  A lender expecting 6% APR on a $5M construction loan loses up to 25% of expected interest, i.e. **$75,000** per year.
  Such deviation destroys lender trust and damages the platform's credibility.

* **Technical Impact**:
  The issue stems from **precision loss** due to day-level granularity.
  Solidity integer division truncates decimals, so interest under 1 full day's duration is always ignored.

* **Attack Surface**:

  * **Unintentional:** frequent updates by the lender or automated systems.
  * **Intentional:** a builder could manipulate project activity frequency to keep resetting timestamps before full-day thresholds.

## Vulnerable Code Reference

**1) Truncated day calculation in `returnToLender()`**

```solidity
uint256 _noOfDays = (block.timestamp - _communityProject.lastTimestamp) / 86400;
uint256 _unclaimedInterest =
    _lentAmount * _communities[_communityID].projectDetails[_project].apr * _noOfDays / 365000;
```

**2) Frequent `lastTimestamp` resets in `claimInterest()`**

```solidity
if (_interestEarned > 0) {
    _communityProject.lastTimestamp = block.timestamp;
}
```

When called indirectly via `lendToProject()` or `reduceDebt()`, this reset happens often — even if `_interestEarned == 0`.

## Recommended Mitigation

1. **Use time in seconds instead of full days**
   Replace integer day division with second-based calculation for continuous interest accrual:

   ```solidity
   uint256 timeElapsed = block.timestamp - _communityProject.lastTimestamp;
   uint256 _unclaimedInterest =
       (_lentAmount * _communities[_communityID].projectDetails[_project].apr * timeElapsed)
       / (365 * 86400 * 1000);
   ```

   → This preserves fractional interest and ensures linear accrual.

2. **Add guard against premature timestamp resets**
   Only update `lastTimestamp` when actual positive interest is accrued.

   ```solidity
   if (_interestEarned > 0) {
       _communityProject.lastTimestamp = block.timestamp;
   }
   ```

3. **Add unit tests for sub-day scenarios**
   Test for:

   * <1 day elapsed (should accrue proportional interest)
   * 1.99 days (should yield ~1.99 days' worth)
   * Frequent repeated actions (should not cause zero-interest periods)

4. **Consider APR precision improvements**
   If using fractional APRs or continuous compounding, adopt fixed-point math (e.g., using `wadMul` or similar libraries).

## Pattern Recognition Notes

* **Truncation Granularity Bugs**:
  When dividing by time constants (like 86400), fractional loss can aggregate into large discrepancies.

* **Temporal Reset Attacks**:
  Any protocol where state resets based on elapsed time is vulnerable if the update interval truncates partial units.

* **Precision-Sensitive Financial Logic**:
  Lending protocols should never use integer day counts for interest accrual — second-based or block-based deltas are safer.

* **Silent Value Drift**:
  Bugs that slightly underpay each cycle can escape detection for long periods but cause major aggregate loss.

* **Unit Testing Lesson**:
  Always simulate sub-unit intervals and repeated calls — many real-world finance errors stem from rounding and granularity edge cases.

### Quick Recall (TL;DR)

* **Bug**: `(timeDiff / 86400)` truncates fractional days → partial or zero interest accrued.
* **Impact**: Lenders lose up to 25-100% of earned interest, damaging protocol credibility.
* **Fix**: Use second-based precision for time deltas and avoid resetting `lastTimestamp` unless actual interest is credited.
