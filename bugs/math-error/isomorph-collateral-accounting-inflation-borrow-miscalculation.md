# Bad Debt Persistence via Truncation Mismatch in Isomorph Velo Vault

* **Severity**: Medium
* **Source**: [Sherlock Audit - Isomorph 2022-11](https://github.com/sherlock-audit/2022-11-isomorph-judging/issues/174)
* **Affected Contract**: [Vault_Velo.sol](https://github.com/sherlock-audit/2022-11-isomorph/blob/main/contracts/Isomorph/contracts/Vault_Velo.sol)
* **Vulnerability Type**: Accounting / Precision Error / Bad Debt Handling

## Summary

When a user in **Vault_Velo** is fully liquidated, the protocol should clear any remaining loan balance because all their collateral has been seized. However, due to **mismatched rounding logic** between two valuation functions — `totalCollateralValue()` and `_calculateProposedReturnedCapital()` — the system may **fail to recognize** that all collateral was liquidated.

This leads to a case where the vault removes *all NFTs* but still believes some debt remains, resulting in **persistent "bad debt"** that cannot be cleared.

## A Better Explanation (With Simplified Example)

### Intended Behavior

During liquidation, the vault:

1. Calculates total user collateral (`totalUserCollateral`).
2. Determines the proposed liquidation amount (`proposedLiquidationAmount`).
3. If *all collateral* is being liquidated, i.e.:

   ```solidity
   if (proposedLiquidationAmount >= totalUserCollateral) {
       // Clear bad debt
   }
   ```

   then all user debt should be forgiven since no collateral remains.

### What Actually Happens (Bug)

Both values are computed differently:

1. **`totalCollateralValue()`**

   * Sums all pooled tokens **first**, then prices them **once**.
   * Truncation occurs **only once** (end of calculation).

   ```solidity
   totalPooledTokens = sum(all NFTs' pooledTokens)
   totalUserCollateral = priceLiquidity(totalPooledTokens)
   ```

2. **`_calculateProposedReturnedCapital()`**

   * Prices each NFT **individually**, truncating **each value** before summing.

   ```solidity
   proposedLiquidationAmount += priceCollateral(NFT[i])
   ```

This difference causes slight under-valuation of `proposedLiquidationAmount`.

### Concrete Example

| NFT     | True Value    | Truncated Individually | Combined (once truncated) |
| ------- | ------------- | ---------------------- | ------------------------- |
| NFT #1  | 10.6          | 10                     | —                         |
| NFT #2  | 10.7          | 10                     | —                         |
| **Sum** | **21.3 → 21** | **10 + 10 = 20**       | **21**                    |

Even though both NFTs are liquidated:

* `totalUserCollateral = 21`
* `proposedLiquidationAmount = 20`

→ `20 >= 21` **fails**, so the vault *thinks* collateral remains and **does not clear the bad debt**.

## Why This Matters

This mismatch produces **inconsistent accounting**:

* The user has **no collateral left**, but a **non-zero loan** remains.
* The vault's system now contains **bad debt** that cannot be offset or liquidated further.
* Over time, this can **pollute total debt metrics**, mislead risk models, or confuse off-chain systems tracking solvency.

While it doesn't directly cause fund loss, it **breaks debt clearing invariants** and could complicate recovery logic or reporting.

## Vulnerable Code Reference

### 1) Liquidation block relying on inconsistent comparison

```solidity
if (proposedLiquidationAmount >= totalUserCollateral) {
    //@audit bad debt cleared here
}
```

### 2) `totalCollateralValue` — truncates once (sum then price)

```solidity
function totalCollateralValue(address _collateralAddress, address _owner) public view returns(uint256) {
    uint256 totalPooledTokens;
    for (uint256 i = 0; i < NFT_LIMIT; i++) {
        if (userNFTs.ids[i] != 0) {
            totalPooledTokens += depositReceipt.pooledTokens(userNFTs.ids[i]);
        }
    }
    return depositReceipt.priceLiquidity(totalPooledTokens);
}
```

### 3) `_calculateProposedReturnedCapital` — truncates per NFT

```solidity
function _calculateProposedReturnedCapital(...) internal view returns(uint256) {
    uint256 proposedLiquidationAmount;
    for (uint256 i = 0; i < NFT_LIMIT; i++) {
        if (_loanNFTs.slots[i] < NFT_LIMIT) {
            proposedLiquidationAmount += _priceCollateral(IDepositReceipt(_collateralAddress), _loanNFTs.ids[i]);
        }
    }
    return proposedLiquidationAmount;
}
```

## Recommended Mitigation

To ensure consistency, `_calculateProposedReturnedCapital()` should follow the same approach as `totalCollateralValue()` — **sum pooled tokens first**, then call `priceLiquidity()` once at the end.

**Fixed Implementation:**

```solidity
function _calculateProposedReturnedCapital(
    address _collateralAddress,
    CollateralNFTs calldata _loanNFTs,
    uint256 _partialPercentage
) internal view returns(uint256) {
    IDepositReceipt depositReceipt = IDepositReceipt(_collateralAddress);
    uint256 totalPooledTokens;

    require(_partialPercentage <= LOAN_SCALE, "partialPercentage greater than 100%");

    for (uint256 i = 0; i < NFT_LIMIT; i++) {
        if (_loanNFTs.slots[i] < NFT_LIMIT) {
            if (i == NFT_LIMIT - 1 && _partialPercentage < LOAN_SCALE && _partialPercentage > 0) {
                totalPooledTokens += (depositReceipt.pooledTokens(_loanNFTs.ids[i]) * _partialPercentage) / LOAN_SCALE;
            } else {
                totalPooledTokens += depositReceipt.pooledTokens(_loanNFTs.ids[i]);
            }
        }
    }
    return depositReceipt.priceLiquidity(totalPooledTokens);
}
```

## Pattern Recognition Notes

* **Truncation Order Sensitivity** — Summing before truncation versus truncating before summation can create silent off-by-one errors in financial logic.
* **Cross-Function Consistency** — When two code paths are compared for equality (`>=`, `==`), ensure they share identical valuation pipelines.
* **Debt-Clearing Invariant** — When all collateral is seized, resulting debt must reach zero; otherwise, future state accounting diverges.
* **Precision Risks in Solidity** — Integer math without fixed-point decimals demands cautious structuring to prevent cumulative truncation bias.
* **Verification Tests** — Unit tests should include multi-NFT full liquidations to assert that debt is fully cleared when collateral arrays are emptied.

### Quick Recall (TL;DR)

* **Bug**: `totalCollateralValue` and `_calculateProposedReturnedCapital` use different rounding logic.
* **Impact**: Even after full liquidation, users loan remains partially uncleared → persistent bad debt.
* **Fix**: Sum pooled tokens before pricing (align both functions' valuation methods).
* **Severity**: Medium — no direct theft, but breaks debt invariants and pollutes accounting.
