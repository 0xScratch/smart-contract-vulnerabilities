# Cumulative Rounding Refund Error in TWAMM Leads to Pool-Wide DoS and Value Leakage

* **Severity**: Medium
* **Source**: [Code4rena](https://code4rena.com/audits/2025-11-ekubo/submissions/F-270)
* **Affected Contract**: [TWAMM.sol](https://github.com/code-423n4/2025-11-ekubo/blob/bbc87eb26d73700cf886f1b3f06f8a348d9c6aef/src/extensions/TWAMM.sol#L301-L312)
* **Vulnerability Type**: Business Logic Flaw / Precision & Rounding Error / Denial of Service (DoS)

## Summary

The TWAMM extension contains a **rounding-direction bug** when calculating the *remaining sell amount* during order updates or cancellations. Specifically, the contract **rounds up** a value that represents how much the user *still owes*, which should instead be **rounded down**.

This causes the protocol to **over-refund users by ~1 wei per order update**. While negligible for a single order, an attacker can **amplify the effect** by creating and canceling many small orders. The cumulative rounding error results in **internal accounting debt** inside the TWAMM extension.

Once this debt exceeds the extension's saved balances, **future TWAMM executions revert**, freezing all pool interactions (swaps, LP updates, fee collection) until external intervention.

## A Better Explanation (With Simplified Example)

### Intended Behavior

When a user **updates or cancels** a TWAMM order:

1. The contract computes how much time remains in the order (`durationRemaining`).
2. It calculates:

   * `remainingSellAmount`: how much token the user would still sell if the order stayed unchanged.
   * `amountRequired`: how much token is needed for the *new* (updated) sale rate.
3. The difference (`amountDelta`) determines whether the user:

   * receives a refund, or
   * must add more tokens.
4. The pool's internal balances are updated to reflect this change.

**Key invariant**:

> The protocol must never refund *more* than the user is actually entitled to.

### What Actually Happens (Bug)

Both values are computed with `roundUp: true`:

```solidity
uint256 amountRequired = computeAmountFromSaleRate({
    saleRate: saleRateNext,
    duration: durationRemaining,
    roundUp: true
});

uint256 remainingSellAmount = computeAmountFromSaleRate({
    saleRate: saleRate,
    duration: durationRemaining,
    roundUp: true  // ❌ should be false
});
```

However:

* `remainingSellAmount` represents **user liability**
* Rounding it **up** assumes the user owes *more* than reality
* When subtracting, the protocol **refunds slightly too much**

This creates a **systematic 1-wei over-refund** whenever fractional values are involved.

### Why This Matters

* TWAMM orders often involve **tiny sale rates** spread over long durations.
* Fractional math is common.
* The rounding error occurs **per order**, not per user.

This allows an attacker to:

* Split one logical order into many small ones
* Cancel each order early
* Accumulate rounding refunds that exceed protocol reserves

### Concrete Walkthrough (Alice & Mallory)

**Setup**

* TWAMM pool initialized
* Sale rates active
* Extension accounting is balanced

**Mallory attack**

1. Mallory creates **256 tiny TWAMM orders**, each with:

   * very small `saleRate`
   * long duration
2. Before the orders expire, Mallory cancels all of them.

For each cancellation:

* `remainingSellAmount` is rounded **up**
* Protocol over-refunds ≈ **1 wei**
* Fee mechanism absorbs the refund → Mallory doesn't profit directly

**Result**

* TWAMM extension accumulates ≈ **255 wei debt**
* Internal saved balances are now insufficient

**Aftermath**

* When TWAMM tries to execute the *next legitimate order*:

  * `CORE.updateSavedBalances` reverts
  * `SavedBalanceOverflow` is triggered
  * All pool interactions freeze

> **Analogy**:
> Imagine a cashier who rounds *change owed* **up** instead of down. One customer loses nothing, but 256 customers later, the cash drawer is empty and the store must close.

## Vulnerable Code Reference

### Incorrect rounding when computing remaining user liability

```solidity
// TWAMM.sol#L301-L312

uint256 amountRequired = computeAmountFromSaleRate({
    saleRate: saleRateNext,
    duration: durationRemaining,
    roundUp: true
});

uint256 remainingSellAmount = computeAmountFromSaleRate({
    saleRate: saleRate,
    duration: durationRemaining,
    roundUp: true // ❌ should be false
});
```

### Resulting debt breaks future TWAMM execution

```solidity
// Later during virtual order execution
CORE.updateSavedBalances(...); // reverts due to insufficient balance
```

## Impact

### Primary Impact: Pool-Wide Denial of Service

* Once TWAMM accounting becomes negative:

  * All swaps
  * All LP updates
  * All TWAMM order executions
    **revert**
* Pool remains frozen until manual recovery.

### Secondary Impact: Economic Leakage (Chain-Dependent)

If an attacker:

* Creates their own pool
* Is the sole LP
* Uses high-value, low-decimals tokens
* Operates on a cheap chain

Then:

* 1 wei loss per interaction can exceed gas cost
* Attack becomes **economically profitable**

## Recommended Mitigation

### 1️⃣ Fix rounding direction (primary)

```solidity
uint256 remainingSellAmount = computeAmountFromSaleRate({
    saleRate: saleRate,
    duration: durationRemaining,
    roundUp: false // ✅ correct
});
```

This ensures:

* Refunds are **rounded down**
* Protocol never over-refunds
* Accounting remains conservative

### 2️⃣ Add invariant tests for cumulative rounding

* Multiple small orders
* Repeated cancellations
* Assert saved balances never go negative

### 3️⃣ Defensive accounting assertions

* Consider sanity checks that prevent TWAMM execution if internal debt exists
* Fail early with explicit errors rather than implicit balance overflows

## Pattern Recognition Notes

* **Refunds Must Round Down**: Any value representing *user entitlement* should use conservative rounding.
* **Cumulative Precision Errors**: Tiny per-action errors can become fatal when actions are repeatable.
* **Per-Order vs Aggregate Math**: Splitting one action into many can bypass assumptions made at aggregate level.
* **Internal Accounting Fragility**: Virtualized systems (like TWAMM) are especially sensitive to precision drift.
* **DoS via Accounting Debt**: Not all DoS attacks require reverts — silent balance erosion can be just as deadly.

## Quick Recall (TL;DR)

* **Bug**: `remainingSellAmount` is rounded **up** instead of down.
* **Exploit**: Many small orders → cumulative 1-wei over-refunds.
* **Impact**: TWAMM internal debt → pool-wide DoS.
* **Fix**: Round **down** when computing remaining user liability.
