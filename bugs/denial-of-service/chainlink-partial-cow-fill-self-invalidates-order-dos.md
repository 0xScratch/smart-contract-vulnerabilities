# Dust Partial Fills Can Self-Invalidate Remaining CoW Orders Until Workflow Refresh

* **Severity**: Medium
* **Source**: [Code4rena](https://code4rena.com/audits/2026-03-chainlink-payment-abstraction-v2/submissions/S-61)
* **Affected Contract**: [GPV2CompatibleAuction.sol](https://github.com/code-423n4/2026-03-chainlink/blob/main/src/GPV2CompatibleAuction.sol), [GPv2Trade.sol](https://github.com/code-423n4/2026-03-chainlink/blob/main/src/vendor/@cowprotocol/contracts/src/contracts/libraries/GPv2Trade.sol#L16-L27)
* **Vulnerability Type**: Denial of Service (DoS) / State Desynchronization / Partial-Fill Logic Error

## Summary

`GPV2CompatibleAuction` integrates with CoW Protocol by publishing signed orders that solvers can execute. The contract explicitly requires orders to be **partially fillable**, meaning solvers are allowed to execute only a portion of the posted order rather than the entire amount.

However, `isValidSignature()` validates orders using the **original full signed `sellAmount`**, while CoW settlement executes trades using a separate `executedAmount` field that may be significantly smaller.

As a result, a solver can perform a tiny partial fill, reducing the auction's token balance by a negligible amount. The original signed order then immediately becomes invalid because future signature validation still requires the contract to hold the full original `sellAmount`.

This effectively blocks all further fills of the remaining inventory until the next workflow refresh generates a new order matching the updated balance.

## A Better Explanation (With Simplified Example)

### Intended Behavior

The protocol wants to auction assets through CoW Protocol.

Suppose the auction contract holds:

```text
100,000 USDC
```

The workflow publishes a CoW order:

```text
Sell: 100,000 USDC
Buy: 5,000 LINK
Partially Fillable: Yes
```

Because the order is partially fillable, solvers should be able to execute:

```text
10,000 USDC
20,000 USDC
5,000 USDC
...
```

until the entire order is gradually filled.

This is the purpose of partial fills.

### What Actually Happens (Bug)

The contract validates orders using:

```solidity
if (order.sellAmount > assetInBalance) {
    revert InsufficientAssetInBalance(...);
}
```

Notice that validation uses:

```solidity
order.sellAmount
```

which is the **original signed amount**.

Meanwhile CoW settlement executes:

```solidity
trade.executedAmount
```

which may be much smaller.

For example:

```text
Signed order = 100,000 USDC
Executed amount = 1 USDC
```

After settlement:

```text
Auction balance:
100,000 USDC
↓
99,999 USDC
```

The original signed order still says:

```text
Sell 100,000 USDC
```

So future validations check:

```text
Do I still own 100,000 USDC?
```

The answer is now:

```text
No (only 99,999 remain)
```

Therefore the order immediately becomes invalid.

The protocol ends up invalidating its own remaining order inventory.

### Why This Matters

The protocol explicitly supports partial fills, but the validation logic assumes the full original quantity remains available forever.

This creates a contradiction:

```text
Partial fills are allowed
↓
Partial fill occurs
↓
Balance decreases
↓
Order becomes invalid
↓
Remaining inventory cannot be sold
```

The attacker only needs to execute an extremely small fill to trigger the issue.

As a result:

* Remaining auction inventory becomes temporarily unsellable through CoW.
* Solvers can no longer fill the posted order.
* Asset conversion into LINK is delayed.
* The workflow must publish a replacement order before trading can continue.

Because the attack cost can be as low as a single token unit, the griefing cost is extremely small relative to the disruption created.

### Concrete Walkthrough (Alice & Mallory)

#### Setup

Auction contract owns:

```text
100,000 USDC
```

Workflow publishes:

```text
Sell: 100,000 USDC
Buy: LINK
Partially Fillable: True
```

The order is valid.

#### Mallory Executes Dust Fill

Mallory acts as a CoW solver.

Instead of filling the entire order, she fills:

```text
1 USDC
```

which is allowed because the order is partially fillable.

After settlement:

```text
Auction balance:
100,000
↓
99,999
```

#### Alice Attempts Normal Fill

Alice attempts to fill the remaining order.

CoW calls:

```solidity
isValidSignature(...)
```

The contract computes:

```text
Required balance = 100,000
Actual balance   = 99,999
```

and reverts:

```solidity
InsufficientAssetInBalance(...)
```

#### Result

The order becomes unusable even though:

```text
99.999% of inventory still remains
```

The remaining balance cannot be sold through that order.

The workflow must later publish a new order reflecting:

```text
Sell: 99,999 USDC
```

before trading resumes.

> **Analogy:** Imagine a store posts a sign saying "Selling 100 apples." A customer legally buys 1 apple. The store now has 99 apples left. However, the store's validation rule says the sign is valid only if exactly 100 apples remain in inventory. As soon as the first apple is sold, the sign becomes invalid and nobody can buy the remaining 99 apples until management prints a new sign.

## Vulnerable Code Reference

### 1) Full Amount Validation During Signature Verification

```solidity
uint256 assetInBalance = order.sellToken.balanceOf(address(this));

if (order.sellAmount > assetInBalance) {
    revert InsufficientAssetInBalance(
        address(order.sellToken),
        order.sellAmount,
        assetInBalance
    );
}
```

The validation compares:

```text
Original signed sellAmount
```

against:

```text
Current token balance
```

This assumes the full order quantity always remains available.

### 2) Orders Are Explicitly Required To Be Partially Fillable

```solidity
if (!order.partiallyFillable) {
    revert OrderNotPartiallyFillable();
}
```

The protocol requires:

```text
partiallyFillable == true
```

meaning partial execution is expected behavior.

### 3) CoW Settlement Uses Executed Amounts

```solidity
struct Data {
    ...
    uint256 executedAmount;
    ...
}
```

Settlement consumes:

```text
executedAmount
```

rather than the full signed amount.

Therefore balance reductions naturally occur during valid partial fills.

## Root Cause

The protocol mixes two incompatible assumptions:

### Assumption #1

```text
Orders must be partially fillable.
```

### Assumption #2

```text
Order validation should require the full original sell amount to remain available.
```

These assumptions conflict.

Once any partial fill occurs:

```text
Current Balance < Original sellAmount
```

causing the order to invalidate itself.

## Recommended Mitigation

### 1. Use Fill-Or-Kill Orders

If validation requires the entire quantity to remain available:

```text
Disable partial fills entirely.
```

```solidity
order.partiallyFillable = false;
```

This keeps validation and execution semantics aligned.

### 2. Repost Orders After Every Partial Fill

Whenever a partial fill occurs:

```text
Old Order → Invalidate
Remaining Inventory → New Order
```

This ensures the signed amount always matches the remaining balance.

### 3. Validate Executable Quantity Instead Of Original Quantity

Rather than validating:

```solidity
order.sellAmount
```

the protocol should validate the quantity that remains executable.

This aligns signature validation with CoW's partial-fill model.

### 4. Enforce Minimum Fill Thresholds

Prevent griefing through microscopic fills:

```solidity
require(
    executedAmount >= MIN_FILL_SIZE,
    "fill too small"
);
```

This raises the attack cost significantly.

### 5. Add Regression Tests

Add tests covering:

* Partial fills of various sizes.
* Single-unit dust fills.
* Multiple sequential fills.
* Remaining inventory execution.
* Automatic order refresh behavior.

## Pattern Recognition Notes

* **Original State vs Current State Validation**: Systems frequently fail when validation relies on the originally signed state while execution mutates live state.

* **Partial Fill Mismatch**: Supporting partial fills requires validation logic that understands shrinking inventory over time.

* **Self-Invalidating Orders**: An operation can legally modify state in a way that causes future executions of the same object to fail.

* **State Desynchronization**: Off-chain order quantities and on-chain balances can diverge after partial execution.

* **Low-Cost Griefing**: Tiny actions that force expensive or disruptive protocol reactions often create attractive DoS vectors.

* **Workflow Refresh Dependency**: If a protocol requires periodic off-chain refreshes to recover from normal execution paths, temporary outages become possible whenever refreshes are delayed.

## Quick Recall (TL;DR)

* **Bug**: Orders are partially fillable, but `isValidSignature()` validates against the original full `sellAmount`.
* **Attack**: Execute a tiny partial fill that reduces the balance below the signed amount.
* **Impact**: The remaining order immediately becomes invalid and cannot be filled until a fresh workflow update creates a replacement order.
* **Root Cause**: Validation assumes static inventory while execution allows dynamic inventory reduction.
* **Fix**: Either disable partial fills, refresh orders after every fill, or validate remaining executable quantity instead of the original signed amount.
