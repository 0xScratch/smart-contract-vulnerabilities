# Griefing via Forced Share Lock Extension in Phi Protocol

* **Severity**: Medium
* **Source**: [Code4rena](https://github.com/code-423n4/2024-08-phi-findings/issues/52) / [One Bug Per Day](https://www.onebugperday.com/v1/1477)
* **Affected Contract**: [Cred.sol](https://github.com/code-423n4/2024-08-phi/blob/8c0985f7a10b231f916a51af5d506dd6b0c54120/src/Cred.sol)
* **Vulnerability Type**: Griefing / Denial of Service (DoS)

## Original Bug Description

>## Lines of code
>
>[https://github.com/code-423n4/2024-08-phi/blob/8c0985f7a10b231f916a51af5d506dd6b0c54120/src/Cred.sol#L625](https://github.com/code-423n4/2024-08-phi/blob/8c0985f7a10b231f916a51af5d506dd6b0c54120/src/Cred.sol#L625)
>
>## Vulnerability details
>
>## Summary
>
>**Context:**
>
>The Cred.sol contract provides a buyShareCredFor() function which anyone can use to buy shares for another curator.
>
>We also know that once a share is bought, the curator needs to serve a 10 minute share lock period before being able to sell those tokens.
>
>**Issue & Impact:**
>
>The issue is that an attacker can purchase 1 share every 10 minutes on behalf of the curator to DOS them from selling. As we can see on the [bonding share price curve docs](https://docs.google.com/spreadsheets/d/18wHi9Mqo9YU8EUMQuUvuC75Dav6NSY5LOw-iDlkrZEA/edit?pli=1&gid=859106557#gid=859106557), the price to buy 1 share starts from 2.46 USD and barely increases. Each share bought = 10 minutes of delay for the victim user.
>
>Delaying the user from selling is not the only problem though. During these delays, the price of the share can drop, which could ruin a user's trade strategy causing to incur more loss.
>
>## Proof of Concept
>
>Refer to the [price curve](https://docs.google.com/spreadsheets/d/18wHi9Mqo9YU8EUMQuUvuC75Dav6NSY5LOw-iDlkrZEA/edit?pli=1&gid=859106557#gid=859106557) for cost of shares.
>
>1. Let's assume 30 shares have been bought already by users A, B and C (we ignore the 1 share given to the creator). The shares were bought in order i.e. 10 first by A, then B and then C. This means A bought 10 shares for 25 USD, B for 30 USD and C for 35 USD. The price of 1 share currently is 4 USD as per the price curve after all buys.
>
>2. Let's say A and C hold the shares for a bit longer while B is looking to sell them as soon as possible to profit from C's buy that increased the price.
>
>3. Now let's say the attacker calls function buyShareCredFor() with user B as the curator. This would be done close to the end of the 10 minute share lock period of user B.
>
>    ```solidity
>    File: Cred.sol
>    186:     function buyShareCredFor(uint256 credId_, uint256 amount_, address curator_, uint256 maxPrice_) public payable {
>    187:         if (curator_== address(0)) revert InvalidAddressZero();
>    188:         _handleTrade(credId_, amount_, true, curator_, maxPrice_);
>    189:     }
>    ```
>
>4. When the attacker's call goes through to the internal function _handleTrade(), the lastTradeTimestamp is updated for the user B. This means B has to serve the share lock period all over again for 10 minutes. Note that this would provide user B with 1 more share (which costs the attacker 4 USD).
>
>    ```solidity
>    File: Cred.sol
>    642:         if (isBuy) {
>    643:             cred.currentSupply += amount_;
>    644:             ...
>    649:             lastTradeTimestamp[credId_][curator_] = block.timestamp;
>    ```
>
>5. User B tries to sell but isn't able to due to the share lock period check in _handleTrade().
>
>    ```solidity
>    File: Cred.sol
>    629:             if (block.timestamp <= lastTradeTimestamp[credId_][curator_] + SHARE_LOCK_PERIOD) {
>    630:                 revert ShareLockPeriodNotPassed(
>    631:                     block.timestamp, lastTradeTimestamp[credId_][curator_] + SHARE_LOCK_PERIOD
>    632:                 );
>    633:             }
>    ```
>
>6. Users C and A sell their shares of the credId during this period for 36 USD and 31 USD respectively. This brings the price per share down to 2.57 USD, which for user B would mean 25.7 USD. This means user B lost 4.3 USD approximately from the initial buy.
>
>7. Note that user C could also have been the attacker where the net loss of the attacker i.e. C would result to 4 USD - (final sell amount - initial purchase amount) = 4 USD - (36 USD - 35 USD) = 3 USD.
>
>8. Overall in addition to grief DOSing a user from selling, other users are able to alter another user's strategy by preventing them from selling.
>
>## Tools Used
>
>Manual Review
>
>## Recommended Mitigation Steps
>
>Consider removing the buyShareCredFor() function since it does more harm than good.
>
>## Assessed type
>
>Timing

## Overview

The `Cred.sol` contract allows any user to buy shares on behalf of another user via the `buyShareCredFor()` function. However, doing so **resets the recipient's share lock timer**, which prevents them from selling shares for another 10 minutes.

This can be abused to **grief a curator** by repeatedly gifting them a share every 10 minutes and **extending their lock indefinitely**, thus blocking them from executing a profitable trade or withdrawing in time. This is especially harmful in volatile markets or during emergencies, and constitutes an unintentional **external control over someone else's funds**.

## A Better Explanation (With Simplified Example)

### Intended Behavior

* When a user buys a share, they are subject to a 10-minute lock (`SHARE_LOCK_PERIOD`) before they can sell.
* This is designed to prevent instant buy-sell loops and promote more stable curation.

### What Actually Happens (Bug)

* Anyone can call `buyShareCredFor()` and gift a share to a curator.
* This **resets the curator's `lastTradeTimestamp`**, forcing them to wait another 10 minutes before selling _any_ of their shares—even ones they already owned.
* The cost per share is low and increases slowly, so the griefing can be done **cheaply** and **repeatedly**.

### Why This Matters

* An attacker can strategically delay other users from exiting or profiting.
* During this forced wait, market conditions may change unfavorably—e.g., share price drops.
* In edge cases (e.g., a protocol exploit), this delay can prevent emergency withdrawals, compounding losses.
* Even though the attacker "gifts" a share, this share may be worth little, and the harm is disproportionately higher.

## Vulnerable Code Reference

```solidity
function buyShareCredFor(uint256 credId_, uint256 amount_, address curator_, uint256 maxPrice_) public payable {
    if (curator_ == address(0)) revert InvalidAddressZero();
    _handleTrade(credId_, amount_, true, curator_, maxPrice_);
}

...

if (isBuy) {
    cred.currentSupply += amount_;
    ...
    lastTradeTimestamp[credId_][curator_] = block.timestamp;  // ← Resets lock on gifted share
}

...

if (block.timestamp <= lastTradeTimestamp[credId_][curator_] + SHARE_LOCK_PERIOD) {
    revert ShareLockPeriodNotPassed(...);
}
```

## Recommended Mitigation

1. **Restrict Lock Reset to Self-Trades Only**

   * Only update `lastTradeTimestamp` when the curator is the buyer, not when receiving shares from others.
   * For example:

     ```solidity
     if (isBuy && msg.sender == curator_) {
         lastTradeTimestamp[credId_][curator_] = block.timestamp;
     }
     ```

2. **Disable `buyShareCredFor()` Entirely**

   * If external gifting has no strong UX or utility benefit, consider removing this function altogether.

3. **Add Tests and Checks for Forced-Lock Scenarios**

   * Include unit tests where one user repeatedly extends another's lock period.
   * Verify that time-based locks can't be manipulated by third parties.

## Pattern Recognition Notes

This vulnerability demonstrates a classic form of **griefing** via unintended third-party control of a user's locked funds. It is especially important in protocols involving **time-locked assets**, **price curves**, and **shared marketplaces**.

Key takeaways:

1. **Never Let Third Parties Reset Critical Timers**

   * Time locks (e.g., withdrawal delay, cooldowns) must be owned and controlled solely by the user triggering the action.
   * Allowing external resets enables griefing and denial-of-service.

2. **Beware of "Helpful" Functions That Can Be Weaponized**

   * Functions like `buyFor`, `mintFor`, `transferTo`, etc., may seem useful for UX but often introduce attack vectors.
   * Analyze whether they can be used to interrupt or alter another user's flow.

3. **Monitor Lock-Based Systems for External Inputs**

   * If your protocol uses locking periods or cooldowns, **check who can influence the reset**.
   * Any such influence should be limited to the asset owner.

4. **Evaluate the Cost vs Impact of Griefing Attacks**

   * Even if the attack costs the attacker some capital (e.g., share gifting), it may be worth it if:

     * The loss for the victim is larger.
     * The attacker prevents the victim from front-running or exiting.
     * The attacker benefits elsewhere (e.g., price manipulation).

5. **Use Access Control or Conditional Logic on State Updates**

   * Instead of globally setting lock values, use conditions like `msg.sender == user` or track `lock initiated by` to filter abuse.

6. **Simulate User Interactions from Multiple Angles**

   * In audits and test environments, simulate grief attempts from one user to another—not just isolated actions.
   * Look for ways attackers can interfere with the timing, pricing, or usability of others' actions.
