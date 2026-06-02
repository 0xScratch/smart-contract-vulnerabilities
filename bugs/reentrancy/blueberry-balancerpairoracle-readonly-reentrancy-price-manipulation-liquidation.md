# Premature Liquidation via Balancer Read-Only Reentrancy Oracle Manipulation

* **Severity**: High
* **Source**: [Sherlock](https://github.com/sherlock-audit/2023-04-blueberry-judging/issues/141)
* **Affected Contract**: [BalancerPairOracle.sol](https://github.com/sherlock-audit/2023-04-blueberry/blob/main/blueberry-core/contracts/oracle/BalancerPairOracle.sol#L70-L92)
* **Vulnerability Type**: Read-Only Reentrancy / Oracle Manipulation / Inconsistent State Read

## Summary

`BalancerPairOracle.getPrice()` calculates the value of a Balancer LP token by combining pool balances retrieved from the Balancer Vault with the pool's LP token supply (`totalSupply()`).

The oracle assumes these values represent the same pool state snapshot. However, due to a known Balancer read-only reentrancy issue, an attacker can force the oracle to observe a temporary intermediate state where the pool balances and LP supply are out of sync.

By triggering a reentrant call during a Balancer pool operation, the attacker can make the oracle read:

* **Old pool balances**
* **Newly increased LP token supply**

As a result, the calculated LP token price becomes artificially low, allowing healthy collateral positions to appear undercollateralized and become eligible for liquidation.

## A Better Explanation (With Simplified Example)

### Intended Behavior

The oracle determines the LP token price using:

```text
LP Price =
(Total Value of Pool Assets)
/
(Total LP Token Supply)
```

For example:

```text
Pool Assets = $1,000,000
LP Supply   = 1,000

LP Price = $1,000
```

The oracle expects both values to come from the same pool state.

### What Actually Happens (Bug)

Balancer pool operations update multiple pieces of state during execution.

During certain execution windows:

```text
LP supply has already been updated

BUT

Vault token balances have not yet been updated
```

Normally, external users cannot observe this intermediate state.

However, through reentrancy, an attacker can execute arbitrary logic while Balancer is still in the middle of updating its accounting.

The oracle then reads:

```text
Balances = Old Value
Supply   = New Value
```

and computes a completely incorrect LP price.

### Why This Matters

The protocol uses this oracle to determine collateral value during critical operations such as liquidation.

If an attacker can artificially reduce the oracle price:

* Healthy positions appear unhealthy.
* Collateral appears worth significantly less than reality.
* Liquidations become possible when they should not be.
* Victims lose collateral despite remaining properly collateralized.

The issue does not require manipulation of actual market prices.

The attacker only manipulates the timing of oracle reads.

## Concrete Walkthrough (Alice & Mallory)

### Initial State

Balancer pool:

```text
Pool Value = $1,000,000
LP Supply  = 1,000

LP Price = $1,000
```

Alice uses:

```text
100 LP Tokens
```

as collateral.

Real collateral value:

```text
100 × $1,000 = $100,000
```

Position is healthy.

---

### Mallory's Attack

Mallory obtains a large flash loan:

```text
$100,000,000
```

and calls:

```solidity
BalancerVault.joinPool(...)
```

to perform a massive pool deposit.

During execution:

#### Step 1: LP Supply Updates

Balancer mints new LP tokens.

```text
Old Supply = 1,000
New Supply = 101,000
```

Supply is already updated.

#### Step 2: Vault Balances Not Yet Updated

Expected balances:

```text
$101,000,000
```

Actual balances currently visible:

```text
$1,000,000
```

The system is temporarily inconsistent.

---

### Reentrancy Trigger

While processing the pool join, Balancer refunds excess ETH.

This refund triggers:

```solidity
receive()
```

inside Mallory's contract.

Execution flow:

```text
joinPool()

    ↓

Supply Updated

    ↓

ETH Refund

    ↓

Attacker receive()

    ↓

Liquidation Call
```

Mallory now performs:

```solidity
BlueBerryBank.liquidate(...)
```

during the inconsistent state.

---

### Oracle Reads Broken Data

Inside `BalancerPairOracle.getPrice()`:

```solidity
vault.getPoolTokens(...)
```

returns:

```text
Balances = $1,000,000
```

Then:

```solidity
pool.totalSupply()
```

returns:

```text
Supply = 101,000
```

The oracle computes:

```text
LP Price =
1,000,000
/
101,000

≈ $9.90
```

instead of:

```text
$1,000
```

The LP price has effectively collapsed by ~99%.

---

### Alice Gets Liquidated

Alice's collateral is now valued as:

```text
100 × $9.90

≈ $990
```

instead of:

```text
$100,000
```

Blueberry believes the position is underwater.

Liquidation becomes possible.

Mallory liquidates the position and acquires collateral at Alice's expense.

Once Balancer finishes execution, the pool returns to a consistent state, but the liquidation has already occurred.

> **Analogy:** Imagine calculating the price of a company's shares by dividing company assets by the number of shares outstanding. During an accounting update, someone temporarily shows you the new share count but the old asset balance. The company suddenly appears nearly worthless even though nothing actually changed. If a bank uses that incorrect valuation to determine whether your loan is healthy, it may seize your assets unfairly.

## Vulnerable Code Reference

### Oracle Reads Pool Balances

```solidity
(address[] memory tokens, uint256[] memory balances, ) =
    vault.getPoolTokens(pool.getPoolId());
```

### Oracle Reads LP Supply Separately

```solidity
pool.totalSupply();
```

### LP Price Calculation

```solidity
return (fairResA * price0 + fairResB * price1) / pool.totalSupply();
```

The vulnerability exists because these values are assumed to belong to the same state snapshot even though Balancer's read-only reentrancy allows them to become temporarily desynchronized.

## Root Cause Analysis

The issue is not caused by incorrect pricing mathematics.

The issue is caused by trusting externally supplied data that can be observed during an inconsistent state.

The oracle assumes:

```text
Pool Balances
and
LP Supply
```

are always synchronized.

Balancer's read-only reentrancy behavior invalidates that assumption.

Consequently, the oracle becomes vulnerable to snapshot inconsistency attacks.

## Recommended Mitigation

### 1. Check Balancer Vault Reentrancy Status Before Consuming Data

Follow Balancer's recommended approach and ensure critical oracle reads cannot occur while the Vault is inside a reentrant execution path.

This is the primary mitigation recommended by the Balancer team.

### 2. Protect Critical Functions

Critical actions that depend on Balancer pricing should validate that the Vault is not currently executing within a reentrant context before using oracle values.

Examples include:

```solidity
liquidate()
borrow()
withdraw()
```

if they rely on Balancer-based collateral valuation.

### 3. Avoid Mixing Potentially Unsynchronized Data Sources

Where possible, use data sources that are guaranteed to come from the same state snapshot rather than independently querying balances and supply.

### 4. Add Integration Tests

Add tests that simulate:

* Pool joins
* Pool exits
* ETH refunds
* Reentrant callbacks

and verify that liquidation logic cannot execute using inconsistent Balancer state.

## Pattern Recognition Notes

* **Read-Only Reentrancy**: A protocol can be vulnerable even when no state-changing reentrancy occurs. Reading inconsistent data may be sufficient to exploit the system.
* **Snapshot Assumption Failure**: Oracle calculations often assume all retrieved values originate from the same state snapshot. External integrations may violate this assumption.
* **Multi-Source Oracle Construction**: Any oracle that combines multiple external reads (balances, reserves, supply, rates, exchange values) should be reviewed for synchronization risks.
* **Known Third-Party Integration Risks**: When a dependency publicly discloses a vulnerability class (as Balancer did), all downstream integrations must account for it.
* **Liquidation Dependency Risk**: Oracle manipulation becomes especially dangerous when liquidation eligibility depends directly on the manipulated value.

## Pattern Recognition Heuristic

When reviewing code, pay special attention to patterns like:

```solidity
balanceA = protocol.balanceA();
balanceB = protocol.balanceB();
supply   = protocol.totalSupply();

price = f(balanceA, balanceB, supply);
```

Ask:

> "Can these values be observed from different moments in time during reentrancy?"

If the answer is yes, the oracle may be vulnerable to read-only reentrancy manipulation.

## Quick Recall (TL;DR)

* **Bug**: Oracle reads Balancer balances and LP supply during a reentrant intermediate state.
* **Root Cause**: Balancer balances and LP supply can become temporarily desynchronized.
* **Attack**: Reenter during `joinPool()`, force oracle to read old balances and new supply.
* **Impact**: LP token price becomes artificially low, allowing premature liquidations.
* **Fix**: Ensure Balancer data is not consumed during reentrant execution and follow Balancer's recommended reentrancy protections.
