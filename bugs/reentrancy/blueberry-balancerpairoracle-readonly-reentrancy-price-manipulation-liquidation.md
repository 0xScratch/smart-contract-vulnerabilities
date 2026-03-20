# Read-Only Reentrancy Price Manipulation in BalancerPairOracle

* **Severity**: High
* **Source**: [Sherlock](https://github.com/sherlock-audit/2023-04-blueberry-judging/issues/141)
* **Affected Contract**: [BalancerPairOracle.sol](https://github.com/sherlock-audit/2023-04-blueberry/blob/main/blueberry-core/contracts/oracle/BalancerPairOracle.sol#L70-L92)
* **Vulnerability Type**: Read-Only Reentrancy / Oracle Manipulation / Inconsistent State Read

## Summary

`BalancerPairOracle.getPrice` calculates the price of Balancer LP tokens using **two external data sources**:

* `vault.getPoolTokens()` → returns token balances
* `pool.totalSupply()` → returns LP token supply

Due to a known **read-only reentrancy issue in Balancer Vault**, these two values can become **temporarily inconsistent during execution**. Specifically, LP supply may be updated **before** balances are updated.

An attacker can exploit this inconsistency window via reentrancy to force the oracle to compute an **artificially low LP price**, which can then be used to **prematurely liquidate user positions**.

## A Better Explanation (With Simplified Example)

### Intended Behavior

1. Oracle fetches:

   * token balances from the vault
   * LP total supply from the pool

2. Computes LP price:

   ```yaml
   LP Price = (value of reserves) / totalSupply
   ```

3. Assumes:

   * both values reflect the **same state**

### What Actually Happens (Bug)

Due to Balancer's internal execution order:

* `totalSupply` is updated **first**
* `balances` are updated **later**

During this gap:

* Reentrancy allows an attacker to call the oracle
* Oracle reads:

  * **old balances**
  * **new (inflated) supply**

This creates:

```yaml
price = old_value / inflated_supply → artificially LOW price
```

### Why This Matters

* The protocol believes collateral value has dropped
* Triggers **liquidation of healthy positions**
* Results in **direct user fund loss**

### Concrete Walkthrough (Alice & Mallory)

#### Setup

* Pool initially:

  * balances = $1000
  * supply = 100 LP
  * price = $10/LP

#### Mallory Attack

1. **Flash loan**

   * Borrow large capital

2. **Join Balancer pool**

   * Increases LP supply to 200
   * Balances NOT updated yet

3. **Reentrancy triggered**

   * Via ETH refund → `receive()`

#### Exploit Moment

At this exact point:

| Variable | Value         |
| -------- | ------------- |
| balances | $1000 ❌ (old) |
| supply   | 200 ✅ (new)   |

4. **Call liquidation**

   * Triggers `BalancerPairOracle.getPrice()`

#### Oracle Reads

```solidity
balances = vault.getPoolTokens()   // OLD
supply   = pool.totalSupply()      // NEW
```

#### Result

```text
price = 1000 / 200 = $5 (fake)
```

👉 Actual price = $10
👉 Reported price = $5

#### Outcome

* Protocol thinks collateral dropped 50%
* Liquidates Alice unfairly

> **Analogy**:
> Imagine calculating company share price using:
>
> * yesterday's assets
> * today's total shares
>
> You'd think the company lost value — even though nothing changed.

## Vulnerable Code Reference

**1) External balance read (unsafe during reentrancy)**

```solidity
(address[] memory tokens, uint256[] memory balances, ) =
    vault.getPoolTokens(pool.getPoolId());
```

**2) External supply read (can be updated earlier)**

```solidity
pool.totalSupply();
```

**3) Price calculation using inconsistent data**

```solidity
return (fairResA * price0 + fairResB * price1) / pool.totalSupply();
```

## Recommended Mitigation

1. **Check Balancer Vault reentrancy guard**

* Ensure oracle is not called during unsafe execution window
* Use Balancer-provided safeguards if possible

2. **Avoid relying on unsynchronized external reads**

* Fetch balances and supply in a way that guarantees consistency
* Or cache values in a trusted contract layer

3. **Move validation to critical execution paths**

Instead of relying on oracle safety:

* Add checks in:

  * `liquidate()`
  * borrowing logic

4. **Use protected oracle adapters**

* Wrap Balancer calls with:

  * reentrancy-aware checks
  * or state validation

5. **Fail-safe pricing**

* Reject prices if:

  * sudden large deviation
  * abnormal ratio changes

## Pattern Recognition Notes

* **Read-Only Reentrancy is Real**:
  Even `view` functions can be exploited if they read inconsistent external state.

* **Multi-Source Oracle Risk**:
  If a price depends on multiple external reads → ensure atomic consistency.

* **Timing-Based State Desync**:
  Bugs can arise not from logic errors, but from **when** data is read.

* **Division-Based Pricing Red Flag**:
  Patterns like:

  ```yaml
  price = value / totalSupply
  ```

  are highly sensitive to manipulation if inputs can desync.

* **External Protocol Assumptions**:
  Never assume another protocol (Balancer) behaves atomically or safely.

## Quick Recall (TL;DR)

* **Bug**: Oracle reads balances and supply during reentrancy → inconsistent state
* **Exploit**: Inflate supply before balances update → price drops artificially
* **Impact**: Fake low price → **unfair liquidations**
* **Fix**: Ensure atomic reads or guard against reentrancy windows
