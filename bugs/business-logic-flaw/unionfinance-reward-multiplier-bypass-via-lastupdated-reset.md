# Reward Multiplier Bypass via `lastUpdated` Reset on Repetitive Borrow

* **Severity**: High
* **Source**: [Sherlock](https://github.com/sherlock-audit/2022-10-union-finance-judging/issues/74)
* **Affected Contract**: [UserManager.sol](https://github.com/sherlock-audit/2022-10-union-finance/blob/main/union-v2-contracts/contracts/user/UserManager.sol)
* **Vulnerability Type**: Business Logic Flaw / Reward Manipulation / State Reset Abuse

## Summary

Union Finance calculates staker rewards based on borrower health using a **"frozen coin age"** metric. This metric depends on `vouch.lastUpdated`, which is intended to reflect borrower activity and detect overdue loans.

However, `lastUpdated` is incorrectly reset not only on repayments but also on **every new borrow**, even if the borrower is already overdue.

An attacker can exploit this by repeatedly borrowing a **minimal amount (e.g., 1 DAI)** to continuously reset `lastUpdated`, making overdue loans appear fresh. This allows stakers to **bypass penalties and claim maximum reward multipliers (up to 2x)** regardless of borrower health.

## A Better Explanation (With Simplified Example)

### Intended Behavior

1. **Borrowing & Repayment Tracking**:

   * Each borrower has a `lastUpdated` timestamp.
   * If too much time passes → borrower is considered **overdue**.

2. **Reward Calculation**:

   * Stakers earn rewards based on how healthy their borrowers are.
   * Overdue borrowers → increase `memberTotalFrozen` → **reduce rewards**.
   * Healthy borrowers → **higher reward multiplier (up to 2x)**.

### What Actually Happens (Bug)

* `lastUpdated` is reset on:

  * ✅ Initial borrow
  * ✅ Repayment
  * ❌ **Any additional borrow (even tiny amounts)**

* This means:

  * A borrower can be overdue
  * But a **new borrow instantly resets their "age"**

* In `getFrozenInfo()`:

```solidity
uint256 diff = block.number - lastUpdated;
```

* Since `lastUpdated` is always fresh → `diff ≈ 0`
* System thinks borrower is **not overdue**

### Why This Matters

* Borrowers don't need to repay to appear healthy
* Stakers avoid penalties without fixing bad debt
* Rewards system becomes **gameable and unfair**

### Concrete Walkthrough (Bob & Alice)

* **Setup**:

  * Bob = staker
  * Alice = borrower vouched by Bob
  * Alice is overdue

#### Step 1: Normal Case (No Exploit)

* Alice is overdue
* `lastUpdated` is old
* `diff` is large
* Bob gets **reduced rewards**

#### Step 2: Exploit

Bob tells Alice:

> "Borrow 1 DAI before I claim rewards."

This triggers:

```solidity
vouch.lastUpdated = block.number;
```

#### Step 3: System Gets Fooled

* `diff = block.number - lastUpdated ≈ 0`
* Protocol thinks:

  > "Borrower is active and healthy"

#### Step 4: Reward Withdrawal

Bob calls:

```solidity
withdrawRewards()
```

* Gets **maximum multiplier (2x)**
* Even though borrower is still overdue

#### Step 5: Repeat

* Before every reward claim:

  * Borrow small amount
  * Reset `lastUpdated`
  * Claim max rewards

→ **Penalty system completely bypassed**

> **Analogy**:
> A bank checks your credit health based on "last time you took a loan" instead of "last time you repaid it."
> So you can keep taking tiny loans to look financially healthy—even if you never repay.

## Vulnerable Code Reference

**1) `lastUpdated` reset on borrow (core issue)**

```solidity
// UserManager.sol - updateLocked()
vouch.locked += innerAmount;
vouch.lastUpdated = uint64(block.number); // resets even on repetitive borrow
```

**2) Same reset happens on unlock (repay path)**

```solidity
vouch.locked -= innerAmount;
vouch.lastUpdated = uint64(block.number);
```

**3) Frozen calculation depends on `lastUpdated`**

```solidity
// getFrozenInfo()
uint256 lastUpdated = vouch.lastUpdated;
uint256 diff = block.number - lastUpdated;
```

## Recommended Mitigation

1. **Use repayment-based tracking (preferred fix)**
   Replace `lastUpdated` with:

   ```solidity
   lastRepay
   ```

   → Only repayment should reset borrower health.

2. **Do NOT update on repetitive borrow**

```solidity
if (vouchLocked == 0) {
    vouch.lastUpdated = uint64(block.number); // only on initial borrow
}
```

3. **Separate concerns**

* Borrow activity ≠ repayment health
* Maintain distinct variables:

  * `lastBorrow`
  * `lastRepay`

4. **Add invariant tests**

* Repeated borrowing should NOT reduce overdue penalty
* Reward multiplier must reflect actual repayment behavior

## Pattern Recognition Notes

* **State Reset Abuse**:
  Critical metrics (like overdue tracking) should not be reset by unrelated actions (e.g., borrowing vs repaying).

* **Reward Manipulation via Cheap Actions**:
  If a system allows low-cost actions (like 1 DAI borrow) to influence high-value rewards, it is exploitable.

* **Metric Misrepresentation**:
  Using "last interaction" instead of "last meaningful interaction" leads to incorrect system state.

* **Coupled Logic Failure**:
  A single variable (`lastUpdated`) is used for multiple meanings → causes incorrect assumptions.

* **Economic Exploit > Technical Exploit**:
  No code break needed—just rational user behavior exploiting flawed incentives.

### Quick Recall (TL;DR)

* **Bug**: `lastUpdated` resets on every borrow
* **Exploit**: Borrow tiny amount → reset overdue timer
* **Impact**: Always get max rewards (2x), even with bad loans
* **Fix**: Track repayment (`lastRepay`) instead of borrow activity
