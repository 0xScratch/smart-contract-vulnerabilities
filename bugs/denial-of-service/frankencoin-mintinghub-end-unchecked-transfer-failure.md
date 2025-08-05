# Fragile Challenge Finalization Due to Unchecked Transfer Failures in `end()` Function

* **Severity**: Medium
* **Source**: [Code4rena](https://github.com/code-423n4/2023-04-frankencoin-findings/issues/680) / [One Bug Per Day](https://www.onebugperday.com/v1/40)
* **Affected Contract**: [MintingHub.sol](https://github.com/code-423n4/2023-04-frankencoin/blob/1022cb106919fba963a89205d3b90bf62543f68f/contracts/MintingHub.sol)
* **Vulnerability Type**: External Call Risk / Fund Stuck / DoS via Failing Transfers

## Original Bug Description

>## Lines of code
>
>[https://github.com/code-423n4/2023-04-frankencoin/blob/1022cb106919fba963a89205d3b90bf62543f68f/contracts/MintingHub.sol#L252-L272](https://github.com/code-423n4/2023-04-frankencoin/blob/1022cb106919fba963a89205d3b90bf62543f68f/contracts/MintingHub.sol#L252-L272)
>
>## Vulnerability details
>
>## Impact
>
>The `end()` function to conclude a challenge involves several fund transfer, including the return of challenger's collateral, challenger's reward transfer, the bidder's excess return, position owner's excess fund return. Further, in `Position.sol#notifyChallengeSucceeded()`, underlying collateral withdrawal. If anyone transfer of the above failed and revert, all the other transfer calls will fail. The fund could be stuck temporarily or forever.
>
>## Proof of Concept
>
>The issue here is the external dependence of fund transfer. There could be several scenarios the individual transfer could fail. Such as erc20 0 amount transfer revert, position transfer ownership to `addr(0)`, zchf not enough balance, or other unexpected situations encountered, many other functionality will also be affected.
>
>### 0 amount transfer
>
>Concurrent challenges are allowed in the auction as per the comment in `Position.sol`
>
>>333: // Challenge is larger than the position. This can for example happen if there are multiple concurrent
>>
>>334: // challenges that exceed the collateral balance in size. In this case, we need to redimension the bid and
>>
>>335: // tell the caller that a part of the bid needs to be returned to the bidder.
>
>So in one position, there could be many challenges at the same time, the `challengedAmount` can exceed the total collateral balance. As a result, the last challenger to call `MintingHub.sol#end()` will end up with 0 collateral transfer even if the challenge succeed. If the corresponding collateral erc20 revert on 0 amount transfer, the whole `end()` call will fail, further locking the collateral of the challenger.
>
>Since the total collateral balance is not enough to pay all the `challengeAmount`, eventually the collateral balance of the position will be drained, leaves nothing for those who have not call `end()` yet. When they call `end()`, the challenger's collateral and bidder's excess fund should be returned like below:
>
>```solidity
>File: contracts/MintingHub.sol
>252:     function end(uint256 _challengeNumber, bool postponeCollateralReturn) public {
>
>257:         returnCollateral(challenge, postponeCollateralReturn);
>
>260:         (address owner, uint256 effectiveBid, uint256 volume, uint256 repayment, uint32 reservePPM) = challenge.position.notifyChallengeSucceeded(recipient, challenge.bid, challenge.size);
>261:         if (effectiveBid < challenge.bid) {
>262:             // overbid, return excess amount
>263:             IERC20(zchf).transfer(challenge.bidder, challenge.bid - effectiveBid);
>264:         }
>```
>
>Then in `notifyChallengeSucceeded()`, the amount for collateral withdrawal will be 0.
>
>```solidity
>File: contracts/Position.sol
>329:     function notifyChallengeSucceeded(address _bidder, uint256 _bid, uint256 _size) external onlyHub returns (address, uint256, uint256, uint256, uint32) {
>330:         challengedAmount -= _size;
>331:         uint256 colBal = collateralBalance();
>332:         if (_size > colBal){
>333:             // Challenge is larger than the position. This can for example happen if there are multiple concurrent
>334:             // challenges that exceed the collateral balance in size. In this case, we need to redimension the bid and
>335:             // tell the caller that a part of the bid needs to be returned to the bidder.
>336:             _bid = _divD18(_mulD18(_bid, colBal), _size);
>337:             _size = colBal;
>338:         }
>
>352:         internalWithdrawCollateral(_bidder, _size); // transfer collateral to the bidder and emit update
>
>268:     function internalWithdrawCollateral(address target, uint256 amount) internal returns (uint256) {
>269:         IERC20(collateral).transfer(target, amount);
>```
>
>However some erc20 will revert on 0 amount transfer. Such as (e.g., LEND -> see [https://github.com/d-xo/weird-erc20#revert-on-zero-value-transfers)](https://github.com/d-xo/weird-erc20#revert-on-zero-value-transfers), it reverts for transfer with amount 0. Hence the whole `end()` call will fail, leading to lock of challenger's collateral and bidder's fund. Because the `returnCollateral()` and bid fund return are inside function `end()`, the revert of `notifyChallengeSucceeded()` could prevent the return of both.
>
>Although some collaterals used now may not revert on 0 amount transfer, many erc20 are upgradable, it is unknown if they will change the implementation in the future upgrades.
>
>### position owner being `addr(0)`
>
>If a position owner transferring the ownership to `addr(0)`, and the challenge involves return excess fund to the owner, in line 268 of `MintingHub.sol` the transfer will revert due to the ERC20 requirement.
>
>```solidity
>File: contracts/MintingHub.sol
>252:     function end(uint256 _challengeNumber, bool postponeCollateralReturn) public {
>
>260:         (address owner, uint256 effectiveBid, uint256 volume, uint256 repayment, uint32 reservePPM) = challenge.position.notifyChallengeSucceeded(recipient, challenge.bid, challenge.size);
>261:         if (effectiveBid < challenge.bid) {
>262:             // overbid, return excess amount
>263:             IERC20(zchf).transfer(challenge.bidder, challenge.bid - effectiveBid);
>264:         }
>265:         uint256 reward = (volume * CHALLENGER_REWARD) / 1000_000;
>266:         uint256 fundsNeeded = reward + repayment;
>267:         if (effectiveBid > fundsNeeded){
>268:             zchf.transfer(owner, effectiveBid - fundsNeeded);
>
>File: contracts/ERC20.sol
>151:     function _transfer(address sender, address recipient, uint256 amount) internal virtual {
>152:         require(recipient != address(0));
>```
>
>### not enough balance in zchf
>
>When the protocol incur multiple loss event, the balance could be too low. In such extreme situations, the zchf transfer would also fail. The `end()` would DoS temporarily.
>
>## Tools Used
>
>Manual analysis.
>
>## Recommended Mitigation Steps
>
>A more robust way to handle multiple party fund transfer is to provide alternative ways to refund apart from all in one in `end()`. Just like the `returnPostponedCollateral()` in `MintingHub.sol`.
>
>* In `end()`, record the amount should be transferred to each user, and provide option for them to pull the fund later. If separate the transfer from `end()`, provide the option for the challenger and bidder to pull the fund, then in the case that the other transfer or external calls fail in `end()`, the fund transfer will not dependent on other unexpected factors, and the system could be more robust.
>
>* Check for the total challenge amount, disallow the total challenge to be more than the position collateral .

## Summary

The `end()` function in `MintingHub` finalizes a challenge and performs multiple token transfers — returning collateral, excess bids, and rewards. However, all transfers are executed directly without validating outcomes or isolating them from each other. If any single transfer reverts (e.g., due to a 0-token transfer or a bad recipient address), the entire `end()` function reverts, potentially locking funds for all parties involved.

## A Better Explanation (With Simplified Example)

### Intended Behavior

When a challenge ends:

* The challenger's collateral is returned (if not postponed).
* The `notifyChallengeSucceeded()` function is called to:

  * Transfer the relevant collateral to the bidder.
  * Adjust bid/repayment values if the position can't fully cover the challenge.
* Excess bid is returned to the bidder.
* Reward is paid out to the challenger.
* Surplus funds (if any) are returned to the position owner.

This ensures all parties receive their rightful funds.

### What Actually Happens (Bug)

All fund transfers are tightly coupled into the `end()` function. If *any* of the following transfers fail:

* 0-amount transfer to a token that reverts on zero (`LEND`, others).
* Transfer to `address(0)` (possible if position ownership was accidentally assigned to null).
* ZCHF token running out of balance.

Then the *entire call* reverts, and no one gets anything — even if their part was valid. This can cause:

* Loss of trust in the protocol.
* Temporarily or permanently stuck funds.
* DoS on the challenge finalization.

### Realistic Scenario

Multiple users concurrently challenge the same position. Due to race conditions, by the time the last user calls `end()`, there's no collateral left, leading to:

* A 0-token transfer to the last challenger, which may revert depending on the ERC20 implementation.
* The full `end()` call fails.
* That user can't retrieve funds.
* Others waiting to finalize their own `end()` calls may also fail if they're late.

---

## Vulnerable Code Reference

```solidity
function end(uint256 _challengeNumber, bool postponeCollateralReturn) public {
    ...
    returnCollateral(challenge, postponeCollateralReturn);

    (address owner, uint256 effectiveBid, uint256 volume, uint256 repayment, uint32 reservePPM) =
        challenge.position.notifyChallengeSucceeded(recipient, challenge.bid, challenge.size);

    if (effectiveBid < challenge.bid) {
        IERC20(zchf).transfer(challenge.bidder, challenge.bid - effectiveBid); // [1]
    }

    uint256 reward = (volume * CHALLENGER_REWARD) / 1000_000;
    uint256 fundsNeeded = reward + repayment;

    if (effectiveBid > fundsNeeded) {
        zchf.transfer(owner, effectiveBid - fundsNeeded); // [2]
    }
}
```

```solidity
// Inside Position.sol
function notifyChallengeSucceeded(...) external onlyHub returns (...) {
    ...
    if (_size > colBal) {
        _bid = _divD18(_mulD18(_bid, colBal), _size);
        _size = colBal;
    }

    internalWithdrawCollateral(_bidder, _size); // [3]
}

function internalWithdrawCollateral(address target, uint256 amount) internal returns (uint256) {
    IERC20(collateral).transfer(target, amount); // [4]
}
```

---

## Recommended Mitigation

1. **Separate state updates from fund transfers**

   * Update internal state first.
   * Record owed amounts (e.g., in mappings).
   * Allow users to `pull` their funds later (similar to `returnPostponedCollateral()`).

2. **Never assume transfers will succeed**

   * Check for 0-value before transferring (e.g., skip if amount == 0).
   * Avoid transferring to `address(0)` — add guard clauses.

3. **Add fallback or claim mechanisms**

   * Let users retrieve rewards/funds independently using `claimX()`-type functions.

4. **Prevent total over-challenges**

   * Limit cumulative challenges to the position's actual collateral to reduce underfunded states.

---

## Pattern Recognition Notes

This vulnerability reveals a broader class of issues related to brittle control flow and blind trust in external calls (like token transfers). You can recognize and defend against these by following key design patterns:

---

### 1. **Do Not Trust External Calls to Always Succeed**

* ERC20 tokens do *not* behave consistently.

  * Some revert on 0-amount (`LEND`).
  * Some don't return bool.
  * Some may be upgradable/malicious.
* Always guard transfers:

  ```solidity
  if (amount > 0) {
      require(token.transfer(to, amount), "Transfer failed");
  }
  ```

---

### 2. **Avoid Tight Coupling Between Multiple Transfers**

* Never structure critical logic where multiple unrelated transfers are interdependent.
* Use `pull` over `push` patterns — record the owed amount, let users withdraw.

---

### 3. **Validate Transfer Parameters**

* Ensure recipient is not zero.
* Skip zero-value transfers or handle them gracefully.

---

### 4. **Enforce Resource Caps to Avoid Over-Allocation**

* Don't allow challenge totals to exceed available collateral.
* Prevent over-promising multiple users the same funds.

---

### 5. **Modularize Risky Actions**

* Use independent, retryable functions (e.g., `claimBidExcess()`, `withdrawReward()`).
* Avoid logic that's all-or-nothing, especially in finance logic.

---

### 6. **Test With Faulty Tokens**

* During audits, use ERC20 mocks that:

  * Revert on 0-amount.
  * Return false instead of true.
  * Have unexpected behaviors.
* This improves resilience against real-world ERC20 weirdness.
