# Replay Attack in Gauge Voting via Delegation Abuse

- **Severity**: High
- **Source**: [Code4rena](https://github.com/code-423n4/2023-08-verwa-findings/issues/396) / [One Bug Per Day](https://www.onebugperday.com/v1/511)
- **Affected Contracts**: [VotingEscrow.sol](https://github.com/code-423n4/2023-08-verwa/blob/main/src/VotingEscrow.sol#L356) / [GaugeController.sol](https://github.com/code-423n4/2023-08-verwa/blob/main/src/GaugeController.sol#L211)
- **Vulnerability Type**: Business-Logic Flaw - Voting Power Replay / Authorization Bypass

## Original Bug Description

>## Lines of code
>
>[https://github.com/code-423n4/2023-08-verwa/blob/main/src/GaugeController.sol#L211](https://github.com/code-423n4/2023-08-verwa/blob/main/src/GaugeController.sol#L211)
>[https://github.com/code-423n4/2023-08-verwa/blob/main/src/VotingEscrow.sol#L356](https://github.com/code-423n4/2023-08-verwa/blob/main/src/VotingEscrow.sol#L356)
>
>## Vulnerability details
>
>## Impact
>
>Delegate mechanism in `VotingEscrow` allows infinite votes in `vote_for_gauge_weights()` in the `GaugeController`. Users can then, for example, claim more tokens in the `LendingLedger` in the market that they inflated the votes on.
>
>## Proof of Concept
>
>`VotingEscrow` has a delegate mechanism which lets a user delegate the voting power to another user.
>
>The `GaugeController` allows voters who locked native in `VotingEscrow` to vote on the weight of a specific gauge.
>
>Due to the fact that users can delegate their voting power in the `VotingEscrow`, they may vote once in a gauge by calling `vote_for_gauge_weights()`, delegate their votes to another address and then call again `vote_for_gauge_weights()` using this other address.
>
>A POC was built in Foundry, add the following test to `GaugeController.t.sol`:
>
>```solidity
>function testDelegateSystemMultipleVoting() public {
>    vm.deal(user1, 100 ether);
>    vm.startPrank(gov);
>    gc.add_gauge(user1);
>    gc.change_gauge_weight(user1, 100);
>    vm.stopPrank();
>
>    vm.deal(user2, 100 ether);
>    vm.startPrank(gov);
>    gc.add_gauge(user2);
>    gc.change_gauge_weight(user2, 100);
>    vm.stopPrank();
>
>    uint256 v = 10 ether;
>
>    vm.startPrank(user1);
>    ve.createLock{value: v}(v);
>    gc.vote_for_gauge_weights(user1, 10_000);
>    vm.stopPrank();
>
>    vm.startPrank(user2);
>    ve.createLock{value: v}(v);
>    gc.vote_for_gauge_weights(user2, 10_000);
>    vm.stopPrank();
>
>    uint256 expectedWeight_ = gc.get_gauge_weight(user1);
>
>    assertEq(gc.gauge_relative_weight(user1, 7 days), 50e16);
>
>    uint256 numDelegatedTimes_ = 20;
>
>    for (uint i_; i_ < numDelegatedTimes_; i_++) {
>        address fakeUserI_ = vm.addr(i_ + 27); // random num
>        vm.deal(fakeUserI_, 1);
>
>        vm.prank(fakeUserI_);
>        ve.createLock{value: 1}(1);
>
>        vm.prank(user1);
>        ve.delegate(fakeUserI_);
>
>        vm.prank(fakeUserI_);
>        gc.vote_for_gauge_weights(user1, 10_000);
>    }
>
>    // assert that the weight is approx numDelegatedTimes_ more than expected
>    assertEq(gc.get_gauge_weight(user1), expectedWeight_*(numDelegatedTimes_ + 1) - numDelegatedTimes_*100);
>
>    // relative weight has been increase by a lot, can be increased even more if wished
>    assertEq(gc.gauge_relative_weight(user1, 7 days), 954545454545454545);
>}
>```
>
>## Tools Used
>
>Vscode, Foundry
>
>## Recommended Mitigation Steps
>
>The vulnerability comes from the fact that the voting power is fetched from the current timestamp, instead of n blocks in the past, allowing users to vote, delegate, vote again and so on. Thus, the voting power should be fetched from n blocks in the past.
>
>Additionaly, note that this alone is not enough, because when the current block reaches n blocks in the future, the votes can be replayed again by having delegated to another user n blocks in the past. The exploit in this scenario would become more difficult, but still possible, such as: vote, delegate, wait n blocks, vote and so on. For this reason, a predefined window by the governance could be scheduled, in which users can vote on the weights of a gauge, n blocks in the past from the scheduled window start.
>
>## Assessed type
>
>Other

## Summary

A malicious actor can reuse the same locked voting power to vote multiple times for a gauge by repeatedly delegating it to fresh addresses. Because the `GaugeController` tracks consumed vote power per address, not per underlying lock or NFT, each new delegatee appears to have unused voting power and is allowed to cast a full vote. This "replay" of vote power inflates a gauge's weight arbitrarily and siphons disproportionate protocol rewards.

## Root Cause

- **Per-Address Accounting**: `vote_user_power` only prevents an address from overspending its power, but delegation moves the same power to a new address with zero record of prior votes.
- **Delegation Mechanism**: `VotingEscrow.delegate()` transfers lock power without marking the original lock as having "spent" its vote.
- **Replay Window**: Voting power is fetched from the current block, so immediate redelegation and revote within the same epoch is permitted.

## A Better Explanation (With Simplified Example)

Imagine Alice locks 10 CANTO and obtains voting power that decays over five years. She allocates 100% of that power to Gauge X once:

1. **Alice votes** Gauge X → Gauge X weight increases by Alice's bias (say, 1 000 000 units).
2. **Alice delegates** her lock to Bob. Bob now holds the same 1 000 000 units of power but has not voted yet.
3. **Bob votes** Gauge X → Gauge X weight increases by another 1 000 000 units.
4. **Repeat**: Alice re-delegates to Charlie, Charlie votes, adding 1 000 000 again. Each fresh delegate adds Alice's entire vote power anew.

Thus, instead of Gauge X having 1 000 000 units (Alice's true power), it ends up with N × 1 000 000 units after N delegate-and-vote cycles, allowing Alice to siphon N times more rewards than entitled.

## Vulnerable Code Reference

```solidity
// GaugeController.sol: per-address vote tracking
mapping(address => uint256) public vote_user_power;

function vote_for_gauge_weights(address _gauge, uint256 _user_weight) external {
    // ...
    require(power_used + new_slope_power - old_slope_power <= 10_000, "Used too much power");
    vote_user_power[msg.sender] = power_used;
    // ... records new slope & bias ...
}

// VotingEscrow.sol: simple delegation
function delegate(address _addr) external nonReentrant {
    LockedBalance memory locked_ = locked[msg.sender];
    // ... validate ...
    _delegate(delegatee, fromLocked, value, LockAction.UNDELEGATE);
    _delegate(_addr, toLocked, value, LockAction.DELEGATE);
}
```

Because `vote_user_power` is keyed by `msg.sender` only, delegated power appears unspent for each new delegatee.

## Pattern Recognition Notes

- When voting power or other fungible resources can be delegated, ensure that consumption tracking follows the resource (e.g., lock/NFT ID), not merely the holder address.
- Flag any voting or allocation functions that deduct or cap usage by `msg.sender` without considering redelegation or multiple holders of the same underlying stake.
- Look for state variables like `vote_user_power[address]` or similar per-address counters that should instead be indexed by lock ID, token ID, or another unique resource identifier.
- Watch for delegation patterns (`delegate(...)`) that transfer influence without invalidating prior authorizations or marking underlying resources as "spent."
- Identify functions that fetch voting power from the current block rather than from a past snapshot—this allows immediate re-delegation and re-use within the same epoch.
- **Red flags**:
  - Voting or reward-allocation loops keyed by user address after delegation.
  - Lack of a cooldown or snapshot window between delegation changes and vote casting.
  - Absence of per-lock or per-token consumption flags when power can be split or moved.
- **Common locations**:
  - Gauge/vote controllers in ve-token systems (e.g., Curve-style vote-escrowed protocols).
  - Reward distributors that rely on per-address balances without accounting for delegation histories.
  - Governance modules permitting delegation or proxy voting.

- **Heuristic checks**:
  - Verify that every vote or allocation call checks and updates the underlying lock or NFT's spent status.
  - Confirm that delegation functions schedule future power removals and immediately revoke or mark spent power at the source.
  - Ensure voting power is computed from a fixed snapshot (n blocks ago) to prevent replay via immediate redelegation.
