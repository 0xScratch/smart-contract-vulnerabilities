# Unchecked Call Return Value in ETH Transfers

- **Severity**: Medium
- **Source**: [Code4rena](https://github.com/code-423n4/2023-09-goodentry-mitigation-findings/issues/3) / [One Bug Per Day](https://www.onebugperday.com/v1/548)
- **Affected Contracts**: [V3Proxy.sol](https://github.com/GoodEntry-io/ge/blob/c7c7de57902e11e66c8186d93c5bb511b53a45b8/contracts/helper/V3Proxy.sol)
- **Vulnerability Type**: call/delegatecall - Unchecked Return Value

## Original Bug Description

>## Lines of code
>
>[https://github.com/GoodEntry-io/ge/blob/c7c7de57902e11e66c8186d93c5bb511b53a45b8/contracts/helper/V3Proxy.sol#L160](https://github.com/GoodEntry-io/ge/blob/c7c7de57902e11e66c8186d93c5bb511b53a45b8/contracts/helper/V3Proxy.sol#L160)
>[https://github.com/GoodEntry-io/ge/blob/c7c7de57902e11e66c8186d93c5bb511b53a45b8/contracts/helper/V3Proxy.sol#L180](https://github.com/GoodEntry-io/ge/blob/c7c7de57902e11e66c8186d93c5bb511b53a45b8/contracts/helper/V3Proxy.sol#L180)
>[https://github.com/GoodEntry-io/ge/blob/c7c7de57902e11e66c8186d93c5bb511b53a45b8/contracts/helper/V3Proxy.sol#L199](https://github.com/GoodEntry-io/ge/blob/c7c7de57902e11e66c8186d93c5bb511b53a45b8/contracts/helper/V3Proxy.sol#L199)
>
>## Vulnerability details
>
>The original issue [M-08: Return value of low level call not checked](https://github.com/code-423n4/2023-08-goodentry-findings/issues/83), in scope for the mitigation review, was not acted upon, most likely overlooked during the fixing phase.
>
>## Assessed type
>
>call/delegatecall

## Summary

Several swap functions in **V3Proxy.sol** perform a low-level `call` to send ETH back to the user after unwrapping WETH, but **do not check the Boolean return value**. If the recipient's fallback or `receive()` reverts, the call silently fails and the function does **not** revert. As a result, ETH becomes irretrievably locked in the contract while the transaction appears to succeed.

## Root Cause

- **Unchecked Low-Level Call**:

    The pattern

    ```solidity
    payable(msg.sender).call{value: amount}("");
    ```

    is used without capturing or asserting the `(bool success)` return value.
- **Missing Error Handling**:

    No `require(success, "...")` or safe helper is used to revert on send failure.
- **Funds Lock**:

    A reversion in the recipient's fallback does not bubble up, so the swap proceeds without refunding ETH to the caller.

## A Better Explanation (With Simplified Example)

1. **User swaps tokens for ETH** via `swapTokensForExactETH`, which unwraps WETH and executes:

    ```solidity
    payable(msg.sender).call{value: amountOut}("");
    ```

2. **Recipient contract** has a `receive()` that `revert()`s unconditionally.
3. The low-level `call` returns `false`, but V3Proxy ignores it, and the function completes normally.
4. **Outcome**: The user loses their tokens, but the ETH stays trapped in V3Proxy, and no error is reported.

## Vulnerable Code Reference

```solidity
// V3Proxy.sol: unchecked ETH send (lines ~156, 174, 192)
acceptPayable = true;
weth.withdraw(amountOut);
acceptPayable = false;
// Unchecked low-level call:
payable(msg.sender).call{value: amountOut}("");
emit Swap(msg.sender, path[0], path[1], amounts[0], amounts[1]);
```

## Mitigation Recommendations

- **Always check the return value of low-level calls when sending ETH**:

    Instead of writing:

    ```solidity
    payable(msg.sender).call{value: amountOut}("");
    ```

    use:

    ```solidity
    (bool sent, ) = payable(msg.sender).call{value: amountOut}("");
    require(sent, "ETH send failed");
    ```

    This ensures the transaction reverts if the ETH transfer fails, avoiding locked funds.

- **Use a Safe ETH transfer helper from a trusted library**:

    Leverage OpenZeppelin's [Address.sendValue](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/v5.4.0/contracts/utils/Address.sol#L33) which safely sends ETH and reverts on failure:

    ```solidity
    import "@openzeppelin/contracts/utils/Address.sol";

    Address.sendValue(payable(msg.sender), amountOut);
    ```

- **Avoid silent failures when sending ETH**:

    Checking the success return prevents the contract from mistakenly assuming ETH was sent and protects users from stuck or lost funds.

- **Consider fallback scenarios**:

    When calling user contracts that may reject ETH, ensure the calling contract gracefully reverts or handles those failures rather than ignoring them.

## Pattern Recognition Notes

- Look for **low-level ETH transfers** using `.call{value: ...}("")` without `(bool success,)` capture.
- Check that such calls are followed by **`require(success, "...")`** or use **OpenZeppelin's `Address.sendValue`** which reverts on failure.
- Identify any **fallback/receive** functions in recipient contracts that may revert, causing silent failures.
- Flag functions that **unwrap WETH** then immediately `call` `msg.sender` without error handling.
- Ensure **atomicity**: a failed refund should revert the entire transaction to avoid locked funds.
- Common areas: swap wrappers, vaults, payout contracts where ETH is programmatically sent back.
