# Arbitrary `originalAmount` in Flash Loan Data Allows Logic Manipulation

* **Severity**: Medium
* **Source**: [Code4rena](https://github.com/code-423n4/2024-05-bakerfi-findings/issues/2) / [One Bug Per Day](https://www.onebugperday.com/v1/1276)
* **Affected Contract**: [BalanceFlashLender.sol](https://github.com/code-423n4/2024-05-bakerfi/blob/59b1f70cbf170871f9604e73e7fe70b70981ab43/contracts/core/flashloan/BalancerFlashLender.sol)
* **Vulnerability Type**: Business Logic / Access Control Bypass

## Original Bug Description

>## Lines of code
>
>[https://github.com/code-423n4/2024-05-bakerfi/blob/59b1f70cbf170871f9604e73e7fe70b70981ab43/contracts/core/flashloan/BalancerFlashLender.sol#L109](https://github.com/code-423n4/2024-05-bakerfi/blob/59b1f70cbf170871f9604e73e7fe70b70981ab43/contracts/core/flashloan/BalancerFlashLender.sol#L109)
>
>## Vulnerability details
>
>## Impact
>
>receiveFlashLoan does not validate the originalCallData,
>The attacker can pass any parameters into the receiveFlashLoan function and execute any Strategy instruction :`_supplyBorrow` `_repayAndWithdraw` `_payDeb`.
>
>## Proof of Concept
>
>The `StrategyLeverage#receiveFlashLoan` function only validates whether the msg.sender is `_balancerVault`, but does not validate the `originalCallData`:
>
>```solidity
>   function receiveFlashLoan(address[] memory tokens,
>        uint256[] memory amounts, uint256[] memory feeAmounts, bytes memory userData
>    ) external {
> @>     if (msg.sender != address(_balancerVault)) revert InvalidFlashLoadLender();
>        if (tokens.length != 1) revert InvalidTokenList();
>        if (amounts.length != 1) revert InvalidAmountList();
>        if (feeAmounts.length != 1) revert InvalidFeesAmount();
>
>        //@audit originalCallData is not verified
>        (address borrower, bytes memory originalCallData) = abi.decode(userData, (address, bytes));
>        address asset = tokens[0];
>        uint256 amount = amounts[0];
>        uint256 fee = feeAmounts[0];
>        // Transfer the loan received to borrower
>        IERC20(asset).safeTransfer(borrower, amount);
>
>@>      if (IERC3156FlashBorrowerUpgradeable(borrower).onFlashLoan(borrower,
>                tokens[0], amounts[0], feeAmounts[0], originalCallData
>            ) != CALLBACK_SUCCESS
>        ) {
>            revert BorrowerCallbackFailed();
>        }
>        ....
>    }
>```
>
>_balancerVault.flashLoan can specify the recipient:
>
>```solidity
>    _balancerVault.flashLoan(address(this), tokens, amounts, abi.encode(borrower, data));
>```
>
>An attacker can initiate flashLoan from another contract and specify the recipient as `BalancerFlashLender`. `_balancerVault` will call the `balancerFlashlender#receiveFlashLoan` function,
>Since the caller of the receiveFlashLoan function is `_balancerVault`, this can be verified against `msg.sender`.
>
>The `StrategyLeverage#onFlashLoan` function parses the instructions to be executed from `originalCallData(FlashLoanData)` and executes them.
>
>```solidity
>    function onFlashLoan(address initiator,address token,uint256 amount, uint256 fee, bytes memory callData
>    ) external returns (bytes32) {
>        if (msg.sender != flashLenderA()) revert InvalidFlashLoanSender();
>        if (initiator != address(this)) revert InvalidLoanInitiator();
>        // Only Allow WETH Flash Loans
>        if (token != wETHA()) revert InvalidFlashLoanAsset();
>        //loanAmount = leverage - msg.value;
>@>      FlashLoanData memory data = abi.decode(callData, (FlashLoanData));
>        if (data.action == FlashLoanAction.SUPPLY_BOORROW) {
>            _supplyBorrow(data.originalAmount, amount, fee);
>            // Use the Borrowed to pay ETH and deleverage
>        } else if (data.action == FlashLoanAction.PAY_DEBT_WITHDRAW) {
>            // originalAmount = deltaCollateralInETH
>            _repayAndWithdraw(data.originalAmount, amount, fee, payable(data.receiver));
>        } else if (data.action == FlashLoanAction.PAY_DEBT) {
>            _payDebt(amount, fee);
>        }
>        return _SUCCESS_MESSAGE;
>    }
>```
>
>So the attacker by calling the `_balancerVault` flashLoan function, designated `recipient` for `BalancerFlashLender`, `borrower` for `StrategyLeverage`,
>An attacker can invoke the `_supplyBorrow` `_repayAndWithdraw` `_payDeb` function in `StrategyLeverage` with any `FlashLoanData` parameter.
>
>## Tools Used
>
>vscode, manual
>
>## Recommended Mitigation Steps
>
>1. `BalancerFlashLender#flashLoan` function to record the parameters called via hash.
>2. Verify the hash value in the `receiveFlashLoan` function.
>
>## Assessed type
>
>Access Control

## Summary

The protocol's flash loan handling logic (`onFlashLoan`) allows an attacker to pass arbitrary `FlashLoanData` — including the `originalAmount` field — without validation.

As a result, the protocol trusts this user-supplied `originalAmount` in critical downstream functions like `_supplyBorrow` or `_repayAndWithdraw`. This breaks the assumption that `originalAmount` matches the actual flash-loaned `amount`, enabling attackers to manipulate internal calculations and bypass expected limits or repayment rules.

## A Better Explanation (With Simplified Example)

### Intended Behavior

* When a flash loan is initiated, the protocol:

  1. Receives the actual loaned `amount` from the flash loan provider.
  2. Passes metadata (`FlashLoanData`) to `onFlashLoan` for processing.
  3. Uses the real loan amount for supply, borrow, repay, or withdrawal logic.

* `originalAmount` should always reflect the true `amount` of tokens received via the flash loan.

### What Actually Happens (Bug)

* The `FlashLoanData` struct — including `originalAmount` — is **fully attacker-controlled**.
* There is **no validation** that `data.originalAmount == amount`.
* Downstream logic uses `data.originalAmount` directly:

```solidity
_supplyBorrow(data.originalAmount, amount, fee);
```

* This means:

  * `amount` (actual flash loan size) might be 1 token.
  * `originalAmount` (attacker-supplied) could be set to 1,000,000 tokens.
  * The lending/borrowing logic executes based on the fake `originalAmount` instead of the real amount.

### Why This Matters

This trust in unverified input allows attackers to:

* Bypass internal lending/borrowing caps.
* Inflate accounting numbers to manipulate protocol state.
* Potentially drain funds or trigger undesired position changes.

It's essentially a logic injection via a parameter that was assumed to be "system-trusted" but is in fact attacker-controlled.

## Vulnerable Code Reference

```solidity
function onFlashLoan(
    address initiator,
    address token,
    uint256 amount,
    uint256 fee,
    bytes calldata data
) external returns (bytes32) {
    FlashLoanData memory data = abi.decode(data, (FlashLoanData));

    // ❌ No validation: data.originalAmount can be anything
    _supplyBorrow(data.originalAmount, amount, fee); 
    // or
    _repayAndWithdraw(data.originalAmount, amount, fee);
}
```

## Recommended Mitigation

1. **Enforce consistency between `originalAmount` and `amount`**

   * Add a strict check:

   ```solidity
   require(data.originalAmount == amount, "Invalid originalAmount");
   ```

2. **Eliminate redundant parameters**

   * If `amount` is already known from the flash loan callback, remove `originalAmount` from user-supplied data entirely.
3. **Treat all `FlashLoanData` as untrusted input**

   * Validate every field before using it in sensitive operations.
4. **Add tests**

   * Include scenarios where a malicious initiator passes mismatched amounts.
   * Ensure protocol state remains consistent even with unexpected input.

## Pattern Recognition Notes

This vulnerability is a **classic "trusting untrusted input"** problem in smart contracts, especially in flash loan contexts. Here's how to spot similar issues elsewhere:

1. **Check every field in user-supplied structs**

   * Even in callbacks like `onFlashLoan`, `onERC721Received`, etc., verify that all parameters match expected, provable values.

2. **Be wary of duplicated values**

   * If a function receives both a "true" source-of-truth value (`amount` from flash loan provider) and a user-supplied copy (`originalAmount`), assume the copy can be malicious.

3. **Flash loan-specific red flag**

   * In flash loan callbacks, parameters like "expected repay amount" or "original amount borrowed" should always be computed internally, never trusted from the initiator.

4. **Look for downstream trust**

   * See where the unvalidated value flows — is it used in accounting, cap checks, collateralization math? If so, risk is higher.

5. **Eliminate redundant storage of authoritative data**

   * Keep only one canonical source for values like amounts, rates, or addresses; avoid reading attacker-provided duplicates.

6. **Test for input mismatch attacks**

   * Simulate calling the function with `amount = small`, `originalAmount = huge` (or vice versa) and observe effects.
