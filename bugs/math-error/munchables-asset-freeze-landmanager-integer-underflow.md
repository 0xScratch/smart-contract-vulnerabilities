# Asset Freezing via Flawed Reward Penalty Calculation in LandManager

* **Severity**: High
* **Source**: [Code4rena](https://github.com/code-423n4/2024-07-munchables-findings/issues/80) / [One Bug Per Day](https://www.onebugperday.com/v1/1414)
* **Affected Contract**: [LandManager.sol](https://github.com/code-423n4/2024-07-munchables/blob/94cf468aaabf526b7a8319f7eba34014ccebe7b9/src/managers/LandManager.sol)
* **Vulnerability Type**: Math Error / Integer Underflow

## Original Bug Description

>## Lines of code
>
>[https://github.com/code-423n4/2024-07-munchables/blob/94cf468aaabf526b7a8319f7eba34014ccebe7b9/src/managers/LandManager.sol#L232](https://github.com/code-423n4/2024-07-munchables/blob/94cf468aaabf526b7a8319f7eba34014ccebe7b9/src/managers/LandManager.sol#L232)
>
>## Vulnerability Details
>
>The function `_farmPlots()` is used to calculate rewards farming rewards. This function is used in different scenarios, including like a modifier in the function `unstakeMunchable()` function when a user unstakes their NFT. Let's have a look at how finalBonus is calculated in the function:
>
>```solidity
>finalBonus =
>    int16(
>        REALM_BONUSES[
>            (uint256(immutableAttributes.realm) * 5) +
>                uint256(landlordMetadata.snuggeryRealm)
>        ]
>    ) +
>    int16(
>        int8(RARITY_BONUSES[uint256(immutableAttributes.rarity)])
>    );
>```
>
>The `finalBonus` consists of bonus from the realm (`REALM_BONUSES`), as well as a rarity bonus (`RARITY_BONUSES`). `REALM_BONUSES` can be either -10, -5, 0, 5 or 10, and `RARITY_BONUSES` can be either 0, 10, 20, 30 or 50. `finalBonus` is meant to incentivize staking Munchables in a plot in a suitable realm. It can either be a positive bonus or reduction in the amount of `schnibblesTotal`. An example of a negative scenario is when `REALM_BONUSES` is -10 and `RARITY_BONUSES` is 0
>
>Let's look at the calculation of `schnibblesTotal`:
>
>```solidity
>schnibblesTotal =
>        (timestamp - _toiler.lastToilDate) *
>        BASE_SCHNIBBLE_RATE;
>schnibblesTotal = uint256(
>    (int256(schnibblesTotal) +
>        (int256(schnibblesTotal) * finalBonus)) / 100
>    );
>```
>
>`schnibblesTotal` is typed as `uint256`, therefore is meant as always positive. The current calculation, however, will result in a negative value of `schnibblesTotal`, when `finalBonus` < 0, the conversion to uint256 will cause the value to evaluate to something near `type(uint256).max`. In that case the next calculation of `schnibblesLandlord` will revert due to overflow:
>
>```solidity
>schnibblesLandlord =
>    (schnibblesTotal * _toiler.latestTaxRate) /
>    1e18;
>```
>
>This is due to the fact that `_toiler.latestTaxRate` > 1e16.
>
>Since `unstakeMunchable()` has a modifier `forceFarmPlots()`, the unstaking will be blocked if `_farmPlots()` reverts.
>
>## Scenario
>
>* Alice has a Common or Primordial Munchable NFT from Everfrost.
>* Bob has land (plots) in Drench.
>* Alice stakes her NFT in Bob's plot.
>* After some time Alice unstakes her NFT.
>* Since in the calculation of schnibblesTotal is used a negative finalBonus, the transaction will revert.
>* Alice's Munchable NFT and any other staked beforehand are blocked.
>
>## Proof of Concept
>
>Add the following lines of code in `tests/managers/LandManager/unstakeMunchable.test.ts`:
>
>```typescript
>.
>.
>.
>await registerPlayer({
>   account: alice,
>+  realm: 1,
>   testContracts,
>});
>await registerPlayer({
>   account: bob,
>   testContracts,
>});
>await registerPlayer({
>   account: jirard,
>   testContracts,
>});
>.
>.
>.
>await mockNFTOverlord.write.addReveal([alice, 100], { account: alice });
>   await mockNFTOverlord.write.addReveal([bob, 100], { account: bob });
>   await mockNFTOverlord.write.addReveal([bob, 100], { account: bob });
>   await mockNFTOverlord.write.startReveal([bob], { account: bob }); // 1
>-  await mockNFTOverlord.write.reveal([bob, 4, 12], { account: bob }); // 1
>+  await mockNFTOverlord.write.reveal([bob, 1, 0], { account: bob }); // 1
>   await mockNFTOverlord.write.startReveal([bob], { account: bob }); // 2
>   await mockNFTOverlord.write.reveal([bob, 0, 13], { account: bob }); // 2
>   await mockNFTOverlord.write.startReveal([bob], { account: bob }); // 3
>```
>
>When we run the command `pnpm test:typescript` the logs from the tests show that the `successful path` test reverts.
>
>## Impact
>
>A user's Munchable NFT with specific attributes can be stuck in the protocol.
>
>Taking into account the rarity distribution across the different NFT-s and assuming that the distribution across realms is equal, the chance of this problem occurring is around `26%` or 1 in every 4 staked munchables.
>
>Further, this finding blocks unstaking every munchable that has previously been staked.
>
>## Recommended Mitigation
>
>`finalBonus` is meant as a percentage change in the Schnibbles earned by a Munchable. Change the calculation of `schnibblesTotal` in `_farmPlots()` to reflect that by removing the brackets:
>
>```solidity
>schnibblesTotal =
>        (timestamp - _toiler.lastToilDate) *
>        BASE_SCHNIBBLE_RATE;
>-schnibblesTotal = uint256(
>-    (int256(schnibblesTotal) +
>-        (int256(schnibblesTotal) * finalBonus)) / 100
>-    );
>+schnibblesTotal = uint256(
>+    int256(schnibblesTotal) + 
>+       (int256(schnibblesTotal) * finalBonus) / 100
>+    );
>```
>
>## Assessed type
>
>Math

## Summary

The protocol calculates staking rewards ("schnibbles") which can be increased by a bonus or decreased by a penalty, depending on the staked NFT's attributes relative to the plot of land. The formula to apply a penalty is mathematically incorrect. When a user tries to unstake an NFT that has accrued a penalty, this flawed calculation causes an **integer underflow**, which then leads to a transaction **revert**. Because the reward calculation is a mandatory step in the unstaking process, this bug **permanently blocks users from unstaking** not only the penalized NFT but all other NFTs they have staked as well, effectively freezing their assets in the contract.

## A Better Explanation (With Simplified Example)

### Intended Behavior

* The reward system should correctly apply a bonus or a penalty. A penalty is just a negative bonus.
* The intended formula is:
  `Final Rewards = Base Rewards + (Base Rewards * Penalty Percentage)`
* For example, with 1,000 base rewards and a -10% penalty, the result should be:
  `1,000 + (1,000 * -0.10) = 900`.
* This calculation should complete successfully, allowing the user to unstake their NFT.

### What Actually Happens (Bug)

* The formula mistakenly computes:
  `(Base Rewards + (Base Rewards * Penalty)) / 100`
* Using the same example, the code calculates:
  `(1,000 + (1,000 * -10)) / 100` -\> `(1,000 - 10,000) / 100` -\> `-9,000 / 100 = -90`.
* The code then tries to store this negative result (`-90`) in a `uint256` variable, which can only hold positive values. This causes an **integer underflow**, making the value wrap around to a massive positive number.
* This massive number then causes the next calculation (for the landlord's tax) to overflow and revert the entire transaction.

### Why This Matters

* The `unstakeMunchable` function is decorated with a modifier that forces the reward calculation (`_farmPlots`) to run first.
* If the reward calculation fails for even **one** of a user's staked NFTs, the entire unstaking transaction reverts.
* This creates a permanent **Denial of Service** (DoS) condition, trapping user assets in the contract with no way to recover them. The report estimates this could affect up to 26% of staked NFTs.

## Vulnerable Code Reference

```solidity
// Line ~303 in LandManager.sol
function _farmPlots(address _sender) internal {
    // ... loop through staked NFTs ...

    schnibblesTotal =
        (timestamp - _toiler.lastToilDate) *
        BASE_SCHNIBBLE_RATE;

    // The bug is in the following calculation due to incorrect order of operations
    schnibblesTotal = uint256(
        (int256(schnibblesTotal) +
            (int256(schnibblesTotal) * finalBonus)) / 100
    );

    // ... this next line reverts due to overflow if the above underflowed
    schnibblesLandlord =
        (schnibblesTotal * _toiler.latestTaxRate) /
        1e18;

    // ...
}
```

## Mitigation

1. **Fix the math by correcting the order of operations.** The division by 100 should happen *before* the final addition to correctly calculate the percentage.

    Change from:

    ```solidity
    schnibblesTotal = uint256(
        (int256(schnibblesTotal) +
            (int256(schnibblesTotal) * finalBonus)) / 100
    );
    ```

    to:

    ```solidity
    schnibblesTotal = uint256(
        int256(schnibblesTotal) +
           (int256(schnibblesTotal) * finalBonus) / 100
    );
    ```

2. **Add thorough unit tests** that specifically cover scenarios with negative bonuses (penalties) to ensure the calculation is correct and does not revert.

## Pattern Recognition Notes

This vulnerability highlights critical patterns related to integer arithmetic and contract design. Developers can use these takeaways to avoid similar issues.

1. **Carefully Validate Mixed-Sign Arithmetic**

      * Operations involving both positive and negative numbers are a common source of bugs.
      * The order of operations is critical. `(A + (A*B)) / C` is vastly different from `A + (A*B / C)`.
      * **Always be suspicious of casting a signed integer (`int256`) to an unsigned one (`uint256`)**. This is a major red flag and should only be done after guaranteeing the value is non-negative.

2. **Confirm What the Formula Intends vs. What It Computes**

      * Before writing the code, write the financial or mathematical formula in plain terms.
      * For example: "The final reward is the base reward minus a 10% penalty."
      * Ensure the code (`final = base - (base * 10 / 100)`) directly reflects that logic, rather than a convoluted alternative.

3. **Beware of Unsafe Modifiers Creating DoS Conditions**

      * The `forceFarmPlots` modifier made a calculation bug a critical asset-freezing bug.
      * If a modifier performs complex operations that can revert (like reward calculations), it can block the execution of the function it modifies.
      * Consider if such logic should be an explicit, separate function call by the user rather than a mandatory, automatic step within a critical function like `unstake`.

4. **Isolate Complex Calculations into Components**

      * Instead of a single, complex one-liner, break down the calculation into clear, verifiable steps.
      * This improves readability and reduces the chance of error.

        ```solidity
        // Safer approach
        int256 baseReward = ...;
        int256 penaltyAmount = (baseReward * finalBonus) / 100;
        int256 finalReward = baseReward + penaltyAmount;
        require(finalReward >= 0, "Reward cannot be negative");
        schnibblesTotal = uint256(finalReward);
        ```

5. **Write Tests for Unhappy Paths and Edge Cases**

      * The "happy path" (a positive bonus) likely worked perfectly during testing.
      * It is crucial to test the "unhappy paths": what happens when values are negative, zero, or at their maximum/minimum limits? In this case, testing with a `finalBonus < 0` would have immediately revealed the bug.
