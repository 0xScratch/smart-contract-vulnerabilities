# Invalid Input Validation Leading to Slippage/Token Order Mismatch

- **Severity**: Medium
- **Source**: [Code4rena](https://github.com/code-423n4/2023-07-angle-mitigation-findings/issues/30) / [One Bug Per Day](https://www.onebugperday.com/v1/293)
- **Affected Contract**: [Redeemer.sol](https://github.com/AngleProtocol/angle-transmuter/blob/3e43e29d2b2f0b75876396e7c65e48c00c5fd1b2/contracts/transmuter/facets/Redeemer.sol#L119)
- **Vulnerability Type**: Invalid Input Validation / Slippage Protection Failure

## Original Bug Description

> ### Vulnerability details
>
> ### Original Issue
>
> [code-423n4/2023-06-angle-findings#8](https://github.com/code-423n4/2023-06-angle-findings/issues/8)
>
> ### Details
>
> This issue shows users may get fewer tokens than expected when the collateral list order changes.
>
> As mitigation, it recommends checking the length of `minAmountsOut` and `ts.collateralList` as well as the token addresses to resolve the problem completely.
>
> The original submission recommends like the below.
>
> ```code
>The problem could be alleviated a bit by checking the length of minAmountsOut (making sure it is not longer than ts.collateralList). 
>However, that would not help if a collateral is revoked and a new one is added. 
>Another solution would be to provide pairs of token addresses and amounts, which would solve the problem completely.
> ```
>
> ### Mitigation
>
> PR: [AngleProtocol/angle-transmuter@f8d0bf7](https://github.com/AngleProtocol/angle-transmuter/commit/f8d0bf7c4009586f7022d5929359041db3990175)
>
> It validates the length of `minAmountsOut` and `ts.collateralList` but doesn't compare the token addresses.
>
> As a result, the original problem still exists when a collateral is revoked and a new one is added.
>
> ### Recommended Mitigation
>
> We should check the token addresses of `minAmountsOut` and `ts.collateralList` to resolve the original issue completely.
>
> ### Conclusion
>
> This issue wasn't mitigated properly.

## Summary

The `redeem` and `redeemWithForfeit` functions verify only that the length of the internally computed `tokens`/`amounts` array matches the user-supplied `minAmountOuts` array. They do **not** verify that the token addresses align with user expectations. If the protocol's collateral list (or its order) changes between quoting and execution—or if collaterals are revoked/added—users may receive incorrect or fewer tokens than they intended, bypassing slippage protections.

## Real-World Context

- Angle Protocol supports managed collateral strategies that may split a single base asset into multiple sub-collaterals (e.g., `USDC` → `yvUSDC`, `aUSDC`, `fUSDC`).
- Users compute slippage tolerances based on a quoted token list; if the actual token list mutates (added/removed/reordered) on execution, users' `minAmountOuts` apply to the wrong tokens.
- Such mismatches can be exploited by malicious governance actions or front-running, causing users to unknowingly accept unwanted tokens or value loss.

## Key Details

- **Pattern**: Missing cross-validation of user input against dynamic output arrays.
- **Root Cause**: Only `amounts.length == minAmountOuts.length` is enforced, not that `tokens[i] == expectedTokens[i]`.
- **Risk**: Users suffer silent losses or receive tokens they did not request, undermining trust and causing financial harm.

## Vulnerable Code Reference

```solidity
uint256[] memory subCollateralsTracker;
(tokens, amounts, subCollateralsTracker) = _quoteRedemptionCurve(amount);
// Check that the provided slippage tokens length is identical to the redeem one
uint256 amountsLength = amounts.length;
if (amountsLength != minAmountOuts.length) revert InvalidLengths();
```

## Impact

- **Value Loss**: Users may receive lower-value or unwanted assets if collateral order changes.
- **Slippage Failure**: Slippage protections become ineffective, exposing users to front-running and sandwich attacks.
- **Trust Erosion**: Silent mismatches decrease confidence in protocol safety during critical redemption operations.

## Mitigation Steps

- **Token-Amount Pairing**: Require users to submit an `expectedTokens[]` array alongside `minAmountOuts[]`, then enforce:

    ```solidity
    for (uint256 i = 0; i < tokens.length; ++i) {
        if (tokens[i] != expectedTokens[i]) revert TokenOrderMismatch();
        if (amounts[i] < minAmountOuts[i]) revert TooSmallAmountOut();
    }
    ```

- **Struct-Based Input**: Replace parallel arrays with a struct:

    ```solidity
    struct MinOutput { address token; uint256 minAmount; }
    function redeem(MinOutput[] calldata expected) external { … }
    ```

    verifying each `expected[i].token == tokens[i]`.
- **List Length Guard**: Optionally enforce `expected.length == ts.collateralList.length` when no sub-collaterals exist, to detect unauthorized collateral changes.

## Pattern Recognition Notes

- Look for public or external functions that accept user-supplied arrays (e.g., `minAmountOuts`, `forfeitTokens`, `recipients[]`, `amounts[]`) and use `.length` to validate without checking element content.
- Flag any validation logic that only compares array lengths (e.g., `if (a.length != b.length) revert`) but never verifies that the array **elements** correspond to expected on-chain values (addresses, IDs, flags).
- Watch for dynamic output arrays derived from mutable protocol state (collateral lists, strategy tokens, whitelist entries) where the mapping between input and output can shift if the underlying list changes.
- Identify functions that rely on implicit ordering of protocol state (e.g., collateralList order, strategy sub-collaterals) but do not enforce that the user's input order matches the on-chain order.
- **Red flags**
  - Parallel arrays used for slippage or distribution checks without cross-validation of index correspondence.
  - Code comments indicating "order must follow X list" but no runtime assertion of token or identifier equality.
  - Input arrays longer or shorter than a storage list without explicit guard against unexpected elements.
- **Common locations**
  - Redemption, swap, or distribution functions in DeFi protocols.
  - Batch operations (e.g., mass withdrawals, multi-token transfers) where users provide per-token or per-recipient limits.
  - Permissionless upgrade or governance-triggered reconfigurations that alter underlying lists used by public functions.
- **Testing approach**
  - Simulate protocol state changes: add/remove/reorder entries in the collateral, whitelist, or recipient lists between quoting and execution.
  - Attempt calls with stale or mismatched input arrays and verify that the contract either reverts or correctly rejects mismatched tokens/IDs.
  - Use property-based fuzzing to generate input arrays of varying lengths and content, verifying that every array element is validated against expected on-chain state.
  - Check for slippage or distribution failures by front-running list mutations in a forked or staging environment.
