# Liquidation Enabled While Repay Disabled Leading to Forced Loss of Funds

* **Severity**: Medium
* **Source**: [Sherlock](https://github.com/sherlock-audit/2023-02-blueberry-judging/issues/290)
* **Affected Contract**: [BlueBerryBank.sol](https://github.com/sherlock-audit/2023-02-blueberry/blob/main/contracts/BlueBerryBank.sol#L740)
* **Vulnerability Type**: Access Control / Logic Flaw / State Inconsistency

## Summary

`BlueBerryBank` allows the admin to disable specific core actions such as **repayment**, **borrowing**, and **lending** via a bitmask (`bankStatus`). However, **liquidation remains enabled even when repayment is disabled**.

This creates a critical imbalance: users cannot repay their debt to reduce risk, while liquidators are still allowed to liquidate positions. As interest accrues over time, positions can cross the liquidation threshold, leaving users **helpless and forcibly liquidated**, resulting in loss of collateral.

## A Better Explanation (With Simplified Example)

### Intended Behavior

1. Users open leveraged positions:

   * Deposit collateral
   * Borrow funds
2. If risk increases:

   * Users can **repay debt** to reduce risk
3. If risk exceeds threshold:

   * Position becomes **liquidatable**

👉 Key invariant:

> Users must always have a way to **defend their position** (e.g., repay)

### What Actually Happens (Bug)

* Admin disables repayment:

  ```solidity
  bankStatus = bankStatus & ~0x02; // disable repay
  ```
* `repay()` becomes unusable:

  ```solidity
  if (!isRepayAllowed()) revert REPAY_NOT_ALLOWED();
  ```
* However, **liquidation remains active**:

  ```solidity
  function liquidate(...) {
      if (!isLiquidatable(positionId)) revert;
  }
  ```

👉 No check linking liquidation to repay availability

### Why This Matters

* Users **cannot reduce debt**
* Interest **continues to accrue**
* Risk **increases automatically**
* Liquidators can still **seize collateral**

👉 This creates a **one-sided failure mode**

### Concrete Walkthrough (Alice & Admin)

#### Setup:

* Alice opens a leveraged position:

  * Collateral = $1000
  * Debt = $700

#### Step 1: Admin disables repay

* `isRepayAllowed() = false`
* Alice cannot repay anymore

#### Step 2: Interest accrues

* Debt grows → $700 → $800 → $900

#### Step 3: Position becomes risky

* Collateral < Debt
* Liquidation threshold crossed

#### Step 4: Liquidation happens

```solidity
liquidate(positionId, ...)
```

* Liquidator repays part of debt
* Takes Alice's collateral

#### ❌ Critical Issue:

Alice **wanted to repay**, but **was not allowed to**

> **Analogy**:
> A bank freezes your ability to repay your loan but still penalizes you for being undercollateralized and seizes your assets.

## Vulnerable Code Reference

### 1) Repayment is conditionally disabled

```solidity
function repay(...) {
    if (!isRepayAllowed()) revert REPAY_NOT_ALLOWED();
}
```

### 2) Liquidation has no such restriction

```solidity
function liquidate(...) {
    if (!isLiquidatable(positionId)) revert NOT_LIQUIDATABLE(positionId);
}
```

👉 Missing:

```solidity
require(isRepayAllowed(), "Repay disabled");
```

### 3) Admin control over system state

```solidity
function setBankStatus(uint256 _bankStatus)
```

* Bitmask controls:

  * Borrow
  * Repay
  * Lend

👉 But liquidation is **not governed by this state**

## Recommended Mitigation

### 1. Disallow liquidation when repay is disabled (Primary Fix)

```solidity
if (!isRepayAllowed()) revert REPAY_DISABLED();
```

Add this check inside `liquidate()`

### 2. Alternatively: Never disable repay

* Repayment is a **safety-critical function**
* Should always remain enabled

### 3. Graceful shutdown design (Advanced)

If protocol must pause:

* Disable:

  * Borrow
  * New positions
* Keep enabled:

  * Repay
  * Collateral withdrawal (if safe)

### 4. Add invariant checks

* Ensure:

  > If liquidation is allowed → repayment must also be allowed

## Pattern Recognition Notes

* **Asymmetric State Control**
  Critical system actions (repay vs liquidate) must be **symmetrically governed**. Disabling only one side creates exploitable imbalance.

* **User Defense Removal**
  Any system where users lose the ability to **reduce risk** while still being penalized is fundamentally unsafe.

* **Admin-Induced Risk Escalation**
  Admin controls that indirectly increase user risk (like disabling repay) can lead to **forced liquidations** and fund loss.

* **Time-Based Risk Drift**
  Interest accrual continues even when user actions are restricted → creates inevitable liquidation conditions.

* **Missing Safety Invariants**
  Protocol should enforce:

  > "If a position can be liquidated, the user must have at least one way to prevent it."

## Quick Recall (TL;DR)

* **Bug**: Repay can be disabled while liquidation remains enabled
* **Impact**: Users cannot defend positions → forced liquidation → loss of funds
* **Root Cause**: Missing linkage between repay and liquidation logic
* **Fix**: Disallow liquidation when repay is disabled OR always allow repay
