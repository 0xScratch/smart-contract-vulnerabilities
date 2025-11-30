# Single-Sided Redemption Slippage Bypass via Miscomputed Min Amounts

* **Severity**: High
* **Source**: [Sherlock Audits](https://github.com/sherlock-audit/2023-02-notional-judging/issues/10)
* **Affected Contracts**:

  * [`Curve2TokenConvexHelper.sol` (settlement path)](https://github.com/sherlock-audit/2023-02-notional/blob/main/leveraged-vaults/contracts/vaults/curve/external/Curve2TokenConvexHelper.sol#L112)
  * [`TwoTokenPoolUtils.sol` (min exit calculation)](https://github.com/sherlock-audit/2023-02-notional/blob/main/leveraged-vaults/contracts/vaults/common/internal/pool/TwoTokenPoolUtils.sol#L48)
  * [`Curve2TokenPoolUtils.sol` (actual single-side exit)](https://github.com/sherlock-audit/2023-02-notional/blob/main/leveraged-vaults/contracts/vaults/curve/internal/pool/Curve2TokenPoolUtils.sol#L231)
* **Vulnerability Type**: Business Logic / Invalid Validation (Slippage Controls Ineffective)

## Summary

During vault settlement, the helper forcibly **overwrites user-provided slippage limits** (`minPrimary`, `minSecondary`) with values from `_getMinExitAmounts`. That function computes **pro-rata** minimums (share of pool * a small discount). However, settlement may perform a **single-sided withdrawal** (`remove_liquidity_one_coin`), where slippage can be much larger than proportional.

Result: `minPrimary` becomes **far too low**, so the transaction **does not revert even under extreme slippage**, and the vault accepts a severely under-returned amount. This leads to **real loss for vault shareholders**.

## A Better Explanation (With Simplified Example)

### Intended Behavior

1. **At settlement**, the vault exits Curve LP and returns **primary tokens** to the strategy accounting.
2. The caller (or system) should enforce **meaningful slippage limits** (minimum amounts) so the exit **reverts** if the pool is imbalanced or manipulated.

### What Actually Happens (Bug)

* In `Curve2TokenConvexHelper._executeSettlement`, the code **overwrites** any caller-provided `minPrimary`, `minSecondary` by calling `_getMinExitAmounts`:

  ```solidity
  // Curve2TokenConvexHelper.sol
  (params.minPrimary, params.minSecondary) = poolContext.basePool._getMinExitAmounts({
      strategyContext: strategyContext,
      oraclePrice: oraclePrice,
      spotPrice: spotPrice,
      poolClaim: poolClaimToSettle
  });
  ```

* `_getMinExitAmounts` calculates **pro-rata** minimums with a small discount (designed for proportional exits), not for **single-sided** exits where slippage can be very large:

  ```solidity
  // TwoTokenPoolUtils.sol
  minPrimary   = (primaryBalance   * poolClaim * poolSlippageLimitPercent) / (totalPoolSupply * VAULT_PERCENT_BASIS);
  minSecondary = (secondaryBalance * poolClaim * poolSlippageLimitPercent) / (totalPoolSupply * VAULT_PERCENT_BASIS);
  ```

* Later, the actual exit may be **single-sided**:

  ```solidity
  // Curve2TokenPoolUtils._unstakeAndExitPool (single-sided branch)
  primaryBalance = curvePool.remove_liquidity_one_coin(poolClaim, primaryIndex, params.minPrimary);
  ```

* Because `params.minPrimary` was set to a **tiny** pro-rata number, the call **does not revert** even when the pool returns a much smaller-than-expected amount.

### Concrete Walkthrough

Pool state (simplified):

* Tokens: DAI (primary), USDC (secondary)
* Balances: **50 DAI + 150 USDC** = $200 total
* LP `totalSupply` = 100
* Settlement burns **50 LP** (≈ $100 of value expected)
* Caller would ideally set `minPrimary ≈ 95-100 DAI`

But `_getMinExitAmounts` sets:

```text
minPrimary = (50 DAI * 50/100) * 99.75% = ~24.94 DAI
```

If the pool is imbalanced and `remove_liquidity_one_coin` returns **30 DAI**, the transaction **succeeds** (since 30 ≥ 24.94) even though we expected roughly **~100 DAI**. The vault silently realizes a **~70% loss** on this exit.

### Why This Matters

* **Slippage protections are effectively bypassed** for single-side exits.
* During settlement, these exits can be **large and predictable**, inviting **MEV/front-running** or sandwiching that deepens imbalance right before the call.
* Losses are **socialized** across vault shareholders and are **irrecoverable**.

### Analogy

You're owed a $100 redemption. Your "auto-min" is mistakenly set to $25. The cashier hands you $30 — you accept it because your minimum was wrong — you just lost $70 without any alarm bells.

## Vulnerable Code Reference

### 1) Overwriting min amounts in settlement (pro-rata calc used even for single-sided exits)

* [`Curve2TokenConvexHelper._executeSettlement`](https://github.com/sherlock-audit/2023-02-notional/blob/main/leveraged-vaults/contracts/vaults/curve/external/Curve2TokenConvexHelper.sol#L112-L129)

    ```solidity
    // params.minPrimary and params.minSecondary are overwritten here:
    (params.minPrimary, params.minSecondary) = poolContext.basePool._getMinExitAmounts(...);
    ```

### 2) Pro-rata "min" calculation (not fit for single-sided)

* [`TwoTokenPoolUtils._getMinExitAmounts`](https://github.com/sherlock-audit/2023-02-notional/blob/main/leveraged-vaults/contracts/vaults/common/internal/pool/TwoTokenPoolUtils.sol#L48-L65)

    ```solidity
    minPrimary   = (poolContext.primaryBalance   * poolClaim * poolSlippageLimitPercent)
                / (totalPoolSupply * VAULT_PERCENT_BASIS);
    ```

### 3) Single-sided exit uses only `minPrimary`

* [`Curve2TokenPoolUtils._unstakeAndExitPool` (single-sided branch)](https://github.com/sherlock-audit/2023-02-notional/blob/main/leveraged-vaults/contracts/vaults/curve/internal/pool/Curve2TokenPoolUtils.sol#L241-L246)

    ```solidity
    primaryBalance = curvePool.remove_liquidity_one_coin(poolClaim, primaryIndex, params.minPrimary);
    ```

## Recommended Mitigation

1. **Do not overwrite** caller-provided `minPrimary` / `minSecondary` in settlement for single-sided paths.
   *Honor explicit slippage from the caller.*
   Add range checks to prevent unsafe values (e.g., relative to off-chain quote).

2. For **single-sided** exits, compute `minPrimary` using **Curve's `calc_withdraw_one_coin` off-chain** (or on-chain via a trusted quoting module), then apply a **tight discount**.

   > Note: `calc_withdraw_one_coin` uses spot balances and can be manipulated on-chain; use it **off-chain** (or via a TWAP-protected/checked quoting system) to produce the min amount.

3. If the protocol must compute mins on-chain, separate logic:

   * **Proportional exit** → keep pro-rata min calc.
   * **Single-sided exit** → use a **single-side-aware** min calc (quote + safety margin) and/or **multi-point price checks** (oracle vs spot, imbalance caps, max slippage percent).

4. Add **pre-exit imbalance checks** and **max slippage caps** (e.g., if expected_out / pro_rata_value < threshold → **abort**).

5. **Testing & Monitoring**

   * Unit tests for single-side exits under heavy imbalance should **revert** with correct mins.
   * Invariant tests ensuring "effective min" is at least a % of expected single-side quote.

## Pattern Recognition Notes

* **Context-Mismatched Guardrails**: Using **pro-rata** min amounts to guard a **single-sided** exit underestimates slippage risk.
* **Overwriting Caller Protections**: Discarding user-supplied slippage removes a critical safety valve.
* **Spot-Manipulable Quoting**: On-chain quotes based on spot balances can be skewed; prefer off-chain or TWAP-checked quotes for mins.
* **Settlement Is Attackable**: Predictable, large, one-shot exits are prime targets for MEV and pre-positioned imbalance.

### Quick Recall (TL;DR)

* **Bug**: Settlement **overwrites** min amounts with **pro-rata** values not valid for **single-sided** exits.
* **Impact**: Transactions **succeed under extreme slippage**, causing **large, permanent losses** to vault shareholders.
* **Fix**: Respect caller mins; compute single-side mins via **`calc_withdraw_one_coin` off-chain** (with discount) or a trusted quoting path; add strong slippage/imbalance caps and tests.
