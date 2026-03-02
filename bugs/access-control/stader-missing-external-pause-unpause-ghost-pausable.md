# Ghost Pausable: Contracts Inherit PausableUpgradeable but Cannot Be Paused

* **Severity**: Medium
* **Source**: [Code4rena](https://github.com/code-423n4/2023-06-stader-findings/issues/383)
* **Affected Contracts**:

  * [SocializingPool.sol](https://github.com/code-423n4/2023-06-stader/blob/main/contracts/SocializingPool.sol#L21)
  * [Auction.sol](https://github.com/code-423n4/2023-06-stader/blob/main/contracts/Auction.sol#L14)
  * [StaderOracle.sol](https://github.com/code-423n4/2023-06-stader/blob/main/contracts/StaderOracle.sol#L17)
  * [OperatorRewardsCollector.sol](https://github.com/code-423n4/2023-06-stader/blob/main/contracts/OperatorRewardsCollector.sol#L16)

* **Vulnerability Type**: Misconfiguration / Broken Emergency Control / Access Control Design Flaw

## Summary

Multiple Stader core contracts inherit OpenZeppelin's `PausableUpgradeable`, implying that they support emergency pause functionality. However, none of these contracts expose external `pause()` or `unpause()` functions.

Because `_pause()` and `_unpause()` in OpenZeppelin are **internal-only**, the pause mechanism is unreachable in practice. As a result, the protocol loses its intended emergency stop capability, creating operational and security risk during incidents.

This creates a **"ghost pausable"** condition: the contracts appear pausable in design but are permanently unpausable in reality.

## A Better Explanation (With Simplified Example)

### Intended Behavior

When using `PausableUpgradeable`, the expected control flow is:

1. Contract inherits `PausableUpgradeable`.
2. Sensitive functions use `whenNotPaused`.
3. Admin/manager can call external `pause()` in emergencies.
4. Protocol halts safely if something goes wrong.

Typical expected pattern:

```solidity
function pause() external onlyRole(MANAGER_ROLE) {
    _pause();
}
```

### What Actually Happens (Bug)

In the affected Stader contracts:

* They **inherit** `PausableUpgradeable`.
* They **have access** to `_pause()` and `_unpause()`.
* But they **never expose external wrappers**.

Because of this:

* No admin can call pause.
* No manager can call pause.
* The contract itself never calls `_pause()` internally.

👉 The pause mechanism is effectively dead code.

### Why This Matters

Pause functionality exists to mitigate emergencies such as:

* critical logic bugs
* oracle failures
* reward miscalculations
* fund-draining exploits

Without a working pause:

* incidents cannot be contained
* damage continues until upgrade (if possible)
* response time increases dramatically

### Concrete Walkthrough (Alice & Protocol Team)

**Setup**

The protocol deploys `SocializingPool` and assumes it is pausable.

**Incident**

1. A bug is discovered that allows unintended reward extraction.
2. The team attempts emergency response.
3. They look for a pause mechanism.

**Expected**

```yaml
Admin → pause() → contract paused → damage contained
```

**Actual**

```yaml
Admin → ❌ no pause() function exists
Contract → keeps running
Exploit → continues
```

The team has **no immediate circuit breaker**, even though the codebase suggests one exists.

> **Analogy**: Installing a fire alarm system but never wiring the emergency button. The building looks protected, but when a fire starts, no one can trigger the alarm.

## Vulnerable Code Reference

### 1) Contracts inherit PausableUpgradeable

Example pattern in affected contracts:

```solidity
contract SocializingPool is Initializable, PausableUpgradeable {
```

This gives access to:

```solidity
function _pause() internal virtual
function _unpause() internal virtual
```

### 2) Missing external pause/unpause entrypoints

There is **no implementation** like:

```solidity
function pause() external { _pause(); }
function unpause() external { _unpause(); }
```

Because `_pause()` is internal, the system has **no reachable path** to the paused state.

### 3) whenNotPaused becomes ineffective

Even if functions use:

```solidity
modifier whenNotPaused
```

they will **never actually be blocked**, because the contract can never enter the paused state.

## Recommended Mitigation

### 1. Add external pause/unpause functions (primary fix)

Each affected contract should expose controlled entrypoints:

```solidity
function pause() external {
    UtilLib.onlyManagerRole(msg.sender, staderConfig);
    _pause();
}

function unpause() external onlyRole(DEFAULT_ADMIN_ROLE) {
    _unpause();
}
```

### 2. Verify role design

Ensure:

* pause authority is limited (manager/emergency role)
* unpause authority is stricter (admin/governance)
* roles are properly initialized in upgradeable context

### 3. Add emergency response tests

Include tests that verify:

* contract can be paused
* paused functions revert
* contract can be unpaused
* roles are enforced correctly

### 4. Optional hardening

Consider:

* emitting explicit pause events (already in OZ)
* documenting emergency runbooks
* monitoring paused state on-chain

## Pattern Recognition Notes

* **Ghost Pausable Pattern**: Inheriting `PausableUpgradeable` without exposing external pause controls leaves the circuit breaker permanently unreachable.
* **Security Theater Risk**: The presence of pause modifiers can create a false sense of safety for developers and auditors.
* **Upgradeable Contract Gotcha**: Teams often inherit OZ modules but forget to wire the external control surface.
* **Emergency Preparedness Failure**: Missing pause is especially dangerous in fund-handling or reward distribution contracts.
* **Audit Heuristic**: Whenever you see `is PausableUpgradeable`, immediately search for reachable `pause()`/`unpause()`.

## Quick Recall (TL;DR)

* **Bug**: Contracts inherit `PausableUpgradeable` but expose no external pause/unpause.
* **Impact**: Protocol cannot be emergency-stopped despite appearing pausable.
* **Root Cause**: `_pause()` and `_unpause()` are internal-only.
* **Fix**: Add properly access-controlled external pause/unpause functions.
