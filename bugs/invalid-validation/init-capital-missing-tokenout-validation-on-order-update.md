# Arbitrary `tokenOut` Manipulation via Missing Validation in `updateOrder`

* **Severity**: High
* **Source**: [Code4rena](https://github.com/code-423n4/2024-01-init-capital-invitational/blob/main/contracts/hook/MarginTradingHook.sol#L504-L526)
* **Affected Contract**: [MarginTradingHook.sol](https://github.com/code-423n4/2024-01-init-capital-invitational/blob/main/contracts/hook/MarginTradingHook.sol#L504-L526)
* **Vulnerability Type**: Invalid Validation / Missing Input Validation / State Invariant Violation / Front-running

## Summary

`MarginTradingHook` allows users to create **Take Profit** and **Stop Loss** orders for their margin positions. During order creation, the contract correctly validates that the requested `tokenOut` is either the position's **base asset** or **quote asset**, ensuring that orders can only settle in one of the two assets involved in the margin trade.

However, when an existing order is modified through `updateOrder()`, this validation is missing. As a result, the order owner can change `tokenOut` to **any ERC20 token address**, breaking the invariant established during order creation.

Since `fillOrder()` blindly trusts the stored `order.tokenOut`, a malicious order owner can front-run an executor's transaction and replace the expected output token with a completely different, potentially much more valuable token. If the executor has already approved the hook to spend that token, the hook will transfer the substituted token from the executor to the attacker.

## A Better Explanation (With Simplified Example)

### Intended Behavior

Suppose Alice opens a **Long ETH** position.

Her position consists of:

```
Collateral : ETH
Borrowed   : USDC
```

Since this position only involves **ETH** and **USDC**, any order should only be allowed to settle in one of these two assets.

During order creation, the contract enforces exactly this:

```solidity
_require(
    _tokenOut == marginPos.baseAsset ||
    _tokenOut == marginPos.quoteAsset,
    Errors.INVALID_INPUT
);
```

So Alice may choose:

```
✓ ETH
✓ USDC
```

but not

```
✗ WBTC
✗ LINK
✗ UNI
✗ Any unrelated token
```

This keeps every order internally consistent with its associated margin position.

### What Actually Happens (Bug)

The validation only exists inside `_createOrder()`.

When the order is later modified through `updateOrder()`, the contract updates

```solidity
order.tokenOut = _tokenOut;
```

without verifying whether the new token is still one of the position's valid assets.

Therefore, an order originally created as

```
tokenOut = ETH
```

can later become

```
tokenOut = WBTC
```

or literally any ERC20 token address.

The order has now entered an invalid state, but nothing prevents it from being executed.

### Why This Matters

The execution logic inside `fillOrder()` completely trusts the stored value:

```solidity
IERC20(order.tokenOut).safeTransferFrom(
    msg.sender,
    order.recipient,
    amtOut
);
```

Notice that it never validates `order.tokenOut` again.

Instead, it simply transfers whatever token is stored inside the order.

If an attacker changes `tokenOut` immediately before an executor fills the order, the executor unknowingly transfers the substituted token instead of the expected trading asset.

If the executor has previously approved the hook contract to spend that token (for example because it regularly executes many different orders), the hook successfully transfers the valuable token directly to the attacker.

In other words, the contract assumes that **all stored orders remain valid forever**, but `updateOrder()` allows users to violate that assumption.

### Concrete Walkthrough (Alice & Bob)

Suppose Alice has the following margin position:

```
Collateral : ETH
Borrowed   : USDC
```

She creates a Take Profit order:

```
tokenOut = ETH
Trigger = $3000
```

The order passes validation because ETH is a valid asset for this position.

---

Later, Bob (an order executor bot) notices that the trigger price has been reached and submits:

```
fillOrder(orderId)
```

Before Bob's transaction is mined, Alice sees it in the mempool.

She quickly submits:

```
updateOrder(
    tokenOut = WBTC
)
```

with a higher gas price.

The block executes as:

```
1. updateOrder()
2. fillOrder()
```

Now the stored order becomes:

```
tokenOut = WBTC
```

When `fillOrder()` executes, it performs:

```solidity
IERC20(WBTC).safeTransferFrom(
    Bob,
    Alice,
    amtOut
);
```

instead of

```solidity
IERC20(ETH).safeTransferFrom(...)
```

If Bob had previously approved the hook contract to spend his WBTC, Alice successfully receives WBTC even though her margin position has nothing to do with WBTC.

The entire exploit succeeds because `fillOrder()` trusts a value whose validity was silently broken after order creation.

> **Analogy:** Imagine an online food delivery order where the restaurant verifies you ordered a pizza when placing the order. Later, before the delivery driver picks it up, you secretly edit the order to request an expensive steak. Since the driver only looks at the current order details and assumes they were already verified, they deliver the steak even though only a pizza should have been allowed.

## Vulnerable Code Reference

### 1. Order creation correctly validates `tokenOut`

```solidity
MarginPos memory marginPos = __marginPositions[initPosId];

_require(
    _tokenOut == marginPos.baseAsset ||
    _tokenOut == marginPos.quoteAsset,
    Errors.INVALID_INPUT
);
```

Only the base asset or quote asset of the position may be used.

### 2. `updateOrder()` skips the same validation

```solidity
order.triggerPrice_e36 = _triggerPrice_e36;
order.limitPrice_e36 = _limitPrice_e36;
order.collAmt = _collAmt;
order.tokenOut = _tokenOut;
```

The contract directly overwrites `tokenOut` without verifying that it remains a valid asset.

### 3. `fillOrder()` blindly trusts the stored value

```solidity
IERC20(order.tokenOut).safeTransferFrom(
    msg.sender,
    order.recipient,
    amtOut
);
```

Execution assumes `order.tokenOut` is still valid, even though it may have been arbitrarily modified.

## Recommended Mitigation

### 1. Reapply validation inside `updateOrder()` (Primary Fix)

```solidity
_require(
    _tokenOut == marginPos.baseAsset ||
    _tokenOut == marginPos.quoteAsset,
    Errors.INVALID_INPUT
);
```

Both creation and update paths should enforce identical validation rules.

### 2. Preserve Object Invariants

Any field validated during creation should remain valid throughout the object's lifetime.

Whenever mutable state is modified, revalidate every invariant that depends on the updated fields.

### 3. Consider Defensive Validation During Execution

Although validating during updates is sufficient, `fillOrder()` may also defensively verify that the stored `tokenOut` still matches the position's supported assets before executing transfers.

This provides an additional layer of protection against unexpected state corruption.

### 4. Add Regression Tests

Unit tests should verify:

* Order creation rejects unsupported tokens.
* Order updates reject unsupported tokens.
* An existing valid order cannot be mutated into an invalid state.
* Front-running with `updateOrder()` cannot alter settlement assets.

## Pattern Recognition Notes

* **Creation vs Update Validation Mismatch**: Validations performed during object creation must also be enforced whenever the same fields can be modified later.
* **Broken State Invariants**: The invariant "`tokenOut` must always equal either the base asset or quote asset" is established during creation but violated during updates.
* **Trusting Mutable State**: `fillOrder()` assumes stored state remains valid without rechecking it. This is only safe if every mutation path preserves the invariant.
* **Front-running Mutable Parameters**: Any user-controlled field that affects settlement should be examined carefully for last-minute modifications before execution.
* **Business Logic Validation**: Correct access control alone is insufficient. Even authorized users must not be allowed to transition objects into invalid states.

## Quick Recall (TL;DR)

* **Bug**: `updateOrder()` allows `tokenOut` to be changed to any ERC20 token because it omits the validation performed during order creation.
* **Impact**: An order owner can front-run an executor, replace the expected settlement asset with another approved token, and cause the executor to transfer unintended assets.
* **Root Cause**: Validation exists during creation but is missing during updates, breaking the invariant that `tokenOut` must always be either the position's base asset or quote asset.
* **Fix**: Apply the same `tokenOut` validation inside `updateOrder()` (and optionally perform defensive validation during execution).
