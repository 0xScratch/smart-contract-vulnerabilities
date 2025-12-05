# Oracle-Based Post-Exit Skim Nullifies User Slippage Protection in `BLVaultLido`

* **Severity**: High
* **Source**: [Sherlock Audit](https://github.com/sherlock-audit/2023-03-olympus-judging/issues/3)
* **Affected Contract**: [BLVaultLido.sol](https://github.com/sherlock-audit/2023-03-olympus/blob/main/sherlock-olympus/src/policies/BoostedLiquidity/BLVaultLido.sol#L224-L247)
* **Vulnerability Type**: Slippage Bypass / Oracle Drift Exploit / Incorrect Assumption in User Input Validation

## Summary

`BLVaultLido#withdraw` allows the user to specify `minTokenAmounts_` for Balancer pool exit slippage protection. However, the vault **performs an additional oracle-based "wstETH skim" AFTER the Balancer exit**, where it compares the received OHM vs the oracle OHM→wstETH price and **forces the user to accept only the oracle-bounded wstETH amount**.

Any excess wstETH (due to pool mispricing, oracle slop, or MEV) is **skimmed to the Treasury**, *regardless of the user's slippage settings*.
This means:

> `minTokenAmounts_` no longer protects the user's **final** wstETH proceeds.
> It only protects the tokens that the **vault** receives from Balancer, but not what the **user** receives.

Thus, the user cannot protect themselves from **oracle drift**, **pool/oracle mismatch**, or **post-exit price manipulation**, even though the function signature implies they can.

## A Better Explanation (With Simplified Example)

### Intended Behavior

The withdraw function is supposed to allow the user to protect themselves from bad pool prices:

1. User specifies:

   ```text
   minTokenAmounts_ = [minOHM, minWstETH]
   ```

   to ensure Balancer returns at least those amounts.
2. Vault exits the Balancer pool.
3. Vault gives the user the tokens Balancer returned.

### What Actually Happens

The vault instead:

1. Exits Balancer (honoring `minTokenAmounts_`).
2. Measures **how much OHM it received**.
3. Uses an oracle to compute the *expected* wstETH amount for that OHM:

   ```text
   expected = ohmAmountOut * oraclePrice
   ```

4. User is given:

   ```text
   min( wstethAmountOut_from_pool, expected_wstETH_from_oracle )
   ```

5. Any "extra" wstETH is **skimmed to the Treasury**.

So even if Balancer returned the correct amount to satisfy the user's slippage, the vault may still take a significant portion of the wstETH due to oracle differences.

This breaks the promise of slippage control.

### Why This Matters

* **User cannot bound losses.**
  The contract signature suggests slippage protection, but the actual final output can be arbitrarily smaller.

* **Oracle drift makes this worse.**
  Even minor TWAP delay or stale data can cause meaningful wstETH skimming.

* **Users cannot detect or prevent it.**
  Because the skim happens after exit, their `minTokenAmounts_` is meaningless for their final payout.

* **Creates asymmetric risk.**
  Treasury captures upside (arb), user carries all downside (oracle mismatch).

## Concrete Walkthrough (Alice & Bob)

### Setup

Alice attempts to withdraw LP with:

```text
minTokenAmounts = [minOHM = 0, minWstETH = 9e18]
```

### Balancer exit

Balancer gives:

```text
10 wstETH and 8 OHM
```

This satisfies Alice's slippage setting: she got ≥9 wstETH.

### But the vault then enforces oracle price

Oracle says:

```text
1 OHM = 0.6 wstETH
expected = 8 * 0.6 = 4.8 wstETH
```

Therefore the contract returns:

```text
wstethToReturn = min(10, 4.8) = 4.8 wstETH
```

**Skims:**

```text
10 - 4.8 = 5.2 wstETH → Treasury
```

Alice receives **4.8**, even though she required **≥9** as her slippage protection.

> This proves that `minTokenAmounts_` does not actually protect the user from receiving fewer tokens than expected.

## Vulnerable Code Reference

### Oracle-based skim after Balancer exit

(From `BLVaultLido.sol:224-247`)

```solidity
_exitBalancerPool(lpAmount_, minTokenAmounts_);

uint256 ohmAmountOut = ohm.balanceOf(address(this)) - ohmBefore;
uint256 wstethAmountOut = wsteth.balanceOf(address(this)) - wstethBefore;

uint256 wstethOhmPrice = manager.getTknOhmPrice();
uint256 expectedWstethAmountOut = (ohmAmountOut * wstethOhmPrice) / _OHM_DECIMALS;

uint256 wstethToReturn = wstethAmountOut > expectedWstethAmountOut
    ? expectedWstethAmountOut
    : wstethAmountOut;

if (wstethAmountOut > wstethToReturn)
    wsteth.safeTransfer(TRSRY(), wstethAmountOut - wstethToReturn);

// burn OHM
manager.burnOhmFromVault(ohmAmountOut);

// return wstETH to user
wsteth.safeTransfer(msg.sender, wstethToReturn);
```

### Problem Highlight

`minTokenAmounts_` is only checked inside:

```text
_exitBalancerPool()
```

But the user's **actual return** is calculated **afterward**, ignoring those conditions.

## Recommended Mitigation

### 1. Allow user to specify minimum **final** wstETH output

Add a new parameter:

```solidity
uint256 minUserWstethAfterSkim
```

Then enforce:

```solidity
require(wstethToReturn >= minUserWstethAfterSkim, "Post-skim slippage");
```

This restores meaningful slippage protection.

### 2. Cap the skim amount (optional)

Prevent Treasury from capturing unlimited upside due to oracle drift.

```solidity
require(wstethAmountOut - wstethToReturn <= maxSkimAllowed, "Excessive skim");
```

### 3. Require oracle-pool deviation sanity check

Only apply skim when:

```text
|poolPrice - oraclePrice| <= threshold%
```

Otherwise revert for safety.

### 4. Better documentation

Clarify that slippage protection applies only to pool exit, not final user proceeds, unless fixed.

## Pattern Recognition Notes

* **Post-processing breaks slippage assumptions**
  Whenever the user protects amounts at step 1 (exit), but the protocol modifies those amounts in step 2 (post-exit), slippage becomes meaningless.

* **Oracle > Pool mismatch risk**
  Using an external oracle to override actual pool output creates user-visible price risk.

* **Hidden transfer-of-value**
  Treasury gains from mismatches; user loses — creating asymmetric incentives.

* **Input parameter mismatch**
  A function accepting `minTokenAmounts_` implies final-output slippage protection, but actual semantics differ.

## Quick Recall (TL;DR)

* **Bug**: Users specify minimums for Balancer exit, but their final received tokens are modified by an oracle-based skim that they cannot control.
* **Impact**: Users cannot protect themselves against oracle drift or pool/oracle mismatch; they can receive far less wstETH than expected.
* **Fix**: Add post-skim slippage parameter (e.g., `minUserWstethAfterSkim`) or cap the skim amount.
