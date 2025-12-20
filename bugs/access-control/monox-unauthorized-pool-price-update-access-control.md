# Unauthorized Pool Price Manipulation via Missing Access Control in Monoswap

* **Severity**: High
* **Source**: [Solodit (Halborn)](https://solodit.cyfrin.io/issues/role-based-access-control-missing-halborn-monox-monox-markdown)
* **Affected Contract**: `Monoswap.sol`
* **Vulnerability Type**: Access Control / Business Logic Flaw

## Summary

The **MonoX** protocol contained a critical **access control vulnerability** in the `updatePoolPrice` function within `Monoswap.sol`.
Although pool price updates are a **highly sensitive administrative action**, the vulnerable version of the contract **did not properly restrict access**, allowing **any external user** to arbitrarily update pool prices.

This violated the **principle of least privilege** and enabled attackers to manipulate pool prices, potentially leading to **fund loss, unfair trades, and protocol instability**. The issue was later fixed by introducing stricter access control and additional roles.

## A Better Explanation (With Simplified Example)

### Intended Behavior

In MonoX:

1. Each token has an associated **liquidity pool price**
2. Prices are normally adjusted via trades
3. If no trading occurs for a long period, an **admin-controlled function** can manually update the pool price to avoid stale pricing
4. This function is **meant to be callable only by trusted entities** (owner or governance roles)

In other words:

> **Users trade → admins maintain system health**

### What Actually Happened (Bug)

In the vulnerable version of `Monoswap.sol`:

* `updatePoolPrice` was declared as a `public` function
* It **lacked effective access control**
* Any address could call it and set a new pool price

This meant:

* No ownership check
* No role validation
* No governance restriction

So instead of:

> "Only trusted actors can update prices"

It became:

> **"Anyone can update prices."**

### Why This Matters

Pool price directly affects:

* how many tokens a trader receives
* whether an arbitrage opportunity exists
* how value flows in or out of the pool

If an attacker can set prices freely:

* they can manipulate the pool
* trade at unfair prices
* extract value from honest users

This is **not theoretical** — it is immediately exploitable.

### Concrete Walkthrough (Alice & Mallory)

**Setup**:

* Token X pool exists
* Current pool price is fair
* `updatePoolPrice` is publicly callable

**Mallory (attacker)**:

1. Calls `updatePoolPrice(tokenX, fakeLowPrice)`
2. Pool price is drastically lowered
3. Mallory buys token X very cheaply
4. Price is later corrected (or remains manipulated)

**Alice (honest user)**:

* Trades using manipulated price
* Receives fewer tokens than expected
* Suffers direct financial loss

> **Analogy**:
> Imagine a stock exchange where *anyone* can change the displayed stock price before placing a trade. Whoever changes it first wins — everyone else loses.

## Vulnerable Code Reference

### Missing or Ineffective Access Control on Price Update

**Vulnerable version (conceptual)**:

```solidity
function updatePoolPrice(address _token, uint112 _newPrice) public {
    require(_newPrice > 0, "Monoswap: zeroPriceNotAccept");
    // update pool price
}
```

* Function is `public`
* No `onlyOwner`
* No role-based restriction
* Anyone can manipulate prices

### Fixed Version (Post-Mitigation)

```solidity
function updatePoolPrice(address _token, uint112 _newPrice) public onlyOwner {
    require(_newPrice > 0, "Monoswap: zeroPriceNotAccept");
}
```

* Restricted to owner (and later refined into role-based permissions)
* Unauthorized callers are blocked

## Recommended Mitigation

1. **Enforce strict access control on sensitive functions**

   * Pool price updates must be restricted to:

     * owner
     * governance
     * or a dedicated price-manager role

2. **Adopt Role-Based Access Control (RBAC)**

   Instead of a single omnipotent owner:

   * `PRICE_UPDATER_ROLE`
   * `FEE_MANAGER_ROLE`
   * `EMERGENCY_ROLE`

3. **Audit all administrative setters**

   Any function that:

   * updates prices
   * modifies fees
   * changes pool status
     must be reviewed for correct access restrictions.

4. **Add regression tests**

   * Assert that unauthorized users **cannot** call admin functions
   * Tests should explicitly fail when access checks are removed

## Pattern Recognition Notes

* **Missing Modifier on Critical Function**
  A single forgotten `onlyOwner` can compromise the entire protocol.

* **Comment vs Enforcement Mismatch**
  Comments stated "only owner callable", but the code did not enforce it — a common real-world audit pitfall.

* **Price Control Is High-Risk**
  Any function that controls price, oracle input, or exchange rate should be treated as **critical severity by default**.

* **Least Privilege Violation**
  Giving unrestricted access to sensitive operations dramatically increases attack surface.

* **Historical Code Review Matters**
  Always verify **pre-fix versions** when reading audit reports — patched code may hide the original issue.

## Quick Recall (TL;DR)

* **Bug**: `updatePoolPrice` lacked proper access control
* **Impact**: Anyone could manipulate pool prices
* **Risk**: Fund loss, unfair trades, protocol instability
* **Fix**: Introduce strict ownership and role-based access control
