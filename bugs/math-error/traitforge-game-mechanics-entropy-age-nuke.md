# Age Underestimation Due to Early Integer Division in `calculateAge()`

* **Severity**: Medium
* **Source**: [Code4rena](https://github.com/code-423n4/2024-07-traitforge-findings/issues/223) / [One Bug Per Day](https://www.onebugperday.com/v1/1452)
* **Affected Contract**: [NukeFund.sol](https://github.com/code-423n4/2024-07-traitforge/blob/main/contracts/NukeFund/NukeFund.sol)
* **Vulnerability Type**: Math Precision Error

## Original Bug Description

>## Lines of code
>
>[https://github.com/code-423n4/2024-07-traitforge/blob/main/contracts/NukeFund/NukeFund.sol#L118-L133](https://github.com/code-423n4/2024-07-traitforge/blob/main/contracts/NukeFund/NukeFund.sol#L118-L133)
>
>## Vulnerability details
>
>## Impact
>
>The function `calculateAge()` computes the age of a token ID based on the current and creation time difference. This token age is then used to determine the [nuke factor](https://github.com/code-423n4/2024-07-traitforge/blob/main/contracts/NukeFund/NukeFund.sol#L143) to calculate the possible claim amount.
>
>The problem here is that when calculating the age of a token, it first divides the time difference to 1 day in seconds, and then multiplies the result to the other variables.
>
>```solidity
>  function calculateAge(uint256 tokenId) public view returns (uint256) {
>    require(nftContract.ownerOf(tokenId) != address(0), 'Token does not exist');
>
>    uint256 daysOld = (block.timestamp -
>      nftContract.getTokenCreationTimestamp(tokenId)) /
>      60 /
>      60 /
>      24;
>    uint256 perfomanceFactor = nftContract.getTokenEntropy(tokenId) % 10;
>
>    uint256 age = (daysOld *
>      perfomanceFactor *
>      MAX_DENOMINATOR *
>      ageMultiplier) / 365; // add 5 digits for decimals
>    return age;
>  }
>```
>
>With having this pattern, users will face significant losses. Especially, in the case of small differences (less than 1 day), the `daysOld` variable becomes `0`, making the `age` zero respectively.
>
>## Proof of Concept
>
>The function below illustrates the significant discrepancies between the results:
>
>```solidity
>// SPDX-License-Identifier: UNLICENSED
>pragma solidity ^0.8.17;
>
>import "forge-std/Test.sol";
>
>
>contract DifferenceTest is Test {
>
>    uint256 public constant MAX_DENOMINATOR = 100000;
>    error InvalidTokenAmount();
>
>    function setUp() public {
>
>    }
>
>    function calculateAge_precision(
>        uint256 creationTimestamp,
>        uint256 startTime,
>        uint256 ageMultiplier
>    )  public view returns (uint256) {
>
>        uint256 perfomanceFactor = 150000;
>
>        uint256 age = ((startTime - creationTimestamp) *
>        perfomanceFactor *
>        MAX_DENOMINATOR *
>        ageMultiplier) / (60 * 60 * 24 * 365);
>        return age;
>    }
>
>    function calculateAge_actual(
>        uint256 creationTimestamp,
>        uint256 startTime,
>        uint256 ageMultiplier
>    )  public view returns (uint256) {
>
>        uint256 perfomanceFactor = 150000;
>
>        uint256 daysOld = (startTime -
>        creationTimestamp) /
>        60 /
>        60 /
>        24;
>
>        uint256 age = (daysOld *
>        perfomanceFactor *
>        MAX_DENOMINATOR *
>        ageMultiplier) / 365; // add 5 digits for decimals
>        return age;
>    }
>
>    function test_diffAges(
>        uint256 creationTimestamp,
>        uint256 startTime,
>        uint256 ageMultiplier
>    )  public {
>
>        vm.assume(startTime > creationTimestamp);
>        vm.assume(ageMultiplier < MAX_DENOMINATOR);
>        uint actualAge = calculateAge_actual(creationTimestamp, startTime, ageMultiplier);
>        uint accurateAge = calculateAge_precision(creationTimestamp, startTime, ageMultiplier);
>        console.log("The actual age is:   ", actualAge);
>        console.log("The accurate age is: ", accurateAge);
>        assertFalse(actualAge == accurateAge);
>    }
>}
>```
>
>For these results we will have:
>
>(`creationTimestamp = 1722511200`,
>
>`startTime = 1722799000`,
>
>`ageMultiplier = 150000`,
>
>`perfomanceFactor = 15620`):
>
>```logs
>[PASS] test_diffAges() (gas: 8201)
>Logs:
>  The actual age is:    1925753424657
>  The accurate age is:  2138240106544
>
>Traces:
>  [8201] DifferenceTest::test_diffAges()
>    ├─ [0] console::log("The actual age is:   ", 1925753424657 [1.925e12]) [staticcall]
>    │   └─ ← [Stop]
>    ├─ [0] console::log("The accurate age is: ", 2138240106544 [2.138e12]) [staticcall]
>    │   └─ ← [Stop]
>    ├─ [0] VM::assertFalse(false) [staticcall]
>    │   └─ ← [Return]
>    └─ ← [Stop]
>
>Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 520.80µs (150.20µs CPU time)
>```
>
>The calculated age is: 1925753424657
>The accurate age is : 2138240106544
>
>This will lead to ~ 11% of error which is a significant error here.
>This huge difference will also increase the nuke factor (even more) and affect the results.
>
>## Tools Used
>
>Manual Review
>
>## Recommended Mitigation Steps
>
>Consider multiplying the numerator variables first and then divide by the denominator:
>
>```logs
>  function calculateAge(uint256 tokenId) public view returns (uint256) {
>    require(nftContract.ownerOf(tokenId) != address(0), 'Token does not exist');
>
>-    uint256 daysOld = (block.timestamp -
>-      nftContract.getTokenCreationTimestamp(tokenId)) /
>-      60 /
>-      60 /
>-      24;
>    uint256 perfomanceFactor = nftContract.getTokenEntropy(tokenId) % 10;
>
>-    uint256 age = (daysOld *
>+    uint256 age = (perfomanceFactor *
>      MAX_DENOMINATOR *
>+      (block.timestamp - nftContract.getTokenCreationTimestamp(tokenId)) *
>      ageMultiplier) / (60 * 60 * 24 * 365); // add 5 digits for decimals
>    return age;
>  }
>```
>
>## Assessed type
>
>Math

## Summary

The `calculateAge()` function in `NukeFund.sol` calculates NFT "age" in days, which feeds into the **nuke factor** — a multiplier for determining ETH payout when burning NFTs. However, the current implementation **converts seconds to days using integer division early**, discarding any fractional day before applying performance and multiplier factors.

This truncation causes up to **nearly 1 full day** of age to be ignored, leading to **underestimated nuke factors** and **reduced player payouts**.

## A Better Explanation (With Simplified Example)

### Intended Behavior

* Age should reflect **exact elapsed time** (including fractional days) since NFT creation:

  ```plaintext
  age = (elapsed_seconds / seconds_per_year) * performance_factor * MAX_DENOMINATOR * age_multiplier
  ```

* This ensures partial days are proportionally included, so a token aged 2.9 days is worth more than one aged exactly 2 days.

---

### What Actually Happens (Bug)

* Current code:

  ```solidity
  uint256 daysOld = (block.timestamp - creationTime) / 60 / 60 / 24;
  uint256 age = (daysOld * perfomanceFactor * MAX_DENOMINATOR * ageMultiplier) / 365;
  ```

* **Problem:** `(block.timestamp - creationTime) / 60 / 60 / 24` truncates to whole days **before** multiplying by performance and age multipliers.

* A token aged 2.98 days is treated as exactly 2 days.

---

**Example:**

| Real Age | Current Code Days | Correct Age Fraction | Age Error            |
| -------- | ----------------- | -------------------- | -------------------- |
| 2.98 d   | 2                 | 2.98                 | -0.98 d (\~33% loss) |

When converted to ETH payout, this can cause **\~10-15% lower claim** depending on other factors.

### Why This Matters

* **Economic fairness:** Players lose ETH for partial days worked toward nuke factor growth.
* **Timing exploit:** Users who wait until the next full-day rollover get a sudden jump in payout, incentivizing unnatural game timing.
* **Inconsistency:** Nuke factor progression becomes "stair-stepped" instead of smooth.

## Vulnerable Code Reference

```solidity
// NukeFund.sol
function calculateAge(uint256 tokenId) public view returns (uint256) {
    uint256 creationTime = nftCreationTime[tokenId];
    uint256 daysOld = (block.timestamp - creationTime) / 60 / 60 / 24; // early truncation
    uint256 age = (daysOld * perfomanceFactor * MAX_DENOMINATOR * ageMultiplier) / 365;
    return age;
}
```

## Recommended Mitigation

1. **Preserve fractional days by avoiding early division.**
   Replace with:

   ```solidity
   uint256 secondsOld = block.timestamp - creationTime;
   uint256 age = (secondsOld * perfomanceFactor * MAX_DENOMINATOR * ageMultiplier) / (60 * 60 * 24 * 365);
   ```

2. **Add tests covering fractional-day scenarios** (e.g., 0.5 days, 1.25 days, 2.98 days) to ensure payouts scale smoothly.

3. **Audit other time-based calculations** in the codebase to ensure no similar truncation occurs prematurely.

## Pattern Recognition Notes

When looking for similar math-related vulnerabilities in other protocols, keep these checks in mind:

1. **Division Without Scaling or Order Control**

   * If a contract performs `a / b * c` with integers, small values might be lost due to truncation before multiplication.
   * **Look for:** Any math operation where division comes before multiplication, especially in reward, fee, or probability calculations.

2. **Rounding Bias**

   * Solidity's integer division always rounds down (towards zero). Repeated calculations can accumulate this rounding error in a way that benefits one side and disadvantages another.
   * **Look for:** Repeated integer math in loops, reward distribution formulas, or percentage calculations.

3. **Unbounded Growth or Decay**

   * Systems that update values over "time" (like Age or Entropy) can spiral if growth/decay rates are miscalculated.
   * **Look for:** Multipliers applied each update cycle — especially if they can exceed expected ranges.

4. **State Reset or Overflow Triggers**

   * Big "reset" actions (like Nukes) can unintentionally break invariants if they don't fully reset related variables.
   * **Look for:** Any function that zeroes or massively alters a state variable — verify it resets *all* dependent values.

5. **Time-based Calculation Assumptions**

   * Age/entropy style mechanics often assume consistent block timestamps or intervals. Manipulated or inconsistent timing can break formulas.
   * **Look for:** Dependencies on `block.timestamp` or assumptions that each update happens at fixed intervals.
