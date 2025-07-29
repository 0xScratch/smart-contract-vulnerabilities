# Incremental Compensation Blocked by Strict CR Check in `compensate()`

- **Severity**: Medium
- **Source**: [Code4rena](https://github.com/code-423n4/2024-06-size-findings/issues/107) / [One Bug Per Day](https://www.onebugperday.com/v1/1349)
- **Affected Contract**: [Size.sol](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/src/Size.sol)
- **Vulnerability Type**: Business Logic Flaw - Restrictive User-State Validation

## Original Bug Description

>## Lines of code
>
>[https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/src/Size.sol#L250](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/src/Size.sol#L250)
>
>## Vulnerability details
>
>## Impact
>
>A borrower of a loan could have other credit positions with borrowers that have healthy CRs which he could use to compensate his lenders to avoid liquidations and improve his CR. However he is not able to do this via [`compensate()`](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/src/Size.sol#L247) and he is forced to sell his credit positions via [`sellCreditMarket()`](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/src/Size.sol#L188) or [`sellCreditLimit()`](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/src/Size.sol#L172). The complications of this are described in the example below.
>
>## Proof of Concept
>
>Before we start just to have context, [`repay()`](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/src/Size.sol#L198) repays the whole debt amount of the loan in full and can be called even when the borrower is underwater or past due date.
>
>[`compensate()`](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/src/Size.sol#L247) is used to either split a loan in smaller parts in order to be more easily repaid with [`repay()`](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/src/Size.sol#L198) or to repay certain lenders and reduce the debt of the loan by the borrower giving specific lenders some of his own healthy credit positions that he owns.
>
>Here is an example of the issue:
>
>Bob has collateral X in place and has 10 different debt positions (he borrowed).
>With the newly acquired borrow tokens he buys 10 different credit positions.
>
>Some time passes and his collateral value drops significantly (volatile market), his CR is below the healthy threshold. At the moment Bob does not have any more borrow tokens available but he has 10 healthy credit positions that he can use to improve his CR by compensating some of his lenders.
>
>Here comes the important part.
>
>He can successfully compensate 1 of his lenders if at the end of the [`compensate()`](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/src/Size.sol#L247) tx his CR becomes healthy. The problem appears when he must compensate for 3 out of all 10 debt positions (more than 1 position) in order to become with healthy CR.
>
>He will call compensate for the first time and the end of this tx he will face a revert due to this check because he needs to compensate 2 more times to become healthy.
>
>Link to code: [`link`](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/src/Size.sol#L247-L251)
>
>```solidity
>    function compensate(CompensateParams calldata params) external payable override(ISize) whenNotPaused {
>        state.validateCompensate(params);
>        state.executeCompensate(params);
>@>      state.validateUserIsNotUnderwater(msg.sender);
>    }
>```
>
>Link to code: [`link`](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/src/libraries/RiskLibrary.sol#L117-L133)
>
>```solidity
>    function isUserUnderwater(State storage state, address account) public view returns (bool) {
>@>      return collateralRatio(state, account) < state.riskConfig.crLiquidation;
>    }
>
>    function validateUserIsNotUnderwater(State storage state, address account) external view {
>@>      if (isUserUnderwater(state, account)) {
>            revert Errors.USER_IS_UNDERWATER(account, collateralRatio(state, account));
>        }
>    }
>```
>
>The only option Bob has currently is to sell some of his credit positions via `sellCreditMarket` or `sellCreditLimit`. There might not be any buyers at the moment or he might be forced to take a very bad deal for his credit positions because he will be in a rush in order to repay part of his loans on time to not get liquidated.
>
>## Tools Used
>
>Manual Review
>
>## Recommended Mitigation Steps
>
>The CR check at the end of compensate is important because fragmentation fees could occur and lower the CR and make it under the healthy threshold in some specific situations.
>
>What I propose as a solution is measuring the CR before and after the [`compensate()`](https://github.com/code-423n4/2024-06-size/blob/8850e25fb088898e9cf86f9be1c401ad155bea86/src/Size.sol#L247) logic and it should revert if the CR is becoming worse.
>
>This way Bob will be able to improve his CR by using his credit positions even if he has to compensate multiple times before becoming healthy again.

## Summary

The `compensate()` function forbids any partial compensation that does not immediately restore a borrower's collateral ratio (CR) above the liquidation threshold. As a result, underwater borrowers cannot gradually repay multiple debt positions using healthy credit positions; every single `compensate()` call must fully bring the CR back to safety, or it reverts—and undoes even beneficial, incremental improvements.

## Detailed Explanation

1. **Intended Purpose**
    - Let an underwater borrower transfer ("compensate") some healthy credit positions to specific lenders, reducing debt and improving CR without selling on the open market.
2. **Actual Behavior**

    ```solidity
    function compensate(CompensateParams calldata params) external payable whenNotPaused {
        state.validateCompensate(params);
        state.executeCompensate(params);
        state.validateUserIsNotUnderwater(msg.sender);  // ← Enforces CR ≥ liquidation threshold
    }
    ```

    - After executing compensation, the contract immediately checks `validateUserIsNotUnderwater`.
    - If CR remains below `crLiquidation`, the call reverts and rolls back—even if CR improved compared to before.
3. **Reproduction Scenario**
    - **Setup**: Bob has 10 loans (unhealthy CR) and 10 healthy credit positions.
    - **Goal**: Use three sequential compensations to pay down three loans. Each partial step slightly raises CR.
    - **Outcome**:
        1. First `compensate()` call: state.executeCompensate succeeds but final CR still < threshold → revert.
        2. Bob cannot make progress; partial improvements are impossible.
        3. Forced to use `sellCreditMarket()` or `sellCreditLimit()`, risking poor execution and slippage.

## Main Impact

- **User Experience**: Borrowers cannot leverage built-in compensation to recover; they must sell into possibly illiquid markets.
- **Economic Efficiency**: Increased slippage and fragmented liquidity; borrowers suffer worse rates.
- **Protocol Robustness**: Key feature (`compensate`) effectively disabled for many real-world scenarios.

## Code References

```solidity
// Size.sol
function compensate(CompensateParams calldata params) external payable whenNotPaused {
    state.validateCompensate(params);
    state.executeCompensate(params);
    state.validateUserIsNotUnderwater(msg.sender);
}

// RiskLibrary.sol
function validateUserIsNotUnderwater(State storage state, address account) external view {
    if (collateralRatio(state, account) < state.riskConfig.crLiquidation) {
        revert Errors.USER_IS_UNDERWATER(account, collateralRatio(state, account));
    }
}
```

## Recommended Mitigation

Allow incremental improvements so long as CR does not worsen:

1. **Measure CR Before \& After**

    ```solidity
    uint256 crBefore = collateralRatio(state, msg.sender);
    state.executeCompensate(params);
    uint256 crAfter = collateralRatio(state, msg.sender);
    ```

2. **Enforce Non-Regression Only**

    ```solidity
    if (crAfter < crBefore) {
        revert Errors.COMPENSATION_MADE_HEALTH_WORSE(crBefore, crAfter);
    }
    ```

    - Permits calls where `crAfter ≥ crBefore`, even if still underwater.
    - Prevents any compensation that would actually degrade CR (e.g., due to fragmentation fees).

This adjustment restores the intended use of `compensate()`, enabling borrowers to gradually recover from an underwater state without being forced into suboptimal market sales.

## Pattern Recognition Notes

- **Too Strict Rules Can Block Progress**
  - Some functions stop users from making partial improvements unless the problem is fixed all at once. This can trap users and stop them from slowly fixing their loans step-by-step.

- **Allow Small Steps to Recovery**
  - When users need to fix things over multiple tries, each try should be allowed to help, even if it doesn't completely solve the problem right away.

- **Don't Reject Helpful Actions Just Because The Whole Problem Isn't Solved Yet**
  - Sometimes a user action makes things better but not perfect. The contract should not cancel these helpful actions just because the user is still not fully "healthy."

- **Understand The Difference Between Security and Business Rules**
  - Security checks stop harmful actions, but business rules manage how users interact with the protocol. Mixing these can accidentally block normal, helpful use.

- **Fees and Small Costs Can Make Things Look Worse Temporarily**
  - Sometimes paying fees or other small costs during actions can make numbers temporarily look worse. The contract should allow actions that don't make things actually worse overall.

- **Actions That Work Together Should Be Allowed To Complete One By One**
  - In protocols where users can do many actions in one go (like multicall), each small action should be allowed to succeed if it doesn't cause harm—no need to block partial progress.

- **Quick Audit Tips**
  - Check if the contract immediately cancels actions if things aren't perfect after.
  - See if it compares new states to hard limits or to the state before the action.
  - Make sure it lets users improve step-by-step without forcing an all-in fix.
  - Watch for extra fees or side effects that might cause false failures.
  - Prefer rules that only stop actions if they actually make things worse, not if they just don't fix everything right now.
