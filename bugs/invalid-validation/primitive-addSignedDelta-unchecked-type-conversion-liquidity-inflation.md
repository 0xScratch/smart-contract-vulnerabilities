# Unchecked Type Conversion in `addSignedDelta` Leads to Liquidity Inflation & Fund Theft

* **Severity**: High
* **Source**: [Trail of Bits (Solodit)](https://solodit.cyfrin.io/issues/risk-of-token-theft-due-to-unchecked-type-conversion-trailofbits-none-primitive-hyper-pdf)
* **Affected Function**: `addSignedDelta` (Assembly-based arithmetic)
* **Vulnerability Type**: Integer Overflow / Type Confusion / Data Validation Failure

## Summary

The `addSignedDelta` function is used to update liquidity balances across multiple core flows (`allocate`, `unallocate`, `unstake`). It attempts to safely handle signed arithmetic using inline assembly and detect overflow/underflow conditions.

However, the implementation incorrectly assumes **128-bit arithmetic**, while EVM assembly operates on **256-bit values**. As a result:

* Overflow checks **never trigger**
* Attackers can supply large positive `delta` values
* Liquidity balances can be **artificially inflated without depositing assets**

This enables attackers to **mint fake liquidity** and subsequently **withdraw real funds from the pool**, leading to full pool drain.

## A Better Explanation (With Simplified Example)

### Intended Behavior

1. Users deposit assets → receive liquidity balance (`input`)
2. When modifying liquidity:

   * `delta > 0` → increase liquidity
   * `delta < 0` → decrease liquidity
3. Function ensures:

   * No overflow when adding
   * No underflow when subtracting

### What Actually Happens (Bug)

* The function uses assembly `add` and `sub`, which operate on **256-bit integers**
* Inputs are declared as `uint128` and `int128`, but **types are ignored in assembly**
* Overflow detection relies on this check:

```solidity
if (output < input) revert;
```

👉 This only works if arithmetic wraps (i.e., in 128-bit)

But:

* Adding two 128-bit numbers **cannot overflow a 256-bit value**
* Therefore:

  * No wraparound occurs
  * Condition `output < input` is **never true**
  * Overflow is **never detected**

### Why This Matters

* The system trusts `addSignedDelta` for safety
* But it silently allows invalid state transitions
* Attackers can:

  * Increase their liquidity arbitrarily
  * Withdraw funds they never deposited

### Concrete Walkthrough (Alice & Eve)

#### Scenario: Liquidity Inflation Attack

* **Alice** deposits 100 USDC → gets 100 liquidity
* **Eve (attacker)** starts with 0 liquidity

**Step 1: Exploit overflow logic**

* Eve calls a function that internally uses:

```solidity
addSignedDelta(0, VERY_LARGE_DELTA)
```

* Assembly computes:

  * `output = 0 + huge_number`
  * No overflow (256-bit safe)
  * No revert

👉 Eve now has **fake liquidity**

**Step 2: Withdraw funds**

* Eve calls `unallocate`
* Protocol believes she owns liquidity
* She withdraws **real assets from pool**

👉 Result:

> Eve drains funds that belong to Alice and other users

### Analogy

> Imagine a bank system that checks for overflow only if balances "wrap around."
> But the system secretly upgraded to a much larger number system where wrap never happens.
> So the fraud detection **never triggers**, and attackers can mint money freely.

## Vulnerable Code Reference

```solidity
function addSignedDelta(uint128 input, int128 delta) pure returns (uint128 output) {
    bytes memory revertData = abi.encodeWithSelector(InvalidLiquidity.selector);
    assembly {
        switch slt(delta, 0)

        // Negative delta (subtraction)
        case 1 {
            output := sub(input, add(not(delta), 1))
            switch slt(output, input)
            case 0 {
                revert(add(32, revertData), mload(revertData))
            }
        }

        // Positive delta (addition)
        case 0 {
            output := add(input, delta)
            switch slt(output, input)
            case 1 {
                revert(add(32, revertData), mload(revertData))
            }
        }
    }
}
```

## Root Cause

* **Type mismatch between Solidity and EVM assembly**

  * Solidity: `uint128`, `int128`
  * Assembly: always `uint256` / `int256`
* Overflow detection logic assumes **128-bit wraparound**
* But actual computation occurs in **256-bit space**
* Therefore:

  * Overflow condition is **mathematically impossible**

## Recommended Mitigation

### 1. Use Safe High-Level Solidity (Preferred)

```solidity
function addSignedDelta(uint128 input, int128 delta) pure returns (uint128) {
    if (delta < 0) {
        uint128 absDelta = uint128(-delta);
        require(input >= absDelta, "Underflow");
        return input - absDelta;
    } else {
        uint128 deltaUint = uint128(delta);
        uint128 result = input + deltaUint;
        require(result >= input, "Overflow");
        return result;
    }
}
```

### 2. If Using Assembly → Enforce Bounds Explicitly

* Manually check:

  * `result <= type(uint128).max`
* Do NOT rely on wraparound behavior

### 3. Add Strong Testing

* Unit tests:

  * max values
  * boundary values (2^128 - 1)
* Fuzz tests:

  * random delta inputs
* Property:

  * "Liquidity cannot increase without corresponding asset input"

### 4. Avoid Mixed Bit-Width Assumptions

* Never assume sub-256-bit behavior inside assembly
* Explicitly clamp or validate values

## Pattern Recognition Notes

### 🔴 Assembly + Smaller Types = Red Flag

If you see:

```solidity
uint128 / int128
```

inside:

```solidity
assembly { ... }
```

👉 Immediately question correctness

### 🔴 Overflow Detection via Comparison

Pattern:

```solidity
if (output < input)
```

👉 Only valid if:

* arithmetic is done in same bit-width

### 🔴 Trusted Internal Math Functions

If a protocol relies on:

> "this internal function guarantees safety"

👉 That function becomes a **critical attack surface**

### 🔴 Silent Precision Expansion

EVM silently:

* expands smaller types → 256-bit
* breaks assumptions without warning

## Quick Recall (TL;DR)

* **Bug**: Overflow detection assumes 128-bit math but runs in 256-bit assembly
* **Impact**: Liquidity can be artificially inflated
* **Exploit**: Fake liquidity → withdraw real funds
* **Fix**: Use safe Solidity or enforce explicit bounds in assembly
