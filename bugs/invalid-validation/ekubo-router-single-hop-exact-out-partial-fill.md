# Partial Fill Allowed in Single-Hop Exact-Out Swaps in Router

* **Severity**: Medium
* **Source**: [Code4rena](https://code4rena.com/audits/2025-11-ekubo/submissions/F-298)
* **Affected Contract**: [Router.sol](https://github.com/code-423n4/2025-11-ekubo/blob/main/src/Router.sol#L94-L149)
* **Vulnerability Type**: Input Validation / Broken Semantic Guarantee / Unexpected Partial Fill
* **Protocol**: Ekubo Protocol

## Summary

The `Router` contract supports **Exact-In** and **Exact-Out** swaps for both **single-hop** and **multi-hop** routes. While **multi-hop Exact-Out swaps correctly forbid partial fills**, **single-hop Exact-Out swaps do not**.

As a result, when a pool cannot satisfy the full requested output amount (due to insufficient liquidity, tick boundaries, or restrictive `sqrtRatioLimit`), the swap may execute **partially** and still succeed. The router only checks that the **input paid does not exceed the user's maximum**, but fails to enforce that the **exact requested output is received**.

This breaks the semantic guarantee of Exact-Out swaps and can lead to **silent failures**, **broken downstream assumptions**, and **economic loss** for users or integrating protocols.

## A Better Explanation (With Simplified Example)

### Intended Behavior (Exact-Out)

An Exact-Out swap means:

> "Give me **exactly X tokens out**, and I am willing to pay **up to Y tokens in**."

Guarantees:

* Output amount is **non-negotiable**
* Input amount is **capped**, not fixed

If the router cannot deliver **exactly X output**, the transaction **must revert**.

### What Actually Happens (Bug)

In **single-hop Exact-Out swaps**, the router:

* Calls `CORE.swap(...)`
* Receives a `PoolBalanceUpdate`
* **Only checks** that the input paid is within the user-specified maximum
* **Does NOT check** that the full requested output was actually delivered

Because the Ekubo Core allows **partial fills**, the pool may return a smaller output **and** a smaller input. Since the input is still within bounds, the router allows the swap to succeed — even though the user did not receive the exact amount they asked for.

### Concrete Walkthrough (Alice)

#### Setup

* Alice wants **exactly 100 token0**
* Alice is willing to pay **up to 101 token1**
* This is a **single-hop Exact-Out swap**

```text
Requested output: 100 token0
Max input:        101 token1
```

#### What Alice Expects

| Outcome                          | Should Succeed? |
| -------------------------------- | --------------- |
| Gets 100 token0, pays 100 token1 | ✅               |
| Gets 100 token0, pays 101 token1 | ✅               |
| Gets 49 token0, pays 51 token1   | ❌               |

#### What Actually Happens

Due to low liquidity or tick constraints, the pool returns:

```text
delta0 = -49   (user receives only 49 token0)
delta1 = +51   (user pays only 51 token1)
```

The router checks:

```text
51 ≤ 101 → OK
```

✅ Transaction succeeds
❌ Alice receives **less than the exact amount she requested**

### Why This Matters

* The router **violates Exact-Out semantics**
* Failure is **silent** — no revert, no warning
* Users may proceed assuming they received the exact amount
* Integrating contracts may rely on Exact-Out guarantees and break

> **Important**: This is not a direct theft of funds. The unused tokens remain with the user.
> The loss arises from **broken assumptions**, not from stolen assets.

## Vulnerable Code Reference

### 1) Single-Hop Exact-Out lacks full-fill enforcement

```solidity
(PoolBalanceUpdate balanceUpdate,) = _swap(...);

int128 amountCalculated =
    params.isToken1()
        ? -balanceUpdate.delta0()
        : -balanceUpdate.delta1();

if (amountCalculated < calculatedAmountThreshold) {
    revert SlippageCheckFailed(calculatedAmountThreshold, amountCalculated);
}
```

**Issue**:

* This only enforces **max input**
* It does **not** enforce `actualOutput == requestedOutput`

### 2) Contrast: Multi-Hop swaps correctly forbid partial fills

```solidity
if (update.delta0() != tokenAmount.amount)
    revert PartialSwapsDisallowed();
```

This invariant is **missing** for single-hop Exact-Out swaps.

## Why This Can Lead to Losses (Clarification)

There is **no immediate loss inside the swap** itself.

However, losses occur when:

1. **Downstream logic assumes exact output**

   * Debt repayment
   * Minting / burning
   * Unlock conditions
2. **Economic or timing assumptions break**

   * User retries at worse price
   * Opportunity cost
3. **Integrating protocols trust router semantics**

   * Skip post-swap balance checks
   * Execute unsafe follow-up actions

This makes the issue **Medium severity**, not Low.

## Recommended Mitigation

### 1️⃣ Enforce full output for Exact-Out single-hop swaps (preferred)

When `Exact-Out` and no explicit partial-fill intent:

```solidity
require(actualOutput == requestedOutput, "Partial fill not allowed");
```

This mirrors Uniswap V3 router behavior when `sqrtRatioLimit == 0`.

### 2️⃣ Optional: Explicit opt-in for partial fills

Add a user-controlled flag:

```solidity
bool allowPartialFill;
```

Default: `false`

### 3️⃣ Align semantics across swap types

Ensure **single-hop and multi-hop** Exact-Out swaps enforce the same invariant unless explicitly overridden.

## Pattern Recognition Notes

* **Broken Semantic Guarantee**
  When a function name or mode (e.g., *Exact-Out*) implies a strict invariant, failing to enforce it is a vulnerability — even if no funds are directly stolen.

* **Partial Fill Hazards**
  AMM cores may allow partial fills, but **routers must decide whether they are acceptable**.

* **Silent Failure > Loud Revert**
  Silent success with incorrect results is often more dangerous than a revert.

* **Router vs Core Responsibility Split**
  Core may be permissive; Router must be opinionated and enforce user intent.

* **Integration Risk**
  Routers are often reused or copied. Weak guarantees propagate bugs downstream.

## Quick Recall (TL;DR)

* **Bug**: Single-hop Exact-Out swaps allow partial fills.
* **Cause**: Router checks max input but not exact output.
* **Impact**: Users receive less than requested without revert.
* **Loss**: Contextual — broken assumptions, downstream failures.
* **Fix**: Enforce full output or require explicit partial-fill opt-in.
