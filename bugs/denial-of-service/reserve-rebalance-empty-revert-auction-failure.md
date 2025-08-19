# Dutch Auctions Can Fail to Settle Due to Silent Error Handling in BackingManager

* **Severity**: Medium
* **Source**: [Code4rena](https://github.com/code-423n4/2024-07-reserve-findings/issues/32) / [One Bug Per Day](https://www.onebugperday.com/v1/1437)
* **Affected Contract**: [BackingManager.sol](https://github.com/code-423n4/2024-07-reserve/blob/3f133997e186465f4904553b0f8e86ecb7bbacbf/contracts/p1/BackingManager.sol)
* **Vulnerability Type**: Error Handling / Misinterpretation of Error Conditions — DoS (Denial of Service)

## Original Bug Description

>## Lines of code
>
>[https://github.com/code-423n4/2024-07-reserve/blob/3f133997e186465f4904553b0f8e86ecb7bbacbf/contracts/p1/BackingManager.sol#L97](https://github.com/code-423n4/2024-07-reserve/blob/3f133997e186465f4904553b0f8e86ecb7bbacbf/contracts/p1/BackingManager.sol#L97)
>
>## Vulnerability details
>
>When a Dutch auction that originated from the backing manager receives a bid, it [calls](https://github.com/code-423n4/2024-07-reserve/blob/3f133997e186465f4904553b0f8e86ecb7bbacbf/contracts/plugins/trading/DutchTrade.sol#L222) `BackingManager.settleTrade()` to settle the auction immediately, which [attempts to chain](https://github.com/code-423n4/2024-07-reserve/blob/3f133997e186465f4904553b0f8e86ecb7bbacbf/contracts/p1/BackingManager.sol#L92) into another `rebalance()` call. This chaining is implemented using a try-catch block that attempts to catch out-of-gas errors.
>
>However, this pattern is not safe because empty error data does not always indicate an out-of-gas error. Other types of errors also return no data, such as calls to empty addresses casted as contracts and revert / require statements with no error message.
>
>The `rebalance()` function interacts with multiple external assets and performs several operations that can throw empty errors:
>
>1. In `basketsHeldBy()`, which calls `_quantity()`, which in turn calls `coll.refPerTok()` (this function should in theory [never revert](https://github.com/3docSec/2024-07-reserve/blob/main/docs/collateral.md#refpertok-reftok), but in case it interacts with the underlying ERC20, its implementation may have been upgraded to one that does).
>2. In `prepareRecollateralizationTrade()`, which calls `basketRange()`, which also calls `_quantity()`.
>3. In `tryTrade()` if a new rebalancing trade is indeed chained, which calls `approve()` on the token via `AllowanceLib.safeApproveFallbackToMax()`. This is a direct interaction with the token and hence cannot be trusted, especially if the possibility of [upgradeability](https://github.com/code-423n4/2024-07-reserve/tree/3f133997e186465f4904553b0f8e86ecb7bbacbf?tab=readme-ov-file#erc20-token-behaviors-in-scope) is considered.
>
>If any of these operations result in an empty error, the auction settlement will fail. This can lead to the Dutch auction being unable to settle at a fair price.
>
>Note: we have found [this](https://github.com/code-423n4/2023-06-reserve-findings/issues/8) finding pointing out the very same issue in a previous audit, but this report highlights a different root cause in where the error originates.
>
>## Impact
>
>Dutch auctions may fail to settle at the appropriate price or at all.
>
>## Proof of Concept
>
>1. A Dutch auction is initiated for rebalancing collateral.
>2. After some time, a bidder attempts to submit a bid at fair market value.
>3. BackingManager.settleTrade() is called by the trade contract.
>4. The rebalance() function is called within the try-catch block.
>5. The underlying ERC-20 of one of the collateral assets in the basket has an empty revert clause that currently throws when one of its functions is called.
>6. The catch block receives an empty error and reverts the transaction.
>
>## Tools Used
>
>Manual review
>
>## Recommended Mitigation Steps
>
>Avoid usage of this pattern to catch OOG errors in any functions that cannot revert and may interact with external contracts. Instead, in such cases always employ the [`_reserveGas()`](https://github.com/code-423n4/2024-07-reserve/blob/3f133997e186465f4904553b0f8e86ecb7bbacbf/contracts/p1/AssetRegistry.sol#L253) pattern that was iterated on to mitigate previous findings ([1](https://github.com/code-423n4/2023-01-reserve-findings/issues/254), [2](https://github.com/code-423n4/2023-02-reserve-mitigation-contest-findings/issues/73), [3](https://github.com/code-423n4/2023-06-reserve-findings/issues/7)) with a similar root cause. We have found no other instances in which this applies.
>
>## Assessed type
>
>DoS

## Summary

Within `BackingManager.settleTrade()`, the protocol chains a call to `rebalance()` inside a `try/catch` block, assuming that **empty error data signals an out-of-gas (OOG) failure**. However, in Solidity, many other failure modes—such as a call to an empty address, `require()` or `revert()` without a message—also return *empty* error data. Since `rebalance()` interacts with external assets and ERC-20 tokens (via `refPerTok()` and `approve()`), these operations may fail silently and cause the entire settlement to revert, resulting in Dutch auctions that fail **completely** or at unfair prices.

## A Better Explanation (With Simplified Example)

### Intended Behavior

1. A Dutch auction concludes; `settleTrade()` is called.
2. Protocol attempts an immediate follow-up `rebalance()` to restore collateral - any error should be handled transparently.

### What Actually Happens (Bug)

* `rebalance()` interacts with multiple external parts:

  1. `coll.refPerTok()` via `basketsHeldBy()`
  2. `basketRange()`, using `_quantity()` logic
  3. ERC-20 `approve()` via `AllowanceLib.safeApproveFallbackToMax()`
* If any of these calls revert without a message (empty data), the `catch` deems it an "OOG" error and forcibly reverts the transaction, cancelling the auction settlement entirely.

### Why This Matters

* **Dutch auctions may hang**, never settling or mispricing.
* Leads to stalled recollateralization—key system operations like revenue forwarding or trading halts.
* Difficult to debug since the revert is silent—no reason string is captured.

## Vulnerable Code Reference

```solidity
function settleTrade(IERC20 sell) public override returns (ITrade trade) {
    delete tokensOut[sell];
    trade = super.settleTrade(sell);

    if (_msgSender() == address(trade)) {
        try this.rebalance(trade.KIND()) {
            // do nothing
        } catch (bytes memory errData) {
            if (errData.length == 0) revert();  // Unsafe assumption: empty = OOG
        }
    }
}
```

Within `rebalance()`, any of the following external calls can silently fail:

* `coll.refPerTok()`
* `basketRange()` → `_quantity()` logic
* ERC-20 `approve()` via `AllowanceLib.safeApproveFallbackToMax()`

## Proof-of-Concept Flow

1. A Dutch auction is opened for rebalancing.
2. A bidder tries to settle at fair value.
3. `settleTrade()` calls `rebalance()` via `try/catch`.
4. Suppose `refPerTok()`, `basketRange()`, or `approve()` silently reverts.
5. The `catch` receives empty `errData` → treats as OOG → `settleTrade()` reverts.
6. Auction remains unsettled → protocol becomes blocked.

## Recommended Mitigation

1. **Avoid using `errData.length == 0` to detect OOG**. Replace this unreliable pattern.
2. Use the **`_reserveGas()`** pattern for unbound external calls — similar to prior mitigations.
3. Explicitly handle other failure modes:

   * Wrap calls like `refPerTok()` and `approve()` in `try/catch` with *clear revert messages*.
   * Example:

     ```solidity
     try collateral.refPerTok() returns (uint192 value) {
         ...
     } catch {
         revert("refPerTok failed");
     }
     ```

4. **Enhance observability** - capture error context and emit events when settlement errors occur.

## Pattern Recognition Notes

This vulnerability highlights a subtle but critical pattern: **treating empty revert data as synonymous with out-of-gas can mask real failures**.

### How to Spot Similar Issues

1. **Search for `try/catch` blocks checking `errData.length == 0`** — these are red flags.
2. **Audit paths that rely on external calls with no clear error fallback** (e.g., `approve()`, `price()`, `exchangeRate()`).
3. **Test with "non-standard" tokens** (like USDT) that revert silently in common operations.
4. **Consider upgradeability** — even a previously well-behaved plugin can be updated to misbehave.
5. Ensure **critical operations have clear failure paths**, not assumptions.
