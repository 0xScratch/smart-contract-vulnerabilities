# Fixed-Term Lenders Rugged via APR Reduction During Active Loan Term

* **Severity**: Medium
* **Source**: [Code4rena](https://github.com/code-423n4/2024-08-wildcat-findings/issues/77)
* **Affected Contract**: [FixedTermLoanHooks.sol](https://github.com/code-423n4/2024-08-wildcat/blob/main/src/access/FixedTermLoanHooks.sol)
* **Vulnerability Type**: Business Logic Error / Missing State Validation / Broken Fixed-Term Invariant

## Summary

`FixedTermLoanHooks` implements fixed-duration Wildcat lending markets where lenders are prevented from withdrawing their funds until a predefined maturity timestamp (`fixedTermEndTime`).

The fixed-term model relies on a simple agreement:

* Lenders accept locked funds for a fixed period.
* Borrowers pay the agreed APR during that period.

While the hook correctly prevents lenders from withdrawing before the term ends, it does **not prevent borrowers from reducing the market APR during the same period**.

This allows a borrower to attract lenders with a high APR, lock their funds inside a fixed-term market, and then reduce the APR (even to 0%) while lenders have no ability to exit.

The protocol enforces the lender's commitment but fails to enforce the borrower's commitment.

## A Better Explanation (With Simplified Example)

### Intended Behavior

In a fixed-term Wildcat market:

1. Borrower creates a market:

   * Loan duration: 1 year
   * APR: 10%

2. Lenders deposit funds because they accept the conditions:

   * Funds cannot be withdrawn early.
   * Borrower pays 10% APR.

3. Until the fixed term expires:

   * Lenders cannot withdraw.
   * Borrower should not be able to reduce APR.

The fixed interest rate is effectively part of the agreement.

### What Actually Happens (Bug)

The withdrawal restriction is implemented correctly:

```solidity
if (market.fixedTermEndTime > block.timestamp) {
    revert WithdrawBeforeTermEnd();
}
````

Any lender attempting to exit early is blocked.

However, the APR update hook simply forwards the request:

```solidity
return
  super.onSetAnnualInterestAndReserveRatioBips(
      annualInterestBips,
      reserveRatioBips,
      intermediateState,
      hooksData
  );
```

There is no validation checking:

```solidity
market.fixedTermEndTime > block.timestamp
```

before allowing APR changes.

Therefore:

* Withdraw before maturity → blocked
* Reduce APR before maturity → allowed

This creates an unfair state where lenders remain locked but the borrower can remove their expected yield.

### Why This Matters

Fixed-term lending depends on predictable terms.

A lender may accept:

```
1 year lockup + 10% APR
```

but would never accept:

```
1 year lockup + 0% APR
```

The vulnerability allows the borrower to change the deal after lenders are already trapped.

The normal Wildcat APR reduction protection relies on lenders being able to ragequit. Fixed-term markets remove that option, so APR reductions must also be restricted.

### Concrete Walkthrough (Alice & Bob)

* **Setup**

Bob (borrower) creates a fixed-term market:

```
APR = 15%
Duration = 1 year
```

Alice deposits:

```
1,000,000 USDC
```

because she expects:

```
15% yield after one year
```

* **Attack**

Immediately after receiving liquidity, Bob updates:

```
APR: 15% → 0%
```

The contract allows this because `onSetAnnualInterestAndReserveRatioBips` does not check whether the fixed term is still active.

* **Alice tries to exit**

Alice calls withdrawal:

```solidity
onQueueWithdrawal()
```

The hook checks:

```solidity
if (fixedTermEndTime > block.timestamp)
    revert WithdrawBeforeTermEnd();
```

Result:

```
Withdrawal rejected.
```

Alice must stay until maturity while earning almost nothing.

> **Analogy**:
> Imagine renting your money for one year because someone promises you $10,000 rent. After you hand over the keys, they change the contract to $0 rent, but the door is locked and you cannot take your property back until the year ends.

## Vulnerable Code Reference

### 1) Fixed-term withdrawal protection

```solidity
function onQueueWithdrawal(
    address lender,
    uint32 expiry,
    uint scaledAmount,
    MarketState calldata state,
    bytes calldata hooksData
) external override {

    HookedMarket memory market = _hookedMarkets[msg.sender];

    if (!market.isHooked)
        revert NotHookedMarket();

    if (market.fixedTermEndTime > block.timestamp) {
        revert WithdrawBeforeTermEnd();
    }
}
```

This correctly blocks lenders from withdrawing early.

### 2) Missing fixed-term validation during APR update

```solidity
function onSetAnnualInterestAndReserveRatioBips(
    uint16 annualInterestBips,
    uint16 reserveRatioBips,
    MarketState calldata intermediateState,
    bytes calldata hooksData
)
public
override
returns (
    uint16 updatedAnnualInterestBips,
    uint16 updatedReserveRatioBips
)
{
    return
      super.onSetAnnualInterestAndReserveRatioBips(
          annualInterestBips,
          reserveRatioBips,
          intermediateState,
          hooksData
      );
}
```

Missing validation:

```solidity
if (market.fixedTermEndTime > block.timestamp) {
    revert();
}
```

Because this check does not exist, APR reductions remain possible during active fixed-term loans.

## Recommended Mitigation

Prevent APR reductions while the fixed term is active.

Example:

```solidity
function onSetAnnualInterestAndReserveRatioBips(...)
    public
    override
    returns (...)
{
    HookedMarket memory market = _hookedMarkets[msg.sender];

    if (market.fixedTermEndTime > block.timestamp) {
        revert CannotReduceAprDuringFixedTerm();
    }

    return super.onSetAnnualInterestAndReserveRatioBips(...);
}
```

A more flexible solution is to only block APR decreases:

```solidity
if (
    newAPR < currentAPR &&
    market.fixedTermEndTime > block.timestamp
) {
    revert CannotReduceAprDuringFixedTerm();
}
```

APR increases can remain allowed because they benefit lenders.

## Pattern Recognition Notes

* **One-Sided Commitment Bug**:
  If a protocol locks one party into an agreement, ensure the other party cannot modify the important terms afterward.

* **Missing Symmetric Restrictions**:
  The protocol protected:

  ```
  lender cannot leave
  ```

  but forgot:

  ```
  borrower cannot worsen terms
  ```

* **Documentation vs Implementation Mismatch**:
  Documentation explicitly stated APR reductions should not happen when withdrawals are blocked, but the hook failed to enforce this.

* **Ragequit Assumption Failure**:
  APR reduction mechanisms often assume users can exit. If another module removes the exit path, the original protection no longer works.

* **Hook Composition Risk**:
  When using modular hook systems, every hook must preserve global protocol invariants. Adding withdrawal restrictions may require adding update restrictions elsewhere.

## Quick Recall (TL;DR)

* **Bug**: Fixed-term markets block lender withdrawals but still allow borrower APR reductions.
* **Impact**: Borrower attracts deposits with high APR → locks lenders → reduces APR to 0%.
* **Root Cause**: Missing fixed-term validation in APR update hook.
* **Fix**: Prevent APR reductions while `fixedTermEndTime > block.timestamp`.
