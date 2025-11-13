# Flash-Loan Fee Ignorance Leading to Rebalance & Withdraw DoS in Sandclock Vaults

* **Severity**: High
* **Source**: [Solodit](https://solodit.cyfrin.io/issues/receiveflashloan-does-not-account-for-fees-trailofbits-none-lindy-labs-sandclock-pdf)
* **Affected Contracts**:
  * [`scWETHv2.sol`](https://github.com/lindy-labs/sandclock-contracts/blob/a100f21a30dd332b69351d1e05d98dbc748c6ddc/src/steth/scWETHv2.sol)
  * [`scUSDCv2.sol`](https://github.com/lindy-labs/sandclock-contracts/blob/a100f21a30dd332b69351d1e05d98dbc748c6ddc/src/steth/scUSDCv2.sol)
* **Vulnerability Type**: Data Validation / External Call Integration Risk / Withdraw DoS

## Summary

Both `scWETHv2` and `scUSDCv2` rely on **Balancer flash loans** during rebalances and withdrawals.
However, their `receiveFlashLoan()` implementations **ignore the flash-loan fee parameter** (`feeAmounts`) provided by Balancer and repay **only the principal amount**.

Balancer currently charges **zero fees**, which hides the issue.
If Balancer governance ever enables non-zero fees (the interface is already built for this), the vaults will begin **repaying less than the required amount**, causing Balancer's flash-loan logic to revert every time.

Since **rebalances and withdrawals depend on flash loans**, this leads to a **permanent stall / global DoS**, preventing users from withdrawing anything beyond the small float balance.

## A Better Explanation (Easy + Detailed)

### Intended Behavior

1. Vault requests a flash loan from Balancer (e.g., 12,000 WETH).
2. Balancer transfers the tokens and calls:

   ```solidity
   receiveFlashLoan(tokens, amounts, feeAmounts, userData)
   ```

3. Vault performs its sequence of internal operations (multicall).
4. Vault repays:

   ```solidity
   amounts[i] + feeAmounts[i]
   ```

   for every token.
5. Balancer checks repayment and completes the flash loan successfully.

### What Actually Happens

Inside Sandclock's `receiveFlashLoan`:

```solidity
// Only repays the principal
asset.safeTransfer(address(balancerVault), flashLoanAmount);
```

It **ignores**:

```solidity
uint256[] memory feeAmounts
```

passed in by Balancer.

If Balancer's fee ever becomes non-zero, repayment becomes:

* Returned: **flashLoanAmount**
* Required: **flashLoanAmount + feeAmount**

The difference (`feeAmount`) is **not returned**, so Balancer reverts the call.

**This breaks every rebalance and every large withdrawal.**

## Why This Creates a DoS

The vault must often **deleverage** to satisfy user withdrawals:

* repay debt on lending markets
* withdraw wstETH collateral
* convert it back to WETH
* return the flash loan

If flash loans revert:

❌ collateral cannot be withdrawn

❌ debt cannot be repaid

❌ user withdrawals cannot be serviced

❌ rebalances cannot complete

=> **Vault becomes stuck in leveraged state**

Only the small idle float (e.g., ~1 ETH or stablecoins) remains withdrawable.

This is effectively a **global withdraw freeze**.

## Concrete Walkthrough (Simple Numbers)

Assume Balancer starts charging a **0.09%** flash loan fee.

Vault requests:

```solidity
flash loan = 10,000 WETH
fee        = 9 WETH
```

### What the vault should repay

```solidity
10,000 + 9 = 10,009 WETH
```

### What the vault actually repays

```solidity
10,000 WETH
```

Balancer sees:

```solidity
receivedFeeAmount = postLoanBalance - preLoanBalance = 0
feeAmounts[0] = 9
```

Check:

```solidity
0 >= 9  ❌ revert
```

→ the entire flash-loan operation fails
→ withdrawToVault() fails
→ rebalance() fails
→ protocol becomes non-functional

## Vulnerable Code Reference

### 1) Balancer's flashLoan implementation enforces fee repayment

```solidity
receivedFeeAmount = postLoanBalance - preLoanBalance;
_require(receivedFeeAmount >= feeAmounts[i]);
```

If `receivedFeeAmount < feeAmounts[i]`, the entire flash loan **reverts**.

### 2) Sandclock's receiveFlashLoan ignores the fee argument

```solidity
function receiveFlashLoan(address[] memory, uint256[] memory amounts, uint256[] memory, bytes memory userData)
    external
{
    ...
    asset.safeTransfer(address(balancerVault), flashLoanAmount); // ❌ Repays only principal
    _enforceFloat();
}
```

Because the `feeAmounts` array is unused, repayment becomes invalid when fees > 0.

## Concrete Example (Alice Withdraws)

1. Alice requests to withdraw 5000 WETH.
2. The vault needs to deleverage using a flash loan.
3. Balancer now charges flash loan fees.
4. Flash loan callback returns principal only.
5. Balancer reverts.
6. Vault cannot deleverage → cannot free collateral → cannot serve Alice's withdrawal.
7. Every subsequent withdrawal attempt fails as well.

> **Analogy**:
> You borrow money from a bank that suddenly adds a service fee. Your repayment covers only the borrowed amount but not the fee — the bank rejects the repayment, and all operations requiring that loan grind to a halt.

## Recommended Mitigation

### **Short Term: Repay Flash-Loan Fees**

Use `feeAmounts[i]` when repaying:

```solidity
for (uint256 i = 0; i < amounts.length; i++) {
    uint256 repayAmount = amounts[i] + feeAmounts[i];
    ERC20(tokens[i]).safeTransfer(address(balancerVault), repayAmount);
}
```

### **Long Term: Improve External Call Validation**

* Document all external parameters that are intentionally ignored.
* Create invariant tests for:

  * non-zero flash loan fees
  * multi-token flash loans
  * fee repayment exceeding float
* Ensure minimum float covers potential fee repayment.

## Pattern Recognition Notes

* **Silent Parameter Ignoring**
  Ignoring external callback parameters (especially fee or rate parameters) is extremely dangerous. The external contract expects specific handling — deviations break integration.

* **External Fee-Dependent Logic**
  If the protocol's correctness depends on third-party fee schedules, the system should:

  * read,
  * validate, and
  * incorporate
    external fee variables.

* **Rebalance/Withdraw Coupling**
  Designs where withdrawal depends on leverage unwinding are very sensitive to failures in the unwinding step. A single unpaid fee can block the entire unwind path.

* **Future-Protocol-Change Risk**
  Relying on "fees are zero right now" is fragile. Governance-controlled protocols may change fees anytime.

## TL;DR

* **Bug**: `receiveFlashLoan()` repays **only the principal**, ignoring `feeAmounts`.
* **Impact**: If Balancer ever enables non-zero flash loan fees, **all rebalances and withdrawals revert**, freezing most user funds.
* **Fix**: Repay `amount + feeAmount` for every flash-loaned token; adjust float/reserve logic accordingly.
