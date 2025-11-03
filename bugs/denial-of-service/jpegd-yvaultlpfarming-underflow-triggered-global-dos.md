# Global Withdraw DoS via Negative-Delta Accounting in yVaultLPFarming

* **Severity**: High
* **Source**: [Code4rena](https://github.com/code-423n4/2022-04-jpegd-findings/issues/80)
* **Affected Contracts**: [`yVaultLPFarming.sol`](https://github.com/code-423n4/2022-04-jpegd/blob/e72861a9ccb707ced9015166fbded5c97c6991b6/contracts/farming/yVaultLPFarming.sol#L170), [`yVault.sol`](https://github.com/code-423n4/2022-04-jpegd/blob/e72861a9ccb707ced9015166fbded5c97c6991b6/contracts/vaults/yVault/yVault.sol#L108)
* **Vulnerability Type**: Denial of Service (DoS) / Arithmetic Underflow / State Desynchronization

## Summary

The **`yVaultLPFarming`** contract tracks JPEG rewards using a running balance delta:

```solidity
uint256 currentBalance = vault.balanceOfJPEG() + jpeg.balanceOf(address(this));
uint256 newRewards = currentBalance - previousBalance;
```

It assumes the total JPEG balance can only **increase** over time.
However, if `currentBalance < previousBalance` (for example, due to a vault controller change or reward withdrawal), the subtraction underflows and **reverts**, halting `_update()`.

Since `_update()` is called at the **start** of deposit, withdraw, and claim functions, **all user actions become impossible** — effectively a **global freeze of the farming contract**.

## A Better Explanation (With Simplified Example)

### Intended Behavior

1. **Reward accrual**:

   * The contract stores `previousBalance` = total JPEG held last time.
   * Each `_update()` call measures new rewards minted since then:
     `newRewards = currentBalance - previousBalance`.
   * These rewards are distributed proportionally to stakers via `accRewardPerShare`.

2. **Update flow**:

   * `_update()` calls `_computeUpdate()` to get new rewards.
   * `previousBalance` is updated to `currentBalance` afterward.

### What Actually Happens (Bug)

If the vault's JPEG balance **drops** instead of rising — for example, when the owner sets a **new controller** that reports a smaller `balanceOfJPEG()` — we get:

```text
previousBalance = 100
currentBalance  = 40
newRewards      = 40 - 100 → underflow → revert
```

Because Solidity ≥0.8 reverts on underflow, `_computeUpdate()` reverts.
That revert bubbles up into `_update()`, causing **every dependent function (deposit, withdraw, claim)** to revert too.

The result: **nobody can withdraw or interact anymore**.

### Why This Matters

* The farming system depends on `_update()` executing successfully before any state change.
* A single accounting inconsistency or vault controller change can **brick all interactions** globally.
* It doesn't require an attacker — a normal owner operation (`setController()`) or protocol upgrade can unintentionally trigger this.

### Concrete Walkthrough (Example Scenario)

1. `previousBalance = 100 JPEG`.
2. Owner replaces vault controller with a new one:

   ```solidity
   function setController(address _controller) public onlyOwner {
       controller = IController(_controller);
   }
   ```

   The new controller reports less JPEG (e.g., `balanceOfJPEG = 40`).
3. On the next `_update()` call:

   ```solidity
   currentBalance = 40 + jpeg.balanceOf(this);
   newRewards = currentBalance - previousBalance; // 40 - 100 → revert
   ```

4. `_update()` reverts → `withdraw()`, `deposit()`, and `claim()` revert as well.
   The entire pool is **stuck**.

> **Analogy**: The contract keeps a "previous balance receipt" assuming the total never shrinks.
> When it suddenly does, the math can't handle the negative — it tears the receipt in half, and the entire counter stops working.

## Vulnerable Code Reference

**1) In `yVaultLPFarming::_computeUpdate()`**

```solidity
uint256 currentBalance = vault.balanceOfJPEG() + jpeg.balanceOf(address(this));
uint256 newRewards = currentBalance - previousBalance; // underflow if current < previous
```

**2) In `yVault.setController()`**

```solidity
function setController(address _controller) public onlyOwner {
    require(_controller != address(0), "INVALID_CONTROLLER");
    controller = IController(_controller); // new controller may report smaller balance
}
```

**3) `balanceOfJPEG()` uses controller's balance view**

```solidity
function balanceOfJPEG() external view returns (uint256) {
    return controller.balanceOfJPEG(address(token)); // may drop suddenly
}
```

## Recommended Mitigation

1. **Tolerate balance decreases (primary fix)**

   Modify `_computeUpdate()` to handle negative deltas gracefully:

   ```solidity
   function _computeUpdate() internal view returns (uint256 newAcc, uint256 currentBalance) {
       currentBalance = vault.balanceOfJPEG() + jpeg.balanceOf(address(this));

       if (currentBalance <= previousBalance) {
           // no new rewards, just return current state
           newAcc = accRewardPerShare;
           return (newAcc, currentBalance);
       }

       uint256 newRewards = currentBalance - previousBalance;
       newAcc = accRewardPerShare + (newRewards * 1e36) / totalStaked;
   }
   ```

2. **Synchronize balances safely**

   Always set `previousBalance = currentBalance` after `_computeUpdate()` to keep state consistent, even when no rewards were added.

3. **Controller migration policy**

   Enforce a two-step controller change:

   * `setPendingController(newController)`
   * `migrateController()` — performs transfer/migration of JPEG, verifies balances, and only then finalizes.

4. **Add an emergency admin sync**

   ```solidity
   function emergencySync() external onlyOwner {
       previousBalance = vault.balanceOfJPEG() + jpeg.balanceOf(address(this));
       emit EmergencyBalanceSync(previousBalance);
   }
   ```

   → allows governance to recover from stuck state if something goes wrong.

5. **Unit tests**

   * Simulate `currentBalance < previousBalance` — should no longer revert.
   * Controller replacement scenarios.
   * Reentrancy and reward distribution sanity checks.

## Pattern Recognition Notes

* **Arithmetic Fragility** - When protocols rely on `current - previous` accounting, any negative delta (even from admin actions) can crash the logic.
* **Governance Coupling** - Admin operations like changing a controller should not impact user availability.
* **Underflow-Triggered DoS** - Solidity 0.8's automatic revert on underflow means logic errors double as denial-of-service vectors.
* **Safe Delta Tracking** - Always bound delta updates with `if (current < previous) { … }`.
* **Recoverability Hooks** - Include emergency sync/migration functions so the system can resynchronize without redeployment.

### Quick Recall (TL;DR)

* **Bug**: Reward update assumes balances only increase → subtraction underflow when they decrease.
* **Impact**: `_update()` reverts → deposits, withdrawals, and claims **fully freeze**.
* **Fix**: Add safeguard for `currentBalance <= previousBalance`; sync state safely; control migrations carefully; add emergency recovery hooks.
