# Curve LP Oracle Manipulation via Read-Only Reentrancy

* **Severity**: High (Critical Impact, Medium Likelihood)
* **Source**: [ChainSecurity Post-Mortem](https://www.chainsecurity.com/blog/curve-lp-oracle-manipulation-post-mortem)
* **Affected Contract**: Curve StableSwap Pools (via `get_virtual_price`)
* **Vulnerability Type**: Oracle Manipulation / Read-Only Reentrancy

## Summary

Curve's `get_virtual_price` is widely used as an **oracle** to value Curve LP tokens.
It works by dividing the **invariant `D` (total pool value)** by the **total LP token supply**.

Normally, this ratio can only **increase** over time because swap fees accumulate into the pool, ensuring a **monotonic (non-decreasing) invariant**.

However, by exploiting **read-only reentrancy**, an attacker could **temporarily desynchronize `D` and the LP supply** during a reentrant read. This breaks the invariant assumption and allows **manipulated virtual price values** to be returned.

This impacted protocols relying on Curve LP token oracles (e.g., MakerDAO, Enzyme, Abracadabra, TribeDAO, Opyn).

## A Better Explanation (With Simplified Example)

### Intended Behavior

1. Curve computes `virtual_price = D / totalSupply`.
2. Since both numerator (`D`) grows with fees and denominator (`totalSupply`) grows only with new LP mints/burns, the ratio should **always go up or stay flat**.
3. External protocols (MakerDAO vaults, lending protocols) safely assume `virtual_price` is a **trustworthy, manipulation-resistant oracle**.

### What Actually Happens (Bug)

* Curve contracts allowed **reentrancy on read-only functions** (i.e., functions that don't directly mutate state).

* During a reentrant call, the state seen by `get_virtual_price` could reflect **partially updated values**:

  * Example: `D` has been updated but `totalSupply` hasn't, or vice versa.
  * Result: The ratio `D / totalSupply` can appear **artificially high or low**.

* Attackers can exploit this **flash-oracle window** to trick dependent protocols that read the manipulated `virtual_price` during their logic.

### Why This Matters

* `get_virtual_price` was **heavily used as an oracle**.
* Many DeFi protocols assumed **monotonicity** (value only increases). Breaking this allowed attackers to:

  * Artificially inflate LP token values.
  * Borrow more against collateral.
  * Bypass safety checks in vaults and lending protocols.
* The vulnerability was **systemic**: multiple major protocols inherited risk from the same source (Curve pools).

### Concrete Walkthrough (Alice & Mallory)

* **Setup**: MakerDAO uses `get_virtual_price` to value Curve LP tokens as vault collateral.

* **Mallory attack**: Mallory triggers a read-only reentrancy sequence:

  1. Calls into Curve pool.
  2. During reentrancy, Curve updates `D` but LP `totalSupply` hasn't caught up.
  3. `get_virtual_price = D / totalSupply` returns an **artificially inflated number**.

* **Alice protocol (MakerDAO)**: Maker queries Curve for collateral valuation:

  * Sees inflated LP value.
  * Believes Mallory's collateral is worth more than it is.
  * Lets Mallory borrow more DAI than she should.

* After reentrancy finishes, the values normalize, but Maker has already accepted the inflated collateral → Mallory gains an unfair loan.

> **Analogy**: Imagine a stock exchange where "shares outstanding" updates once per day, but "company valuation" updates instantly. For a brief moment, the ratio (valuation ÷ shares) looks much higher, tricking investors into thinking each share is worth more than it really is.

## Vulnerable Code Reference

### 1) Virtual price calculation in Curve

```solidity
function get_virtual_price() external view returns (uint256) {
    return D * 1e18 / totalSupply;
}
```

*Assumption*: `D` and `totalSupply` are consistent.
*Reality*: Reentrancy could desync them temporarily.

### 2) No protection against read-only reentrancy

```solidity
// No reentrancy guard applied here
function get_virtual_price() external view returns (uint256) {
    ...
}
```

Protocols called this in **sensitive contexts** (e.g., vault health checks).

## Recommended Mitigation

1. **Reentrancy protection even for view functions**:

   * Introduce **reentrancy guards** for oracle reads.
   * Or compute values in a way that guarantees consistency (snapshot state).

2. **Protocol-side defenses**:

   * Don't rely on a single call to `get_virtual_price`.
   * Use **TWAPs (time-weighted averages)** to smooth out anomalies.
   * Add **bounds checks** (value can't drop or increase more than X% per block).

3. **Curve fix**: They patched affected pools to prevent read-only reentrancy manipulations.

## Pattern Recognition Notes

* **Read-Only Reentrancy**: Even if no state mutation happens, an inconsistent intermediate state can still be observed.
* **Oracle Fragility**: When protocols rely on "increasing only" invariants, attackers look for ways to **break monotonicity**.
* **Cross-Protocol Risk**: Oracle bugs propagate far beyond the original protocol (Curve → MakerDAO, Enzyme, Abracadabra, etc.).
* **Invariant Assumption Break**: Assumptions like "this value can never decrease" should always be validated with guardrails in dependent systems.

## Quick Recall (TL;DR)

* **Bug**: `get_virtual_price` could be manipulated via read-only reentrancy.
* **Impact**: Inflated/deflated LP token value → downstream protocols misprice collateral.
* **Fix**: Add reentrancy protection + protocol-side validation (TWAPs, bounds checks).
