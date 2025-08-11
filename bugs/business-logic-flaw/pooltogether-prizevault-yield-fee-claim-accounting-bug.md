# Loss of Unclaimed Yield Fees Due to Partial Claim Reset in PrizeVault

* **Severity**: High
* **Source**: [Code4rena](https://github.com/code-423n4/2024-03-pooltogether-findings/issues/59) / [One Bug Per Day](https://www.onebugperday.com/v1/1082)
* **Affected Contract**: [PrizeVault.sol](https://github.com/code-423n4/2024-03-pooltogether/blob/main/pt-v5-vault/src/PrizeVault.sol)
* **Vulnerability Type**: Business Logic / State Accounting Error

## Original Bug Description

>## Lines of code
>
>[https://github.com/code-423n4/2024-03-pooltogether/blob/main/pt-v5-vault/src/PrizeVault.sol#L611-L622](https://github.com/code-423n4/2024-03-pooltogether/blob/main/pt-v5-vault/src/PrizeVault.sol#L611-L622)
>
>## Vulnerability details
>
>## Impact
>
>Any fee claim by the fee recipient lesser than the accrued internal accounting of the `yieldFeeBalance` is lost and locked in the `PrizeVault` contract with no way to pull out the funds.
>
>## Proof of Concept
>
>The `claimYieldFeeShares` allows the `yieldFeeRecipient` fee recipient to claim fees in yields from the `PrizeVault` contract. The claimer can claim up to the `yieldFeeBalance` internal accounting and no more. The issue with this function is it presents a vulnerable area of loss with the `_shares` argument in the sense that if the accrued yield fee shares is 1000 shares and the claimer claims only 10, 200 or even any amount less than 1000, they forfeit whatever is left of the `yieldFeeBalance` e.g if you claimed 200 and hence got minted 200 shares, you lose the remainder 800 because it wipes the `yieldFeeBalance` 1000 balance whereas only minted 200 shares.
>
>Let's see a code breakdown of the vulnerable `claimYieldFeeShares` function:
>
>```solidity
>function claimYieldFeeShares(uint256 _shares) external onlyYieldFeeRecipient {
>        if (_shares == 0) revert MintZeroShares();
>
>        uint256 _yieldFeeBalance = yieldFeeBalance;
>        if (_shares > _yieldFeeBalance) revert SharesExceedsYieldFeeBalance(_shares, _yieldFeeBalance);
>
>        yieldFeeBalance -= _yieldFeeBalance; // @audit issue stems and realized next line of code
>
>        _mint(msg.sender, _shares); // @audit the point where the claimant gets to lose
>
>        emit ClaimYieldFeeShares(msg.sender, _shares);
>    }
>```
>
>This line of the function caches the total yield fee balance accrued in the contract and hence, the fee recipient is entitled to e.g 100
>
>```solidity
>uint256 _yieldFeeBalance = yieldFeeBalance;
>```
>
>This next line of code enforces a comparison check making sure the claimer cannot grief other depositors in the vault because the claimant could for example try to claim and mint 150 shares whereas they are only entitled to 100.
>
>```solidity
>if (_shares > _yieldFeeBalance) revert SharesExceedsYieldFeeBalance(_shares, _yieldFeeBalance);
>```
>
>This line of code subtracts the cached total yield fee balance from the state yield fee balance e.g 100 - 100. So if say Bob the claimant tried to only mint 50 shares at this point in time with the `_shares` argument, the code wipes the entire balance of 100
>
>```solidity
>yieldFeeBalance -= _yieldFeeBalance;
>```
>
>And this line of code then mints the specified `_shares` amount e.g 50 shares to Bob.
>
>```solidity
>_mint(msg.sender, _shares);
>```
>
>So what essentially happens is:
>
>* Total accrued fee is 100
>* Bob claims 50 shares of the 100
>* Bob gets minted 50 shares
>* Bob loses the rest 50 shares
>
>Here's a POC for this issue. Place the `testUnclaimedFeesLostPOC` function inside the `PrizeVault.t.sol` file and run the test.
>
>```solidity
>function testUnclaimedFeesLostPOC() public {
>        vault.setYieldFeePercentage(1e8); // 10%
>        vault.setYieldFeeRecipient(bob); // fee recipient bob
>        assertEq(vault.totalDebt(), 0); // no deposits in vault yet
>
>        // alice makes an initial deposit of 100 WETH
>        underlyingAsset.mint(alice, 100e18);
>        vm.startPrank(alice);
>        underlyingAsset.approve(address(vault), 100e18);
>        vault.deposit(100e18, alice);
>        vm.stopPrank();
>
>        console.log("Shares balance of Alice post mint: ", vault.balanceOf(alice));
>
>        assertEq(vault.totalAssets(), 100e18);
>        assertEq(vault.totalSupply(), 100e18);
>        assertEq(vault.totalDebt(), 100e18);
>
>        // mint yield to the vault and liquidate
>        underlyingAsset.mint(address(vault), 100e18);
>        vault.setLiquidationPair(address(this));
>        uint256 maxLiquidation = vault.liquidatableBalanceOf(address(underlyingAsset));
>        uint256 amountOut = maxLiquidation / 2;
>        uint256 yieldFee = (100e18 - vault.yieldBuffer()) / (2 * 10); // 10% yield fee + 90% amountOut = 100%
>        vault.transferTokensOut(address(0), bob, address(underlyingAsset), amountOut);
>        console.log("Accrued yield post in the contract to be claimed by Bob: ", vault.yieldFeeBalance());
>        console.log("Yield fee: ", yieldFee);
>        // yield fee: 4999999999999950000
>        // alice mint: 100000000000000000000
>
>        assertEq(vault.totalAssets(), 100e18 + 100e18 - amountOut); // existing balance + yield - amountOut
>        assertEq(vault.totalSupply(), 100e18); // no change in supply since liquidation was for assets
>        assertEq(vault.totalDebt(), 100e18 + yieldFee); // debt increased since we reserved shares for the yield fee
>
>        vm.startPrank(bob);
>        vault.claimYieldFeeShares(1e17);
>        
>        console.log("Accrued yield got reset to 0: ", vault.yieldFeeBalance());
>        console.log("But the shares minted to Bob (yield fee recipient) should be 4.9e18 but he only has 1e17 and the rest is lost: ", vault.balanceOf(bob));
>
>        // shares bob: 100000000000000000
>        assertEq(vault.totalDebt(), vault.totalSupply());
>        assertEq(vault.yieldFeeBalance(), 0);
>        vm.stopPrank();
>    }
>```
>
>```logs
>Test logs and results:
>Logs:
>  Shares balance of Alice post mint:  100000000000000000000
>  Accrued yield in the contract to be claimed by Bob:  4999999999999950000
>  Yield fee:  4999999999999950000
>  Accrued yield got reset to 0:  0
>  But the shares minted to Bob (yield fee recipient) should be 4.9e18 but he only has 1e17 and the rest is lost:  100000000000000000
>```
>
>## Tools Used
>
>Manual review + foundry
>
>## Recommended Mitigation Steps
>
>Adjust the `claimYieldFeeShares` to only deduct the amount claimed/minted
>
>```diff
>function claimYieldFeeShares(uint256 _shares) external onlyYieldFeeRecipient {
>  if (_shares == 0) revert MintZeroShares();
>
>-  uint256 _yieldFeeBalance = yieldFeeBalance;
>-  if (_shares > _yieldFeeBalance) revert SharesExceedsYieldFeeBalance(_shares, _yieldFeeBalance);
>+  if (_shares > yieldFeeBalance) revert SharesExceedsYieldFeeBalance(_shares, yieldFeeBalance);
>
>-  yieldFeeBalance -= _yieldFeeBalance;
>+  yieldFeeBalance -= _shares;
>
>  _mint(msg.sender, _shares);
>
>  emit ClaimYieldFeeShares(msg.sender, _shares);
>}
>```
>
>## Assessed type
>
>Other

## Summary

The `PrizeVault` contract accrues yield fees in `yieldFeeBalance` and allows the designated fee recipient to claim them in vault shares using `claimYieldFeeShares()`.
However, **if the recipient claims less than the full `yieldFeeBalance`**, the function **still resets the entire balance to zero**, discarding the unclaimed portion.
This leads to **permanent loss of unclaimed yield fees** and breaks the intended fee accrual lifecycle.

## A Better Explanation (With Simplified Example)

### Intended Behavior

* `yieldFeeBalance` represents the total amount of assets owed to the fee recipient, to be converted into shares on claim.
* When the recipient calls `claimYieldFeeShares(amount)`, they should:

  1. Receive *exactly* the shares for `amount`.
  2. Have `yieldFeeBalance` reduced by `amount` — **not reset entirely** — so they can claim the rest later.

Example:

* Accrued yield fee = 5 WETH worth of shares.
* Recipient claims 1 WETH worth.
* Expected result:

  * They receive 1 WETH worth of shares.
  * Remaining `yieldFeeBalance` = 4 WETH worth.

### What Actually Happens (Bug)

* In the current logic, *any* call to `claimYieldFeeShares()` **sets `yieldFeeBalance = 0`** after minting shares for the requested amount.
* This means:

  * Recipient receives only the shares they requested.
  * The remainder is wiped out, as if it never existed.

Example (PoC result):

* Accrued yield fee = `4.999...e18`
* Bob calls `claimYieldFeeShares(0.1e18)`
* `yieldFeeBalance` is reset to `0`
* Remaining `4.9e18` worth of fees are lost forever.

### Why This Matters

* Direct financial loss to the fee recipient (and possibly protocol treasury if fees are routed there).
* Creates inconsistencies in accounting — `totalDebt` may temporarily overstate supply until a claim happens, then abruptly reconcile with losses.
* Could be exploited accidentally by misconfigured automation or by recipients claiming small amounts repeatedly.

## Vulnerable Code Reference

```solidity
function claimYieldFeeShares(uint256 shares) external {
    // ...
    _mint(feeRecipient, shares);
    yieldFeeBalance = 0; // ❌ Wipes all remaining fees, even if not claimed
}
```

From PoC logs:

```logs
Accrued yield in contract:  4.99999999999995e18
Bob claims: 1e17 (0.1 WETH worth)
Remaining yieldFeeBalance after claim: 0 (❌ lost ~4.9 WETH worth)
```

## Recommended Mitigation

1. **Only subtract the claimed portion from `yieldFeeBalance`**
   Change:

   ```solidity
   yieldFeeBalance = 0;
   ```

   to:

   ```solidity
   yieldFeeBalance -= claimedAssets;
   ```

   (where `claimedAssets` is the underlying asset equivalent of `shares` claimed)

2. **Add validation to prevent over-claiming**
   Ensure `shares` requested do not exceed `yieldFeeBalance` converted to shares.

3. **Unit tests for partial claim scenarios**
   Add tests for:

   * Multiple sequential claims from a single accrual.
   * Partial claims across different blocks.
   * Edge cases like claiming exactly the full balance.

## Pattern Recognition Notes

This vulnerability is an example of **state variable reset without proportional deduction** — a common pitfall in accounting-heavy DeFi contracts.

### Key Patterns to Watch For

1. **Proportional Reductions**

   * When tracking balances or accrued amounts, partial withdrawals must deduct proportionally, not reset entirely.
   * Always compare "before" and "after" state to confirm arithmetic correctness.

2. **Lifecycle Consistency**

   * For variables representing cumulative accruals (`yieldFeeBalance`), confirm that *all* state transitions maintain logical continuity — accrual → partial claim → remaining accrual → claim again.

3. **Separation of Accounting & Execution**

   * Avoid combining share minting logic with state reset in the same step unless it's guaranteed to be "all-or-nothing."
   * Use dedicated helper functions for partial vs full claims.

4. **Testing Partial Interactions**

   * Most bugs like this are caught by testing "non-happy-path" scenarios, e.g., claiming smaller-than-full amounts.
   * Write tests for sequential actions that interleave accrual and withdrawal.

5. **Guarding Against Automation Edge Cases**

   * If a keeper bot or automated system interacts with a claim function, ensure that calling with "dust amounts" doesn't cause large losses.
