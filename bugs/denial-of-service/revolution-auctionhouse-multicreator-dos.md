# Auction Settlement DoS via Malicious Multi-Creator Setup in Revolution Protocol

* **Severity**: Medium
* **Source**: [Code4rena](https://github.com/code-423n4/2023-12-revolutionprotocol-findings/issues/93) / [One Bug Per Day](https://www.onebugperday.com/v1/835)
* **Affected Contract**: [AuctionHouse.sol](https://github.com/code-423n4/2023-12-revolutionprotocol/blob/d42cc62b873a1b2b44f57310f9d4bbfdd875e8d6/packages/revolution/src/AuctionHouse.sol)
* **Vulnerability Type**: Denial of Service (DoS) / Gas Exhaustion via External Call Abuse

## Original Bug Description

>## Lines of code
>
>[https://github.com/code-423n4/2023-12-revolutionprotocol/blob/d42cc62b873a1b2b44f57310f9d4bbfdd875e8d6/packages/revolution/src/AuctionHouse.sol#L378](https://github.com/code-423n4/2023-12-revolutionprotocol/blob/d42cc62b873a1b2b44f57310f9d4bbfdd875e8d6/packages/revolution/src/AuctionHouse.sol#L378)
>[https://github.com/code-423n4/2023-12-revolutionprotocol/blob/d42cc62b873a1b2b44f57310f9d4bbfdd875e8d6/packages/revolution/src/AuctionHouse.sol#L394](https://github.com/code-423n4/2023-12-revolutionprotocol/blob/d42cc62b873a1b2b44f57310f9d4bbfdd875e8d6/packages/revolution/src/AuctionHouse.sol#L394)
>[https://github.com/code-423n4/2023-12-revolutionprotocol/blob/d42cc62b873a1b2b44f57310f9d4bbfdd875e8d6/packages/revolution/src/AuctionHouse.sol#L400](https://github.com/code-423n4/2023-12-revolutionprotocol/blob/d42cc62b873a1b2b44f57310f9d4bbfdd875e8d6/packages/revolution/src/AuctionHouse.sol#L400)
>[https://github.com/code-423n4/2023-12-revolutionprotocol/blob/d42cc62b873a1b2b44f57310f9d4bbfdd875e8d6/packages/revolution/src/ERC20TokenEmitter.sol#L212](https://github.com/code-423n4/2023-12-revolutionprotocol/blob/d42cc62b873a1b2b44f57310f9d4bbfdd875e8d6/packages/revolution/src/ERC20TokenEmitter.sol#L212)
>[https://github.com/code-423n4/2023-12-revolutionprotocol/blob/d42cc62b873a1b2b44f57310f9d4bbfdd875e8d6/packages/revolution/src/ERC20TokenEmitter.sol#L109](https://github.com/code-423n4/2023-12-revolutionprotocol/blob/d42cc62b873a1b2b44f57310f9d4bbfdd875e8d6/packages/revolution/src/ERC20TokenEmitter.sol#L109)
>[https://github.com/code-423n4/2023-12-revolutionprotocol/blob/d42cc62b873a1b2b44f57310f9d4bbfdd875e8d6/packages/revolution/src/NontransferableERC20Votes.sol#L131](https://github.com/code-423n4/2023-12-revolutionprotocol/blob/d42cc62b873a1b2b44f57310f9d4bbfdd875e8d6/packages/revolution/src/NontransferableERC20Votes.sol#L131)
>[https://github.com/code-423n4/2023-12-revolutionprotocol/blob/d42cc62b873a1b2b44f57310f9d4bbfdd875e8d6/packages/revolution/src/base/erc20/ERC20Upgradeable.sol#L232](https://github.com/code-423n4/2023-12-revolutionprotocol/blob/d42cc62b873a1b2b44f57310f9d4bbfdd875e8d6/packages/revolution/src/base/erc20/ERC20Upgradeable.sol#L232)
>[https://github.com/code-423n4/2023-12-revolutionprotocol/blob/d42cc62b873a1b2b44f57310f9d4bbfdd875e8d6/packages/revolution/src/base/erc20/ERC20VotesUpgradeable.sol#L62](https://github.com/code-423n4/2023-12-revolutionprotocol/blob/d42cc62b873a1b2b44f57310f9d4bbfdd875e8d6/packages/revolution/src/base/erc20/ERC20VotesUpgradeable.sol#L62)
>[https://github.com/code-423n4/2023-12-revolutionprotocol/blob/d42cc62b873a1b2b44f57310f9d4bbfdd875e8d6/packages/revolution/src/base/VotesUpgradeable.sol#L222](https://github.com/code-423n4/2023-12-revolutionprotocol/blob/d42cc62b873a1b2b44f57310f9d4bbfdd875e8d6/packages/revolution/src/base/VotesUpgradeable.sol#L222)
>[https://github.com/code-423n4/2023-12-revolutionprotocol/blob/d42cc62b873a1b2b44f57310f9d4bbfdd875e8d6/packages/revolution/src/base/VotesUpgradeable.sol#L227](https://github.com/code-423n4/2023-12-revolutionprotocol/blob/d42cc62b873a1b2b44f57310f9d4bbfdd875e8d6/packages/revolution/src/base/VotesUpgradeable.sol#L227)
>[https://github.com/code-423n4/2023-12-revolutionprotocol/blob/d42cc62b873a1b2b44f57310f9d4bbfdd875e8d6/packages/revolution/src/base/VotesUpgradeable.sol#L245-L249](https://github.com/code-423n4/2023-12-revolutionprotocol/blob/d42cc62b873a1b2b44f57310f9d4bbfdd875e8d6/packages/revolution/src/base/VotesUpgradeable.sol#L245-L249)
>
>## Vulnerability details
>
>**Note**: *there is another bug in `AuctionHouse::_settleAuction` that causes DoS (querying `verbs.getArtPieceById` too many times), but this submission describes another issue and from now on, **I assume that `verbs.getArtPieceById` is not called in loop in `_settleAuction`.***
>
>## tl;dr
>
>Malicious user can specify creators of art piece maliciously, so that `AuctionHouse::_settleAuction` will use over `20 000 000` gas units, which is close to the block gas limit (`30 000 000`). If block gas limit is ever decreased in the future or the cost of some operations in EVM increase, this will put `AuctionHouse` in the permanent DoS state as `_settleAuction` will attempt to use more gas than the block gas limit.
>
>## Detailed description
>
>User who calls `CultureIndex::createPiece` can specify up to 100 arbitrary creators, who will get paid if the created piece wins voting and is bought in an auction. The function handling that is called `_settleAuction` and looks like this:
>
>```solidity
>    function _settleAuction() internal {
>            [...]
>            else verbs.transferFrom(address(this), _auction.bidder, _auction.verbId);
>
>
>            [...]
>
>                //Transfer creator's share to the creator, for each creator, and build arrays for erc20TokenEmitter.buyToken
>                if (creatorsShare > 0 && entropyRateBps > 0) {
>                    for (uint256 i = 0; i < numCreators; i++) {
>                        [...]
>
>                        //Transfer creator's share to the creator
>                        _safeTransferETHWithFallback(creator.creator, paymentAmount);
>                    }
>                }
>
>
>                //Buy token from ERC20TokenEmitter for all the creators
>                if (creatorsShare > ethPaidToCreators) {
>                    creatorTokensEmitted = erc20TokenEmitter.buyToken{ value: creatorsShare - ethPaidToCreators }(
>                        vrgdaReceivers,
>                        vrgdaSplits,
>                        IERC20TokenEmitter.ProtocolRewardAddresses({
>                            builder: address(0),
>                            purchaseReferral: address(0),
>                            deployer: deployer
>                        })
>                    );
>                }
>            }
>        }
>        [...]
>```
>
>As can be seen, it performs several external transactions. The most important are (I purposely ignore `verbs.getArtPieceById` here as the issue addresses malicious creators only):
>
>* `_safeTransferETHWithFallback(creator.creator, paymentAmount)`
>* `creatorTokensEmitted = erc20TokenEmitter.buyToken`
>
>In order to make `_settleAuction` as costly as possible, attacker can do several things:
>
>* specify as many creators as possible (`100`)
>* consume as much gas as possible in `receive()` of all of these creators
>* make each creator different, so that `sset` operations performed later in the code change `0` to nonzero entries (so that it costs `20000` per word instead of `2900`, for `sreset`)
>
>Since `_safeTransferETHWithFallback` specifies `50 000` gas in a call, each such call will cost `> 50 000 + 9 000 = 59 000` gas (`9 000` for `callvalue` operation). Since the number of creators can be `100`, it will use over `5 900 000` gas by itself (including cost for converting `ETH` to `WETH` and sending it each time).
>
>I will not analyse each memory reads and writes here (I tried to specify the most important places in the "Links to affected code" section of this submission), but another "heavy" function, in terms of gas usage is `erc20TokenEmitter.buyToken` as it will update users' balances (change from `0` to nonzero) and the same will happen for their voting power.
>
>From the tests I have made (see PoC), it follows that attacker can cause `settleCurrentAndCreateNewAuction` (that calls `_settleAuction`) to use over `20 000 000` gas units. Please see the next ("impact") section for further analysis.
>
>## Impact
>
>If the attack is able to cause `_settleAuction` to reach block gas limit, `AuctionHouse` will be DoSed as it won't be possible to settle auctions.
>
>While it's not the case at the moment (attacker can now reach `20 000 000`), I will now demonstrate why it's problematic even right now:
>
>* Ethereum / Base / Optimism block gas limit can change in the future; if it's decreased, it will be possible to DoS `AuctionHouse`
>* EVM operations' cost can change - it is not a hypothetical scenario: for example the cost of `sload` increased from `50` to `2100` (or `100` in case of warm access) from the frontier hardfork (see [frontier](https://ethereum.github.io/execution-specs/autoapi/ethereum/frontier/vm/gas/index.html?gas-sload#gas-sload) and [shanghai](https://ethereum.github.io/execution-specs/autoapi/ethereum/shanghai/vm/gas/index.html?gas-cold-sload#gas-cold-sload))
>
>Since EVM operations' gas cost increase is a real scenario that has happened in the past and the end effect is severe (DoS of `AuctionHouse`), but the attack is:
>
>* not yet possible (it is in fact possible combined with another exploit of specifying a very "heavy" NFT that I will describe in another submission, not to mention the bug about calling `getArtPieceById` that I mentioned earlier), but it will already harm user who calls `_settleAuction` as he will have to pay a lot for the gas
>* in order for the attack to be performed, attacker would have to win the voting first
>
>Hence, I believe that Medium severity is appropriate for this issue.
>
>## Proof of Concept
>
>First of all, please modify the code in the `AuctionHouse::_settleAuction` in the following way:
>
>* replace `ICultureIndex.CreatorBps memory creator = verbs.getArtPieceById(_auction.verbId).creators[i];` with `ICultureIndex.CreatorBps memory creator = artPiece.creators[i];`
>* put `ICultureIndex.ArtPiece memory artPiece = verbs.getArtPieceById(_auction.verbId);` right before `if (creatorsShare > 0 && entropyRateBps > 0) {`
>
>**It is a fix for another bug that is in the `_settleAuction` and since this submission describes another attack vector, we have to fix it first (as otherwise it won't be possible to calculate precisely the impact of the described attack).**
>
>Then, please put the following contract definition in `AuctionSettling.t.sol`:
>
>```solidity
>contract InfiniteLoop
>{
>    receive() external payable
>    {
>        while (true)
>        {
>
>        }
>    }
>}
>```
>
>Then, please put the following code in `AuctionSettling.t.sol` and run the test (`import "forge-std/console.sol";` will be needed as well):
>
>```solidity
>    // this function creates an auction and wait for its finish
>    // if `toDoS` is true, it will create `100` creators and each creator will be a malicious contract that will
>    // run an infinite loop in its `receive()`
>    // if `toDoS` is false, it will create only `1` "honest" creator
>    function _createAndFinishAuction(bool toDoS) internal
>    {
>        uint nCreators = toDoS ? 100 : 1;
>        address[] memory creatorAddresses = new address[](nCreators);
>        uint256[] memory creatorBps = new uint256[](nCreators);
>        uint256 totalBps = 0;
>        address[] memory creators = new address[](nCreators + 1);
>        for (uint i = 0; i < nCreators + 1; i++)
>        {
>            if (toDoS)
>                creators[i] = address(new InfiniteLoop());
>            else
>                creators[i] = address(uint160(0x1234 + i));
>        }
>
>        for (uint256 i = 0; i < nCreators; i++) {
>            creatorAddresses[i] = address(creators[i]);
>            if (i == nCreators - 1) {
>                creatorBps[i] = 10_000 - totalBps;
>            } else {
>                creatorBps[i] = (10_000) / (nCreators - 1);
>            }
>            totalBps += creatorBps[i];
>        }
>
>        uint256 verbId = createArtPieceMultiCreator(
>            "Multi Creator Art",
>            "An art piece with multiple creators",
>            ICultureIndex.MediaType.IMAGE,
>            "ipfs://multi-creator-art",
>            "",
>            "",
>            creatorAddresses,
>            creatorBps
>        );
>
>        vm.startPrank(auction.owner());
>        auction.unpause();
>        vm.stopPrank();
>
>        uint256 bidAmount = auction.reservePrice();
>        vm.deal(address(creators[nCreators]), bidAmount + 1 ether);
>        vm.startPrank(address(creators[nCreators]));
>        auction.createBid{ value: bidAmount }(verbId, address(creators[nCreators]));
>        vm.stopPrank();
>
>        vm.warp(block.timestamp + auction.duration() + 1); // Fast forward time to end the auction
>    }
>
>    function testDOS() public
>    {
>        uint gasConsumption1;
>        uint gasConsumption2;
>        uint gas0;
>        uint gas1;
>
>        vm.startPrank(cultureIndex.owner());
>        cultureIndex._setQuorumVotesBPS(0);
>        vm.stopPrank();
>
>        _createAndFinishAuction(true);
>
>        gas0 = gasleft();
>        auction.settleCurrentAndCreateNewAuction();
>        gas1 = gasleft();
>        // we calculate gas consumption in case of `100` malicious creators
>        gasConsumption1 = gas0 - gas1;
>
>        _createAndFinishAuction(false);
>
>        gas0 = gasleft();
>        auction.settleCurrentAndCreateNewAuction();
>        gas1 = gasleft();
>        // we calculate gas consumption in case of `1` "honest" creator
>        gasConsumption2 = gas0 - gas1;
>
>        console.log("Gas consumption difference =", gasConsumption1 - gasConsumption2);
>    }
>```
>
>It will output a value `~20 500 000` as a gas difference between the use with one "honest" creator and `100` malicious creators when `settleCurrentAndCreateNewAuction` is called.
>
>## Tools Used
>
>VS Code
>
>## Recommended Mitigation Steps
>
>Consider limiting the number of creators that may be specified.
>
>Alternatively, don't distribute `ETH` and `ERC20` tokens to creators in `_settleAuction` - instead let them receive their rewards in a separate function (for example, called `claimRewards`), that one of them will call, independently from `_settleAuction`.
>
>## Assessed type
>
>DoS

## Summary

`AuctionHouse::_settleAuction` distributes ETH and ERC20 rewards to art piece creators when an auction ends.
Since `CultureIndex::createPiece` allows up to **100 arbitrary creators**, a malicious user can register many hostile smart contracts as creators. During settlement:

1. Each creator is paid ETH via `_safeTransferETHWithFallback` (forwards 50k gas).
2. Additional ERC20 token balances and voting power are updated in `ERC20TokenEmitter.buyToken`.

If creators use **gas-heavy `receive()` functions**, and there are many storage writes from 0→nonzero balances, the settlement gas cost exceeds **20M units**. This approaches Ethereum's \~30M block gas limit.

While not yet a permanent DoS, any future block gas limit reduction or gas repricing could **brick the entire auction system**, as auctions would no longer settle.

## A Better Explanation (With Simplified Example)

### Intended Behavior

1. **Art creation**: Up to 100 creators can be credited for a new art piece.
2. **Auction ends**: Settlement pays ETH shares to creators and mints ERC20 voting tokens for them.
3. **New auction**: After settlement completes, a fresh auction is started.

### What Actually Happens (Bug)

* Settlement **loops over all creators**:

  ```solidity
  for (uint256 i = 0; i < numCreators; i++) {
      _safeTransferETHWithFallback(creator.creator, paymentAmount);
  }
  ```

  Each call is an **external transfer** with 50k gas.
* After ETH payments, `erc20TokenEmitter.buyToken` is called, updating balances for **all creators at once**.
* With 100 malicious creators, ETH transfers cost \~5.9M gas, and ERC20 balance updates cost >15M gas.
* Combined: \~20M gas per settlement.

This does not exceed the current block limit but creates a **fragile dependency**: any protocol or chain change could push it over the threshold → **permanent auction DoS**.

### Concrete Walkthrough (Alice & Mallory)

* **Setup**: Auction system live, block gas limit \~30M.
* **Mallory attack**:

  1. Calls `createPiece` with **100 creators**, each one a contract like:

     ```solidity
     contract InfiniteLoop {
         receive() external payable {
             while (true) {}
         }
     }
     ```

  2. Wins an auction with her malicious art piece.
* **Settlement**: `_settleAuction` tries to pay all 100 creators.

  * Each transfer consumes \~50k gas.
  * ERC20 minting then requires \~15M gas.
  * Total \~20M gas → extremely costly, and potentially above the limit in future Ethereum forks.
* **Result**:

  * Whoever triggers settlement pays a massive gas bill.
  * If block gas rules change unfavorably, **auctions cannot be settled at all** → protocol halts.

> **Analogy**: Imagine a cashier who must hand cash to 100 people before closing the register. If all 100 insist on slow, complicated counting, the cashier might not finish before the store lights turn off. Then, no one can close the register → the store can't open again tomorrow.

## Vulnerable Code Reference

### 1) ETH transfers to each creator in settlement loop**

```solidity
for (uint256 i = 0; i < numCreators; i++) {
    _safeTransferETHWithFallback(creator.creator, paymentAmount);
}
```

### 2) Bulk ERC20 updates via `buyToken`**

```solidity
creatorTokensEmitted = erc20TokenEmitter.buyToken{ 
    value: creatorsShare - ethPaidToCreators
}(
    vrgdaReceivers,
    vrgdaSplits,
    IERC20TokenEmitter.ProtocolRewardAddresses({ ... })
);
```

Both lines are **gas-heavy under attacker-controlled creator sets**.

## Recommended Mitigation

1. **Limit creators per piece**: Enforce a smaller max (e.g., 5-10).
2. **Decouple payout from settlement**:

   * Settlement should only record payout entitlements.
   * Creators claim ETH/ERC20 rewards themselves later via `claimRewards()`.
   * This shifts gas costs to individual creators and prevents settlement overload.
3. **Gas-safe design**: Ensure loops over attacker-controlled input cannot grow unbounded.

## Pattern Recognition Notes

* **Gas Amplification via User Input**: Attacker controls loop size (up to 100) and gas consumption of external calls.
* **External Call Abuse**: `_safeTransferETHWithFallback` forwards gas to attacker-supplied contracts.
* **Bulk State Updates**: Updating many ERC20 balances (0→nonzero) is a storage-cost multiplier.
* **Fragile Gas Assumptions**: Protocol correctness depends on current block gas rules, which can change in future forks.
* **Claim-Instead-of-Pay Pattern**: Core settlement logic should avoid distributing funds directly; instead, record balances and let users claim asynchronously.

### Quick Recall (TL;DR)

* **Bug**: Settlement distributes ETH + ERC20 to up to 100 creators in one loop, attacker can make each call maximally costly.
* **Impact**: \~20M gas per settlement; if block gas rules change, protocol can't settle auctions → permanent DoS.
* **Fix**: Cap creators or move payouts into a separate claim function.
