# Incorrect Daily Lending/Borrowing Cap Due to Off-by-One Scaling in V3Vault

- **Severity**: Medium
- **Source**: [Code4rena](https://github.com/code-423n4/2024-03-revert-lend-findings/issues/415)
- **Affected Contract**: [V3Vault.sol](https://github.com/code-423n4/2024-03-revert-lend/blob/435b054f9ad2404173f36f0f74a5096c894b12b7/src/V3Vault.sol)
- **Vulnerability Type**: Business Logic / Math Error

## Original Bug Description

>## Lines of code
>
>[https://github.com/code-423n4/2024-03-revert-lend/blob/435b054f9ad2404173f36f0f74a5096c894b12b7/src/V3Vault.sol#L1250-L1251](https://github.com/code-423n4/2024-03-revert-lend/blob/435b054f9ad2404173f36f0f74a5096c894b12b7/src/V3Vault.sol#L1250-L1251)
>[https://github.com/code-423n4/2024-03-revert-lend/blob/435b054f9ad2404173f36f0f74a5096c894b12b7/src/V3Vault.sol#L1262-L1263](https://github.com/code-423n4/2024-03-revert-lend/blob/435b054f9ad2404173f36f0f74a5096c894b12b7/src/V3Vault.sol#L1262-L1263)
>
>## Vulnerability details
>
>## Impact
>
>Protocol design includes limitiations on how much a user can deposit and borrow per day.
>So it is 10% of lent money or `dailyLendIncreaseLimitMin`, `dailyDebtIncreaseLimitMin`, whichever is greater.
>Current implementation is wrong and makes it 110% because of mistake in calculations.
>Which means that users are able to deposit/borrow close to 110% amount of current assets.
>
>Issue is in this calculations:
>
>```solidity
>    uint256 lendIncreaseLimit = _convertToAssets(totalSupply(), newLendExchangeRateX96, Math.Rounding.Up)
>                * (Q32 + MAX_DAILY_LEND_INCREASE_X32) / Q32;
>```
>
>```solidity
>   uint256 debtIncreaseLimit = _convertToAssets(totalSupply(), newLendExchangeRateX96, Math.Rounding.Up)
>                * (Q32 + MAX_DAILY_DEBT_INCREASE_X32) / Q32;
>```
>
>## Proof of Concept
>
>For borrow I used function `_setupBasicLoan()` already implemented by you in your tests.
>
>Add this tests beside your other tests in `test/integration/V3Vault.t.sol`
>
>```solidity
>     function testDepositV2() external {
>        vault.setLimits(0, 150000_000000, 0, 10_000000, 0);//10 usdc dailyLendIncreaseLimitMin
>        uint256 balance = USDC.balanceOf(WHALE_ACCOUNT);
>
>        vm.startPrank(WHALE_ACCOUNT);
>        USDC.approve(address(vault), type(uint256).max);
>        vault.deposit(10_000000 , WHALE_ACCOUNT);
>        skip(1 days);
>        for (uint i; i < 10; ++i) {
>            uint256 assets = vault.totalAssets();
>            console.log("USDC vault balance: %s", assets);
>            uint amount = assets + assets * 9 / 100;// 109% 
>            vault.deposit(amount, WHALE_ACCOUNT);
>            skip(1 days);
>        }
>        uint256 assets = vault.totalAssets();
>        assertEq(assets, 15902_406811);//so in 10 days we deposited 15k usdc, despite the 10 usdc daily limitation
>        console.log("USDC balance: %s", assets); 
>    }
>    
>    function testBorrowV2() external {
>        vault.setLimits(0, 150_000000, 150_000000, 150_000000, 1_000000);// 1 usdc dailyDebtIncreaseLimitMin
>        skip(1 days); //so we can recalculate debtIncreaseLimit again
>        _setupBasicLoan(true);//borrow  8_847206 which is > 1_000000 and > 10% of USDC10_000000 in vault
>    }
>```
>
>## Tools Used
>
>Manual review
>
>## Recommended Mitigation Steps
>
>Fix is simple for `_resetDailyLendIncreaseLimit()`
>
>```solidity
>    uint256 lendIncreaseLimit = _convertToAssets(totalSupply(), newLendExchangeRateX96, Math.Rounding.Up)
>-                * (Q32 + MAX_DAILY_LEND_INCREASE_X32) / Q32;
>+                * MAX_DAILY_LEND_INCREASE_X32 / Q32;
>```
>
>and for _resetDailyDebtIncreaseLimit()
>
>```solidity
>   uint256 debtIncreaseLimit = _convertToAssets(totalSupply(), newLendExchangeRateX96, Math.Rounding.Up)
>-                * (Q32 + MAX_DAILY_DEBT_INCREASE_X32) / Q32;
>+                * MAX_DAILY_DEBT_INCREASE_X32 / Q32;
>```
>
>## Assessed type
>
>Math

## Summary

The protocol implements daily caps limiting how much a user can deposit or borrow in a day, either as a percentage of total vault assets (e.g., 10%) or an absolute minimum floor — whichever is greater. However, due to a math error in calculating these limits, the actual cap is around **110%** of the vault's total assets instead of **10%**, allowing users to deposit or borrow substantially more than intended each day.

## A Better Explanation (With Simplified Example)

### Intended Behavior

- The daily limit should be:
`max(dailyIncreaseLimitMin, totalAssets * dailyIncreasePercentage)`
- Where `dailyIncreasePercentage` is represented in Q32 fixed-point format (e.g., 10% = 0.1 * 2³²).
- This prevents excessive daily deposit/borrow increases, enforcing gradual growth for risk management.

### What Actually Happens (Bug)

- The formula mistakenly computes:
`totalAssets * (1 + dailyIncreasePercentage)`
Because it adds the fixed-point base value (`Q32`, representing 1.0) *and* the `MAX_DAILY_XXX_INCREASE_X32` factor before dividing by `Q32`.
- This causes the daily limit to become **110%** (or higher, depending on the configured percent) of total assets, instead of just 10%.

### Why This Matters

- The vault's daily increase caps are circumvented, potentially allowing:
  - Sudden large inflows that risk over-utilization or mispricing.
  - Borrowers to over-leverage positions beyond intended safety thresholds.
- This undermines the protocol's risk control and gradual exposure management.

## Vulnerable Code Reference

```solidity
// Lines ~1250 and ~1262 in V3Vault.sol
function _resetDailyLendIncreaseLimit(uint256 newLendExchangeRateX96, bool force) internal {
    uint256 time = block.timestamp / 1 days;
    if (force || time > dailyLendIncreaseLimitLastReset) {
        uint256 lendIncreaseLimit = _convertToAssets(totalSupply(), newLendExchangeRateX96, Math.Rounding.Up)
            * (Q32 + MAX_DAILY_LEND_INCREASE_X32) / Q32;  // Bug: adds 1.0 (Q32) plus 10% incorrectly
        dailyLendIncreaseLimitLeft =
            dailyLendIncreaseLimitMin > lendIncreaseLimit ? dailyLendIncreaseLimitMin : lendIncreaseLimit;
        dailyLendIncreaseLimitLastReset = time;
    }
}

function _resetDailyDebtIncreaseLimit(uint256 newLendExchangeRateX96, bool force) internal {
    uint256 time = block.timestamp / 1 days;
    if (force || time > dailyDebtIncreaseLimitLastReset) {
        uint256 debtIncreaseLimit = _convertToAssets(totalSupply(), newLendExchangeRateX96, Math.Rounding.Up)
            * (Q32 + MAX_DAILY_DEBT_INCREASE_X32) / Q32;  // Bug repeated here as well
        dailyDebtIncreaseLimitLeft =
            dailyDebtIncreaseLimitMin > debtIncreaseLimit ? dailyDebtIncreaseLimitMin : debtIncreaseLimit;
        dailyDebtIncreaseLimitLastReset = time;
    }
}
```

## Recommended Mitigation

1. **Fix the math by removing the added "1.0" (Q32) factor**

    Change from

    ```solidity
    * (Q32 + MAX_DAILY_LEND_INCREASE_X32) / Q32;
    ```

    to

    ```solidity
    * MAX_DAILY_LEND_INCREASE_X32 / Q32;
    ```

    and similarly for the debt limit.

2. **Add thorough unit and integration tests verifying daily caps behave correctly for various vault sizes and minimum limits.**
3. **Consider adding event emissions on daily limit resets to improve observability.**

## Pattern Recognition Notes

This vulnerability arises from an off-by-one or incorrect scaling error in fixed-point math calculations that govern limits or thresholds. Such math errors are common and can lead to bypassing intended constraints, causing risk management failures. Here are key tips and patterns to recognize and avoid similar issues:

1. **Carefully Validate Fixed-Point Arithmetic**
    - Many protocols use fixed-point math (e.g., Q32, Q64) to represent decimals without floating point.
    - Check that scaling factors (`Q32`, `Q64`, etc.) are used correctly in multiplications and divisions.
    - Verify if addition or subtraction inside multiplications is appropriate, especially when representing percentages or ratios.
    - Avoid adding the scaling base (e.g., `Q32` representing 1.0) when the intention is to multiply by a fractional percentage only.

2. **Confirm What the Formula Intends to Compute vs. What It Actually Computes**
   - Write down the intended mathematical expression in human-readable form before coding.
   - For example, to get "10% of total," ensure the formula looks like: `total * 10%` → in fixed point, `total * factor / scaling`
   - Watch out for formulas that look like `total * (1 + factor)` when only `factor` should be applied.

3. **Beware of Off-By-One or Off-By-Base Errors**
   - Adding the fixed-point scaling base (like 1.0) to a percentage factor mistakenly increases the limit by 100% plus the factor.
   - This can cause limits to be much larger than intended, leading to critical risk breaches.

4. **Separate Minimum Floor Values from Percentage Calculations**
   - When protocols set thresholds as "max(min_limit, percentage_of_total)," verify both values individually.
   - Confirm that the logic comparing these two values picks the correct limit and that neither calculation is inflated.

5. **Write Unit Tests with Various Edge Cases and Size Scales**
   - Test boundary conditions with small, medium, and large vault sizes.
   - Include cases where minimum limits are below or above the percentage limits.
   - Simulate multiple days or increments to verify limits accumulate correctly and are reset timely.

6. **Use Code Reviews and Static Analysis Tools Focused on Math**
   - Review fixed-point math code carefully during audits.
   - Employ tools tailored to detecting math bugs or inconsistencies in Solidity or financial code.

7. **Monitor System Behavior Post-Deployment**
   - Emit events or logs when limits or caps are set or updated.
   - Watch for anomalous user activity exceeding intended thresholds.
   - Consider circuit breakers or emergency controls if limits appear bypassed.
