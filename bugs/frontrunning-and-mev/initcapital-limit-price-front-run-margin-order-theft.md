# Limit Price Manipulation via Front-Running Allows Theft in Margin Order Filling

* **Severity**: High
* **Source**: [Code4rena](https://github.com/code-423n4/2024-01-init-capital-invitational-findings/issues/22)
* **Affected Contract**: [`MarginTradingHook.sol`](https://github.com/code-423n4/2024-01-init-capital-invitational/blob/main/contracts/hook/MarginTradingHook.sol)
* **Vulnerability Type**: Invalid Validation / Slippage Manipulation / MEV Front-Run Theft

## Summary

`MarginTradingHook` allows users to create automated margin exit orders (stop-loss or take-profit). Each order includes a **limitPrice_e36**, which is used when computing `amtOut`, the amount of tokens the executor must pay the order creator during `fillOrder`.

However, the protocol **allows the order creator to freely update limitPrice_e36 at any time** using `updateOrder()`, and enforcement occurs **only in the executor's fill transaction**.

Because `amtOut` is calculated directly from limitPrice_e36—with all rounding favoring the order owner—a malicious user can **front-run** an honest executor's `fillOrder` transaction, **alter limitPrice_e36**, and cause the contract to compute a much larger `amtOut`. The executor then unknowingly transfers an inflated amount of tokens to the attacker.

This enables **theft of tokens from order executors** with no protocol-level prevention.

## A Better Explanation (With Simplified Example)

### Intended Behavior

Margin orders work like this:

1. User creates a stop-loss or take-profit order with:

   * trigger price
   * limit price
   * collateral amount
   * desired output token
2. When market hits the trigger price, **executors** (searchers/bots) call `fillOrder()`.
3. Contract computes `amtOut` using a formula that protects the order owner from slippage:

   ```solidity
   amtOut = ... based on limitPrice_e36;
   ```

4. Executor:

   * sends tokens to repay user's debt
   * transfers `amtOut` tokens to recipient
   * receives collateral

This is supposed to be a **fair execution** where executor profits from getting collateral in exchange for doing work.

### What Actually Happens (Bug)

The creator can modify their order's limit price:

```solidity
function updateOrder(...) external {
    order.limitPrice_e36 = _limitPrice_e36;
}
```

`fillOrder()` does **not** check whether this updated price is reasonable, unchanged, or within slippage bounds.

During `fillOrder`, the contract computes:

```solidity
amtOut = fn(limitPrice_e36)    // large dependency on limit price
```

Thus:

1. Executor sends a transaction to fill an order.
2. Order creator front-runs with `updateOrder()`:

   * Sets a **very favorable limit price** that makes `amtOut` huge.
3. Executor's `fillOrder` uses the **new**, malicious price.
4. Executor pays significantly more to the attacker.
5. Attacker receives excess tokens → **theft**.

There is no protection, timestamping, or min/max validation.

### Why This Matters

* **Direct economic theft** from honest executors.
* Attack is **cheap**, requires **no special tooling**, and is **100% reliable**.
* Completely breaks the trust model around margin order execution.
* Liquidation/searcher networks will avoid filling orders → protocol functionality degrades.

## Concrete Walkthrough (Alice & Mallory)

### Setup

* Mallory creates a take-profit order with:

  * small collateral
  * normal limit price
* Executor bot sees order is ready to fill and sends `fillOrder(orderId)`.

### Attack

1. Mallory watches mempool for an incoming `fillOrder()`.
2. Mallory broadcasts `updateOrder()` before executor's tx.
3. Mallory sets `limitPrice_e36` to a **maliciously high number**.
4. Executor's transaction processes with **inflated amtOut**:

   ```solidity
   amtOut = (collTokenAmt * limitPrice_e36) / 1e36 ...  // huge
   ```

5. Executor transfers massive tokens to Mallory → losing money.
6. Mallory's order is marked `Filled` → theft complete.

> **Analogy**: A user places a sell order on an exchange, sees someone about to buy it, and instantly changes the price to something insane one microsecond before the fill. The buyer is forced to honor the new price.

## Vulnerable Code Reference

### 1) amtOut calculation depends entirely on `limitPrice_e36`

(Excerpt — all branches use limitPrice_e36)

```solidity
amtOut = collTokenAmt - repayAmt * ONE_E36 / _order.limitPrice_e36;
amtOut = collTokenAmt - (repayAmt * _order.limitPrice_e36 / ONE_E36);

amtOut = (collTokenAmt * _order.limitPrice_e36).ceilDiv(ONE_E36) - repayAmt;
amtOut = (collTokenAmt * ONE_E36).ceilDiv(_order.limitPrice_e36) - repayAmt;
```

### 2) Order creator can update limit price at any time

```solidity
function updateOrder(...) external {
    order.limitPrice_e36 = _limitPrice_e36;
}
```

### 3) fillOrder uses *whatever* limitPrice_e36 is stored—no validation

```solidity
(uint amtOut, , ) = _calculateFillOrderInfo(order, marginPos, collToken);
// no check that limit price is unchanged or within acceptable bounds
```

## Recommended Mitigation

### 1. **Validate limit price during fill**

Require executors to pass `expectedLimitPrice_e36`, and revert if stored price differs:

```solidity
function fillOrder(uint orderId, uint expectedLimitPrice) external {
    require(order.limitPrice_e36 == expectedLimitPrice, "Limit price changed");
}
```

This eliminates all front-running attacks.

### 2. **Freeze price after creation**

Disallow modifying limit price entirely once an order becomes active:

```solidity
// in updateOrder
require(order.status == Draft, "Cannot modify an active order");
```

### 3. **Store immutable limits at order creation**

Require minPrice / maxPrice bounds, and enforce:

```solidity
require(order.limitPrice_e36 >= min && order.limitPrice_e36 <= max);
```

### 4. **Add slippage sanity checks**

Cap permissible price deviation from oracle-derived fair price.

## Pattern Recognition Notes

* **User-adjustable parameters used in settlement calculations** are always exploitable if not locked or validated.
* **Slippage protection cannot be user-controlled** once trading execution becomes adversarial (MEV environment).
* **Front-running risk** exists whenever:

  * Users can modify state
  * Executors depend on that state
  * Execution ordering is not guaranteed
* **MEV-aware designs** must treat all inputs as adversarial and immutable after order creation.
* **Executor incentive models fail** if counterparties can manipulate payout variables after commitment.

## TL;DR

**Bug**: Order creators can update limitPrice_e36 after creation.
**Impact**: They can front-run fillOrder and force executors to overpay them → direct theft.
**Fix**: Lock limit price or require executors to provide the expected limit value during fill.
