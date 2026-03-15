# Inflation Schedule Corruption via Premature `executeInflationRateUpdate` Call

* **Severity**: High
* **Source**: [Code4rena](https://github.com/code-423n4/2022-05-backd-findings/issues/99)
* **Affected Contract**: [`Minter.sol`](https://github.com/code-423n4/2022-05-backd/blob/2a5664d35cde5b036074edef3c1369b984d10010/protocol/contracts/tokenomics/Minter.sol)
* **Vulnerability Type**: Initialization Bug / Time-Based Accounting Corruption / Permissionless State Manipulation

## Summary

`Minter.sol` controls the **BKD token inflation schedule**, ensuring tokens are minted gradually over time according to a decaying emission model.

However, the contract contains an initialization flaw: **critical timestamps (`lastEvent` and `lastInflationDecay`) are not initialized during deployment**, and the permissionless function `executeInflationRateUpdate()` uses them directly without verifying that inflation has started.

If `executeInflationRateUpdate()` is called **before governance calls `startInflation()`**, the contract calculates inflation as if it has been running **since the Unix epoch (timestamp = 0)**. This results in:

1. **Massively inflated `totalAvailableToNow`**, effectively allowing minting of decades worth of emissions immediately.
2. **Premature activation of inflation decay logic**, which sets `initialPeriodEnded = true`, skipping the intended bootstrap reward phase.

This corrupts the token emission accounting and breaks the protocol's intended inflation schedule.

## A Better Explanation (With Simplified Example)

### Intended Behavior

The system expects inflation to begin only after governance explicitly activates it.

### Normal lifecycle

1. **Deploy Minter contract**

   ```
   lastEvent = 0
   lastInflationDecay = 0
   ```

2. **Governance starts inflation**

```solidity
startInflation()
```

This sets the initial timestamps:

```yaml
lastEvent = now
lastInflationDecay = now
```

3. **Inflation accumulates gradually**

```yaml
available_tokens = inflation_rate × time_elapsed
```

4. **Inflation manager periodically calls**

```solidity
executeInflationRateUpdate()
```

to update emission rates and apply yearly decay.

### What Actually Happens (Bug)

The function `executeInflationRateUpdate()` is **permissionless** and does not check if inflation has started.

If someone calls it before `startInflation()`:

```yaml
lastEvent = 0
```

The contract calculates elapsed time as:

```yaml
block.timestamp - lastEvent
≈ 52 years
```

Because `0` represents **January 1, 1970**.

So the contract believes **inflation has been running for decades**, immediately crediting a massive amount of mintable tokens.

## Why This Matters

The protocol relies on the invariant:

```yaml
totalMintedToNow <= totalAvailableToNow
```

to ensure inflation limits are respected.

But if `totalAvailableToNow` suddenly becomes **decades worth of emissions**, the mint limit becomes meaningless and the inflation schedule becomes corrupted.

Additionally, the **initial reward period for keepers and AMMs is skipped entirely**, because the decay condition evaluates to true instantly.

## Concrete Walkthrough (Alice & Mallory)

### Initial state after deployment

```yaml
lastEvent = 0
lastInflationDecay = 0
initialPeriodEnded = false
```

Inflation has **not started yet**.

### Mallory triggers the bug

Mallory calls:

```solidity
executeInflationRateUpdate()
```

before governance calls `startInflation()`.

### Step 1 — Decades of inflation credited

The contract executes:

```solidity
totalAvailableToNow += currentTotalInflation * (block.timestamp - lastEvent);
```

Since:

```yaml
lastEvent = 0
```

the elapsed time equals roughly **52 years**.

Example:

```yaml
currentTotalInflation = 4 tokens/sec
```

```yaml
52 years ≈ 1.64B seconds
```

So:

```yaml
totalAvailableToNow ≈ 6.56B tokens
```

The system now believes **billions of tokens are mintable**.

### Step 2 — Inflation decay logic triggers immediately

Next check:

```solidity
if (block.timestamp >= lastInflationDecay + 365 days)
```

Since:

```yaml
lastInflationDecay = 0
```

the condition is always true.

The contract therefore runs the decay logic and sets:

```yaml
initialPeriodEnded = true
```

This skips the **intended bootstrap reward phase** for keepers and AMMs.

### Step 3 — Emission schedule permanently corrupted

Later when minting occurs:

```solidity
require(newTotalMintedToNow <= totalAvailableToNow)
```

But `totalAvailableToNow` is already **massively inflated**, so the mint constraint no longer reflects the intended emission schedule.

> **Analogy**:
> Imagine a salary system that calculates how much someone can withdraw based on how long they have worked. If the system mistakenly thinks the employee started in **1970 instead of today**, it immediately believes they have earned **50+ years of salary**, allowing them to withdraw a huge amount instantly.

## Vulnerable Code Reference

### 1) Inflation start relies on manual initialization

```solidity
function startInflation() external override onlyGovernance {
    require(lastEvent == 0, "Inflation has already started.");
    lastEvent = block.timestamp;
    lastInflationDecay = block.timestamp;
}
```

If this function is not called first, timestamps remain `0`.

### 2) Permissionless function uses uninitialized timestamps

```solidity
function executeInflationRateUpdate() external override returns (bool) {
    return _executeInflationRateUpdate();
}
```

Anyone can trigger this function.

### 3) Inflation accounting assumes timestamps are initialized

```solidity
totalAvailableToNow += (currentTotalInflation * (block.timestamp - lastEvent));
```

If `lastEvent == 0`, the contract interprets inflation as running since the Unix epoch.

### 4) Decay logic activates immediately

```solidity
if (block.timestamp >= lastInflationDecay + _INFLATION_DECAY_PERIOD)
```

With:

```yaml
lastInflationDecay = 0
```

the condition is always true.

## Recommended Mitigation

### 1️⃣ Initialize timestamps during deployment

Set inflation start values in the constructor:

```solidity
lastEvent = block.timestamp;
lastInflationDecay = block.timestamp;
```

This prevents calculations from referencing the Unix epoch.

### 2️⃣ Prevent updates before inflation starts

Add a guard to `executeInflationRateUpdate()`:

```solidity
require(lastEvent != 0, "Inflation not started");
```

This ensures the inflation schedule cannot be manipulated before initialization.

### 3️⃣ Introduce an explicit inflation state flag

A clearer approach is using a dedicated flag:

```solidity
bool public inflationStarted;
```

Then enforce:

```solidity
require(inflationStarted)
```

before any inflation logic executes.

### 4️⃣ Add invariant tests

Unit tests should assert:

```yaml
executeInflationRateUpdate() reverts if inflation not started
```

to prevent regression of this bug.

## Pattern Recognition Notes

### Uninitialized Timestamp Bugs

Time-based accounting frequently uses:

```yaml
block.timestamp - lastUpdate
```

If `lastUpdate` is `0`, contracts may accidentally calculate **decades of elapsed time**.

### Permissionless State-Mutation Functions

Public functions that update global accounting variables should always ensure:

```yaml
system is fully initialized
```

before allowing execution.

### Inflation Accounting Fragility

Token emission systems rely on strict invariants:

```yaml
minted_supply ≤ scheduled_supply
```

Any incorrect timestamp can break this invariant and allow **excess token minting**.

### Initialization Order Dependency

If a contract requires a specific call order (e.g., `startInflation()` before updates), the code should **enforce that order explicitly** rather than relying on off-chain operational discipline.

## Quick Recall (TL;DR)

* **Bug**: `executeInflationRateUpdate()` can be called before inflation starts.
* **Root Cause**: `lastEvent` and `lastInflationDecay` default to `0`.
* **Impact**:

  * Contract credits **~50 years of inflation instantly**.
  * Bootstrap reward phase is skipped.
  * Inflation accounting becomes corrupted.
* **Fix**: Initialize timestamps in the constructor or require inflation to be started before executing updates.
