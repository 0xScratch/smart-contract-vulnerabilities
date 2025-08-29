# Double Royalty Payout Due to Faulty Split Logic in NextGen Minter Contract

* **Severity**: Medium
* **Source**: [Code4rena](https://github.com/code-423n4/2023-10-nextgen-findings/issues/1686) / [One Bug Per Day](https://www.onebugperday.com/v1/600)
* **Affected Contract**: [MinterContract.sol](https://github.com/code-423n4/2023-10-nextgen/blob/main/smart-contracts/MinterContract.sol)
* **Vulnerability Type**: Input Invalidation / Royalty Accounting Error

## Original Bug Description

>## Lines of code
>
>[https://github.com/code-423n4/2023-10-nextgen/blob/main/smart-contracts/MinterContract.sol#L369-L376](https://github.com/code-423n4/2023-10-nextgen/blob/main/smart-contracts/MinterContract.sol#L369-L376)
>[https://github.com/code-423n4/2023-10-nextgen/blob/main/smart-contracts/MinterContract.sol#L380-L382](https://github.com/code-423n4/2023-10-nextgen/blob/main/smart-contracts/MinterContract.sol#L380-L382)
>[https://github.com/code-423n4/2023-10-nextgen/blob/main/smart-contracts/MinterContract.sol#L415-L418](https://github.com/code-423n4/2023-10-nextgen/blob/main/smart-contracts/MinterContract.sol#L415-L418)
>
>## Vulnerability details
>
>## Description
>
>The royalty allocation within the protocol works like this:
>
>First an admin sets the split between team and artist. Here it is validated that the artist and team allocations together fill up 100%:
>
>[`MinterContract::setPrimaryAndSecondarySplits`](https://github.com/code-423n4/2023-10-nextgen/blob/main/smart-contracts/MinterContract.sol#L369-L376):
>
>```solidity
>File: smart-contracts/MinterContract.sol
>
>369:    function setPrimaryAndSecondarySplits(uint256 _collectionID, uint256 _artistPrSplit, uint256 _teamPrSplit, uint256 _artistSecSplit, uint256 _teamSecSplit) public FunctionAdminRequired(this.setPrimaryAndSecondarySplits.selector) {
>370:        require(_artistPrSplit + _teamPrSplit == 100, "splits need to be 100%");
>371:        require(_artistSecSplit + _teamSecSplit == 100, "splits need to be 100%");
>372:        collectionRoyaltiesPrimarySplits[_collectionID].artistPercentage = _artistPrSplit;
>373:        collectionRoyaltiesPrimarySplits[_collectionID].teamPercentage = _teamPrSplit;
>374:        collectionRoyaltiesSecondarySplits[_collectionID].artistPercentage = _artistSecSplit;
>375:        collectionRoyaltiesSecondarySplits[_collectionID].teamPercentage = _teamSecSplit;
>376:    }
>```
>
>The artist can then propose a split between artist addresses, where it is validated that the different artist addresses together add up to the total artist allocation:
>
>[`MinterContract::proposePrimaryAddressesAndPercentages`](https://github.com/code-423n4/2023-10-nextgen/blob/main/smart-contracts/MinterContract.sol#L380-L382):
>
>```solidity
>File: smart-contracts/MinterContract.sol
>
>380:    function proposePrimaryAddressesAndPercentages(uint256 _collectionID, address _primaryAdd1, address _primaryAdd2, address _primaryAdd3, uint256 _add1Percentage, uint256 _add2Percentage, uint256 _add3Percentage) public ArtistOrAdminRequired(_collectionID, this.proposePrimaryAddressesAndPercentages.selector) {
>381:        require (collectionArtistPrimaryAddresses[_collectionID].status == false, "Already approved");
>382:        require (_add1Percentage + _add2Percentage + _add3Percentage == collectionRoyaltiesPrimarySplits[_collectionID].artistPercentage, "Check %");
>```
>
>Then, also when paid out, there is a validation that the split between team and artist allocation adds up:
>
>[`MinterContract::payArtist`](https://github.com/code-423n4/2023-10-nextgen/blob/main/smart-contracts/MinterContract.sol#L415-L418):
>
>```solidity
>File: smart-contracts/MinterContract.sol
>
>415:    function payArtist(uint256 _collectionID, address _team1, address _team2, uint256 _teamperc1, uint256 _teamperc2) public FunctionAdminRequired(this.payArtist.selector) {
>416:        require(collectionArtistPrimaryAddresses[_collectionID].status == true, "Accept Royalties");
>417:        require(collectionTotalAmount[_collectionID] > 0, "Collection Balance must be grater than 0");
>418:        require(collectionRoyaltiesPrimarySplits[_collectionID].artistPercentage + _teamperc1 + _teamperc2 == 100, "Change percentages");
>```
>
>The issue is that, here it compares to the artist allocation from artist/team allocations. When the actual amount paid out is calculated, it uses the proposed (and accepted) artist address percentages:
>
>[`MinterContract::payArtist`](https://github.com/code-423n4/2023-10-nextgen/blob/main/smart-contracts/MinterContract.sol#L429-L431):
>
>```solidity
>File: smart-contracts/MinterContract.sol
>
>429:        artistRoyalties1 = royalties * collectionArtistPrimaryAddresses[colId].add1Percentage / 100;
>430:        artistRoyalties2 = royalties * collectionArtistPrimaryAddresses[colId].add2Percentage / 100;
>431:        artistRoyalties3 = royalties * collectionArtistPrimaryAddresses[colId].add3Percentage / 100;
>```
>
>Hence, there is a possibility that up to double the intended amount can be paid out, imagine this scenario:
>
>1. An admin sets artist/team allocation to 100% artist, 0% team.
>2. Artist proposes a split between addresses (for the total 100%), which is accepted
>3. Admin then changes the allocation to 0% artist, 100% team.
>
>They then call `payArtist` with the 100% team allocation. This will pass the check as `artistPercentage` will be `0`. However, the artist payouts will use the already proposed artist address allocations. Hence double the amount will be paid out.
>
>## Impact
>
>An admin can maliciously or by mistake manipulate the percentage allocation for a collections royalty to pay out up to double the amount of royalty. This could make it impossible to payout royalty later as this steals royalty from other collections.
>
>## Proof of Concept
>
>PoC using foundry. Save file in `hardhat/test/MinterContractTest.t.sol`, also needs `forge-std` in `hardhat/lib`:
>
>```solidity
>// SPDX-License-Identifier: GPL-3.0
>pragma solidity 0.8.19;
>
>import {Test} from "../lib/forge-std/src/Test.sol";
>import {NextGenMinterContract} from "../smart-contracts/MinterContract.sol";
>import {NextGenAdmins} from "../smart-contracts/NextGenAdmins.sol";
>import {INextGenCore} from "../smart-contracts/INextGenCore.sol";
>
>contract MinterContractTest is Test {
>  
>  uint256 constant colId = 1;
>  
>  address team = makeAddr('team');
>  address artist = makeAddr('artist');
>
>  address core  = makeAddr('core');
>  address delegate = makeAddr('delegate');
>
>  NextGenAdmins admin = new NextGenAdmins();
>  NextGenMinterContract minter;
>
>  function setUp() public {
>    minter = new NextGenMinterContract(core, delegate, address(admin));
>
>    vm.mockCall(core, abi.encodeWithSelector(INextGenCore.retrievewereDataAdded.selector), abi.encode(true));
>    vm.mockCall(core, abi.encodeWithSelector(INextGenCore.viewMaxAllowance.selector), abi.encode(1));
>    vm.mockCall(core, abi.encodeWithSelector(INextGenCore.retrieveTokensMintedPublicPerAddress.selector), abi.encode(0));
>    vm.mockCall(core, abi.encodeWithSelector(INextGenCore.viewTokensIndexMin.selector), abi.encode(1));
>    vm.mockCall(core, abi.encodeWithSelector(INextGenCore.viewCirSupply.selector), abi.encode(0));
>    vm.mockCall(core, abi.encodeWithSelector(INextGenCore.viewTokensIndexMax.selector), abi.encode(1_000));
>    vm.mockCall(core, abi.encodeWithSelector(INextGenCore.retrieveArtistAddress.selector), abi.encode(artist));
>    vm.mockCall(core, abi.encodeWithSelector(INextGenCore.mint.selector), new bytes(0));
>
>    minter.setCollectionCosts(colId, 1 ether, 1 ether, 0, 100, 0, address(0));
>    minter.setCollectionPhases(colId, 0, 0, 1, 101, bytes32(0));
>
>    vm.warp(1); // jump one sec to enter public phase
>  }
>
>  function testFaultySplitResultsInDoubleRoyalty() public {
>    // needs to hold one extra eth for payout
>    vm.deal(address(minter),1 ether);
>
>    // mint to increase collectionTotalAmount
>    minter.mint{value: 1 ether}(colId, 1, 1, '', address(this), new bytes32[](0), address(0), 0);
>    assertEq(2 ether, address(minter).balance);
>
>    // begin with setting artist split to 100%, team 0%
>    minter.setPrimaryAndSecondarySplits(colId,
>      100, 0, // primary
>      100, 0  // secondary (not used)
>    );
>
>    // set the actual artist split
>    minter.proposePrimaryAddressesAndPercentages(colId,
>      artist, address(0), address(0),
>      100, 0, 0
>    );
>    minter.acceptAddressesAndPercentages(colId, true, true);
>
>    // set 100% to team, 0% to artist without changing artist address allocation
>    minter.setPrimaryAndSecondarySplits(colId,
>      0, 100, // primary
>      0, 100  // secondary (not used)
>    );
>
>    // when artist is paid, 2x the amount is paid out
>    minter.payArtist(colId, team, address(0), 100, 0);
>
>    // team gets 1 eth
>    assertEq(1 ether, team.balance);
>    // artist gets 1 eth
>    assertEq(1 ether, artist.balance);
>  }
>}
>```
>
>## Tools Used
>
>Manual audit
>
>## Recommended Mitigation Steps
>
>Consider verifying that the artist allocations add up to the artist percentage when calling payArtist.
>
>## Assessed type
>
>Invalid Validation

## Summary

The **NextGen Minter Contract** incorrectly applies royalty splits when both the artist and team allocations are updated inconsistently.

Specifically, when the **split percentages** are changed without updating the **associated payout addresses**, the payout calculation uses both the old and new splits. This results in the **same minted revenue being paid out twice** — once to the originally set artist and once to the newly assigned team — effectively **double-paying royalties**.

This allows funds in the contract to be drained unintentionally or even maliciously if misconfigured.

## A Better Explanation (With Simplified Example)

### Intended Behavior

The system separates payouts into two parts:

* **Primary splits** → How mint revenue is split between artist and team.
* **Secondary splits** → How resale royalties are split (not relevant here).

The correct flow should be:

1. Mint revenue is collected into the contract.
2. Splits are set such that, for example:

   * Artist = 100%
   * Team = 0%
3. When payout occurs, **only the defined percentages** should be applied to the **correct addresses**.

So if there is `2 ETH` inside the contract, and the artist is set to `100%`, only the artist should receive the full `2 ETH`.

### What Actually Happens (Bug)

Because of a logic error:

1. The artist is initially configured to receive **100%**.

   * Artist address is properly stored.
2. The splits are later updated to make the **team = 100%** without resetting the artist address.

   * This leaves the artist address still marked as a payout target.
3. When `payArtist()` is called, the contract attempts to pay **both the team and the artist** because both allocations are still valid in different parts of state.

💥 **Result:** Both the team and artist are paid **as if each had 100% allocation**, leading to **double payouts**.

### Simplified Example

Let's say the minter contract has **2 ETH** from previous mints.

* **Step 1:** Initially configure splits as:

  * Artist = 100%
  * Team = 0%
* **Step 2:** Later change splits to:

  * Artist = 0%
  * Team = 100%
    *(but forget to unset the stored artist address)*
* **Step 3:** Call `payArtist()`

🔹 Expected Outcome:

* Team should get **2 ETH**
* Artist should get **0 ETH**

🔻 Actual Outcome:

* Team gets **1 ETH**
* Artist also gets **1 ETH**
* Total paid out = **2 ETH**, but the accounting assumes it should only be **1 ETH**.

This creates a **double royalty bug**, draining contract funds.

## Vulnerable Code Reference

Key functions involved:

```solidity
function setPrimaryAndSecondarySplits(...) external {
    // Updates split percentages but not necessarily addresses
}

function proposePrimaryAddressesAndPercentages(...) external {
    // Sets actual artist address, but may not be updated later
}

function acceptAddressesAndPercentages(...) external {
    // Finalizes allocation
}

function payArtist(...) external {
    // Pays out both parties based on mismatched state
}
```

And from the PoC:

```solidity
// set 100% artist, then change to 100% team without unsetting artist
minter.setPrimaryAndSecondarySplits(colId, 0, 100, 0, 100);

// when payArtist() is called, BOTH artist and team are paid
minter.payArtist(colId, team, address(0), 100, 0);
```

## Recommended Mitigation

1. **Ensure address and percentage allocations are always updated together**

   * When changing split percentages, force the corresponding payout addresses to be explicitly redefined.

2. **Enforce mutual exclusivity of splits**

   * If a new payout receiver is assigned 100%, ensure old receivers cannot still claim.

3. **Add validation logic**

   * Before paying out, validate that **total split percentages = 100%** and no stale addresses remain in state.

4. **Testing & Simulations**

   * Add unit tests simulating multiple rounds of updating splits and addresses to ensure payouts are consistent.

## Pattern Recognition Notes

This bug falls under **state inconsistency between address mappings and split percentages**. Similar issues often occur in:

1. **Royalty or fee distribution contracts**

   * Multiple roles (artist, team, platform) must be carefully synchronized between addresses and percentages.

2. **Upgradeable allocation systems**

   * Bugs appear when **percentages are changed but addresses are not updated** (or vice versa).

3. **Business logic relying on both historical and current state**

   * If old state values are not invalidated, they may still be used during payout.

🔑 Key lessons:

* Always **atomically update related state variables** (address ↔ percentage).
* Add **invariants**: total payout percentages must equal 100%, and addresses with 0% must not be stored as active.
* Write tests for "partial update" edge cases where only one of (address, percentage) is updated.
