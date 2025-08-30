# Fragile Liquidation Check in `Comptroller.sol` — Zero Borrow Balance Requirement

* **Severity**: Medium
* **Source**: [Code4rena](https://github.com/code-423n4/2023-05-venus-findings/issues/365) / [One Bug Per Day](https://www.onebugperday.com/v1/179)
* **Affected Contract**: [Comptroller.sol](https://github.com/code-423n4/2023-05-venus/blob/8be784ed9752b80e6f1b8b781e2e6251748d0d7e/contracts/Comptroller.sol)
* **Vulnerability Type**: Business Logic / Invalid Validation

## Original Bug Description

>## Lines of code
>
>[https://github.com/code-423n4/2023-05-venus/blob/8be784ed9752b80e6f1b8b781e2e6251748d0d7e/contracts/Comptroller.sol#L690-L693](https://github.com/code-423n4/2023-05-venus/blob/8be784ed9752b80e6f1b8b781e2e6251748d0d7e/contracts/Comptroller.sol#L690-L693)
>
>## Vulnerability details
>
>## Impact
>
>Functon **liquidateAccount** will fail if the transaction is not included in current block because interest accures per block, and **repayAmount** and **borrowBalance** need to match precisely.
>
>## Proof of Concept
>
>At the end of function **liquidateAccount**, a check is performed to ensure that the **borrowBalance** is zero:
>
>```solidity
>for (uint256 i; i < marketsCount; ++i) {
>        (, uint256 borrowBalance, ) = _safeGetAccountSnapshot(borrowMarkets[i], borrower);
>        require(borrowBalance == 0, "Nonzero borrow balance after liquidation");
>}
>```
>
>This means that **repayAmount** specified in calldata must exactly match the **borrowBalance**. (If repayAmount is greater than borrowBalance, Comptroller.preLiquidateHook will revert with error TooMuchRepay.) However, the **borrowBalance** is updated every block due to interest accrual. The liquidator cannot be certain that their transaction will be included in the current block or in a future block. This uncertainty significantly increases the likelihood of liquidation failure.
>
>## Tools Used
>
>Manual
>
>## Recommended Mitigation Steps
>
>Use a looser check
>
>```solidity
>snapshot = _getCurrentLiquiditySnapshot(borrower, _getLiquidationThreshold);
>require (snapshot.shortfall == 0);
>```
>
>to replace
>
>```solidity
>for (uint256 i; i < marketsCount; ++i) {
>        (, uint256 borrowBalance, ) = _safeGetAccountSnapshot(borrowMarkets[i], borrower);
>        require(borrowBalance == 0, "Nonzero borrow balance after liquidation");
>}
>```
>
>## Assessed type
>
>Invalid Validation

## Summary

`liquidateAccount` contains a final validation that requires the borrower's borrow balances to be **exactly zero** after liquidation. Because borrow balances grow automatically **per block** (interest accrual), the exact repayment amount is brittle: a liquidator cannot reliably ensure their `repayAmount` matches the instantaneous borrow balance at the moment the transaction is mined. If the transaction lands in a later block with a tiny extra interest accrual, the check fails and the entire liquidation reverts.

**Impact:** Unreliable/fragile liquidations that randomly revert; discouraged liquidators; accumulation of bad debt; higher systemic risk and potential insolvency under stress.

## A Better Explanation (With Simplified Example)

### Intended invariant

When a liquidator completes a liquidation that targets a borrower, the protocol's *true* safety invariant is:

> After the liquidation, the borrower should **no longer be undercollateralized** (i.e., they must have no shortfall under the liquidation thresholds).

The protocol does **not** need the borrower to have zero nominal borrow balance — it only needs the borrower's collateral to be sufficient relative to remaining debt.

### What the code checks (buggy behavior)

At the end of `liquidateAccount`, the following loop is executed:

```solidity
for (uint256 i; i < marketsCount; ++i) {
    (, uint256 borrowBalance, ) = _safeGetAccountSnapshot(borrowMarkets[i], borrower);
    require(borrowBalance == 0, "Nonzero borrow balance after liquidation");
}
```

This requires **exact zero** borrow balance in every market after the liquidation completes.

### Why that is brittle (interest accrual / timing race)

* Each borrow's outstanding balance is increased on-chain by interest accrual **per block** (the protocol computes per-block interest rates and applies them when balances are read/updated).
* A liquidator prepares a `repayAmount` value expecting it to match the borrow balance at some point in time.
* If the liquidator's transaction is included in a later block than expected (typical in an asynchronous mempool environment), the borrow balance can be slightly larger than the value used to craft `repayAmount`.
* If `repayAmount` is smaller than the *current* borrow balance, the loop will see a nonzero borrowBalance and revert.
* If `repayAmount` is larger than the borrow balance, other checks (e.g., `TooMuchRepay` in `preLiquidateHook`) may revert — so liquidator cannot safely overpay either.

Together these make it impractical to craft a repay value that **always** succeeds.

### Numeric example (concrete)

1. Borrower debt before block `N`: **100.000000** tokens.
2. Per-block interest accrual adds **0.000001** tokens per block (example small rate).
3. Liquidator calculates `repayAmount = 100.000000` and sends the tx.
4. If the tx is mined in block `N`: repay clears debt, `borrowBalance == 0` passes — ✅ success.
5. If the tx is mined in block `N+1`: borrow balance becomes `100.000001` before repay. Liquidator repays `100.000000`. After repay a dust of `0.000001` remains. The final loop sees a nonzero borrowBalance and the tx reverts — ❌ failure.

An analogous failure occurs if the liquidator speculatively overpays: `preLiquidateHook` contains protections like `TooMuchRepay` and will revert if the caller attempts to repay more than the allowed amount.

This variability — caused purely by natural interest accrual across blocks — makes the check unreliable and therefore dangerous for protocol liveness.

### Why the auditor's proposed check is correct

The auditor suggests replacing the exact-zero-balance check with a **health snapshot** check that asserts the borrower's account is no longer liquidatable:

```solidity
snapshot = _getCurrentLiquiditySnapshot(borrower, _getLiquidationThreshold);
require(snapshot.shortfall == 0);
```

This enforces the *semantic* requirement: after liquidation, borrower must not have a shortfall. It does **not** force zeroing dust-level debt — which is unnecessary for safety as long as the borrower is solvent.

To summarize:

* ✅ The protocol cares about *shortfall* (undercollateralization), not exact zero debt.
* ✅ Using `shortfall == 0` tolerates tiny remainder debt caused by interest accrual while preserving safety.

## Vulnerable Code Reference

The failing check appears in `Comptroller.sol` near lines L690-L693 (see repo link above). The relevant snippet is:

```solidity
for (uint256 i; i < marketsCount; ++i) {
    (, uint256 borrowBalance, ) = _safeGetAccountSnapshot(borrowMarkets[i], borrower);
    require(borrowBalance == 0, "Nonzero borrow balance after liquidation");
}
```

(If you view the full function `liquidateAccount`, you will see the loop placed at the end as a final assertion that the liquidation cleared all borrow balances.)

## Recommended Mitigation

1. **Replace exact-zero checks with a liquidity/shortfall snapshot check**

   Replace the loop above with something like:

   ```solidity
   // After performing liquidation actions and transfers, assert borrower is no longer undercollateralized
   LiquiditySnapshot memory snapshot = _getCurrentLiquiditySnapshot(borrower, _getLiquidationThreshold);
   require(snapshot.shortfall == 0, "Borrower still undercollateralized after liquidation");
   ```

   * `shortfall == 0` is robust to tiny interest accrual differences between transaction construction and mining.
   * This preserves the protocol invariant: the borrower is no longer eligible for liquidation.

2. **Keep existing per-market checks that prevent over-repay or protocol abuse**

   * Maintain `preLiquidateHook` protections (e.g., enforcing close factor, preventing `TooMuchRepay`) because they encode important limits.

3. **Add targeted unit & integration tests**

   * Test liquidations across block boundaries using `evm_mine` to advance blocks and verify non-reversion when leftover dust exists but no shortfall.
   * Test the other scenario: if liquidation does not fully remove shortfall, it must revert.

4. **Consider defensive tolerance logging**

   * Optionally emit an event when a liquidation leaves a small remainder dust (for observability). However, prefer the shortfall-based check as primary control.

5. **Review related invariants and code paths**

   * Audit other exact-equality checks that rely on time-based accruals (accruing interest, accumulating indices, or price feeds) and replace with appropriate invariants.

## Pattern Recognition Notes

1. **Avoid exact equality checks on state that changes over time**

   * Any state that accrues continuously (interest, index growth, fees that are applied per-block) should not be validated using `==` unless the code also deterministically updates the state in the same transaction and accounts for the accrual.

2. **Prefer semantic invariants over mechanical ones**

   * Ask *what property do we truly need?* (e.g., borrower not liquidatable) rather than *what exact numeric state would be nice* (e.g., borrow balance equals zero).

3. **Design for asynchronous execution**

   * Transactions may be mined in later blocks; checks must tolerate small differences caused by time-based updates.

4. **Unit tests must simulate time progression**

   * Add tests that advance blocks/time to reveal brittle equality checks; many real-world failures appear only when block timing shifts.

5. **Be cautious with "final state" assertions in multi-market flows**

   * When a function touches multiple markets, preferred checks should be cross-market invariants (e.g., net shortfall) instead of per-market zero balances unless zeroing is actually required and robustly enforced.

6. **Logging & observability**

   * Emit clear events for liquidation outcomes (success, partial clear with dust, failure reason) to ease debugging and monitoring.
