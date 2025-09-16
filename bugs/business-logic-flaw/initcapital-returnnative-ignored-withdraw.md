# ReturnNative Flag Ignored in Withdraw Flow of MoneyMarketHook

* **Severity**: Medium
* **Source**: [Code4rena](https://github.com/code-423n4/2023-12-initcapital-findings/issues/29) / [One Bug Per Day](https://www.onebugperday.com/v1/790)
* **Affected Contract**: [MoneyMarketHook.sol](https://github.com/code-423n4/2023-12-initcapital/blob/main/contracts/hook/MoneyMarketHook.sol)
* **Vulnerability Type**: Business Logic Flaw / User Expectation Mismatch

## Summary

The `MoneyMarketHook` contract allows users to withdraw collateral and set a `returnNative` flag, signaling that they want **native ETH** (not WETH).

However, the withdraw pipeline implemented in `_handleWithdraw` **always directs WETH to the user's chosen `to` address**, without considering the `returnNative` flag. Since no WETH ends up in the hook's balance, the final unwrap logic in `execute` runs on a zero balance, producing **no ETH**.

As a result, users who request native ETH instead receive **ERC20 WETH**, creating a mismatch between expectations and outcome. This breaks integrations that only support ETH and could cause unexpected fund handling errors.

## A Better Explanation (With Simplified Example)

### Intended Behavior

1. Alice withdraws 1 ETH collateral with `returnNative = true`.
2. Protocol should:

   * Send WETH to the hook.
   * Hook unwraps it into ETH.
   * ETH is transferred to Alice's wallet.
   * Alice ends with **ETH**.

### Actual Behavior

1. `_handleWithdraw` routes WETH directly to Alice.
2. At the end of `execute`, the hook tries:

   ```solidity
   IWNative(WNATIVE).withdraw(IERC20(WNATIVE).balanceOf(address(this)));
   ```

   But balance is 0, so unwrap does nothing.
3. Alice ends with **WETH**, not ETH.

### Why This Matters

* **User expectation mismatch**: A user explicitly requested ETH (`returnNative = true`) but receives WETH.
* **Integration breakage**: Smart contracts or dApps consuming ETH directly could malfunction or revert when receiving WETH instead.
* **Silent failure**: There's no error, so users might not even realize until later that they got the wrong token type.

### Concrete Walkthrough (Alice & Bob)

* **Setup**: Alice deposited 1 ETH (wrapped as WETH inside protocol).
* **Withdraw attempt**: Alice calls withdraw with `returnNative = true`.
* **Bug behavior**:

  * Protocol sends 1 WETH to Alice.
  * Hook unwraps 0 WETH (balance empty).
  * Alice's wallet: 1 WETH (ERC20).
* **Expected behavior**:

  * Protocol sends 1 WETH to hook.
  * Hook unwraps 1 WETH → 1 ETH.
  * Alice's wallet: 1 ETH (native).

> **Analogy**: You ask a vending machine for "cash refund," but it quietly gives you store credit tokens instead. You can't spend them everywhere, and if you expected real cash, it's a problem.

## Vulnerable Code Reference

**Withdraw flow doesn't respect `returnNative` when setting `uTokenReceiver`:**

```solidity
address uTokenReceiver = _params[i].to;
// If helper used, redirect
if (_params[i].useRebasingHelpers) {
    uTokenReceiver = address(RebasingHelper);
}
// ❌ Missing: if returnNative && uToken == WNATIVE, send to hook instead
```

**Later unwrap in `execute` only works if WETH is held by the hook:**

```solidity
if (_params.returnNative) {
    IWNative(WNATIVE).withdraw(IERC20(WNATIVE).balanceOf(address(this)));
    (bool success,) = payable(msg.sender).call{value: address(this).balance}('');
    _require(success, Errors.CALL_FAILED);
}
```

Since no WETH is present, this unwrap is a no-op.

## Recommended Mitigation

1. **Route WETH to the hook when returnNative = true**
   Modify `_handleWithdraw`:

   ```solidity
   if (_params[i].returnNative && uToken == WNATIVE) {
       uTokenReceiver = address(this);  // keep WETH in hook
   }
   ```

2. **Ensure unwrap always runs on hook balance**
   By sending WETH to the hook, the final `withdraw` step properly converts it to ETH and forwards it to the user.

3. **Integration tests**
   Add tests confirming that setting `returnNative = true` yields ETH in the user's account, not WETH.

## Pattern Recognition Notes

* **ERC20 vs Native Token Mismatches**: Always ensure conversions are respected when offering both representations (ETH vs WETH).
* **Expectation vs Actual Flow**: Bugs often emerge where protocol flags are set but ignored mid-flow.
* **Receiver Logic in Withdraws**: When designing token pipelines, carefully route assets through the correct receiver if further processing (like unwrapping) is required.
* **Silent Failure Risks**: A function that "succeeds" but does the wrong thing (user got WETH instead of ETH) is dangerous, because errors are harder to detect.

### Quick Recall (TL;DR)

* **Bug**: Withdraw ignores `returnNative` flag, sends WETH directly to user.
* **Impact**: User expects ETH but gets WETH; breaks integrations that rely on ETH.
* **Fix**: If `returnNative && token == WNATIVE`, send to hook for unwrap, then forward ETH to user.
