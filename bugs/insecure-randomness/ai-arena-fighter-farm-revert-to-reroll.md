# NFT Attribute Manipulation via onERC721Received Hook Revert

- **Severity**: Medium
- **Source**: [Code4rena](https://github.com/code-423n4/2024-04-ai-arena-mitigation-findings/issues/50) / [One Bug Per Day](https://www.onebugperday.com/v1/1150)
- **Affected Contract**: [FighterFarm.sol](https://github.com/ArenaX-Labs/2024-02-ai-arena-mitigation/blob/d81beee0df9c5465fe3ae954ce41300a9dd60b7f/src/FighterFarm.sol)
- **Vulnerability Type**: Insecure Randomness / Same-Transaction Trait Assignment

## Original Bug Description

>## Vulnerability details
>
>Mitigation of M-05: NOT mitigated, with error, see comments
>
>[https://github.com/ArenaX-Labs/2024-02-ai-arena-mitigation/blob/d81beee0df9c5465fe3ae954ce41300a9dd60b7f/src/FighterFarm.sol#L574](https://github.com/ArenaX-Labs/2024-02-ai-arena-mitigation/blob/d81beee0df9c5465fe3ae954ce41300a9dd60b7f/src/FighterFarm.sol#L574)
>
>## Mitigated issue(s)
>
>[M-05: Can mint NFT with the desired attributes by reverting transaction](https://github.com/code-423n4/2024-02-ai-arena-findings/issues/376) (Also see duplicated reports [#1017](https://github.com/code-423n4/2024-02-ai-arena-findings/issues/1017), [#578](https://github.com/code-423n4/2024-02-ai-arena-findings/issues/578), [#1991](https://github.com/code-423n4/2024-02-ai-arena-findings/issues/1991))
>
>There were four issues with the randomness of the DNA in FighterFarm.sol:
>
>1. The user can retry the DNA set in `_createNewFighter()` by reverting the `onERC721Received()` hook.
>2. The user can brute force his `msg.sender` which seeds the DNA in `reRoll()` and `claimFighters()`.
>3. The user can wait for an appropriate `fighters.length` which seeds the DNA in `mintFromMergingPool()` and `claimFighters()`.
>4. The user can select the best out of all rerolls, which can be calculated in advance.
>
>(The issue of freely setting DNA in `mintFromMergingPool()` is part of [H-03: Players have complete freedom to customize the fighter NFT when calling redeemMintPass and can redeem fighters of types Dendroid and with rare attributes](https://github.com/code-423n4/2024-02-ai-arena-findings/issues/366).)
>
>Additionally, a mitigation of [#578](https://github.com/code-423n4/2024-02-ai-arena-findings/issues/578), which was duplicated to [M-05 (#376)](https://github.com/code-423n4/2024-02-ai-arena-findings/issues/376), has been attempted. This issue was about the DNA simply being differently generated, due to a likely mistake, but with low impact.
>
>[M-05: Can mint NFT with the desired attributes by reverting transaction](https://github.com/code-423n4/2024-02-ai-arena-findings/issues/376) is only 1. above, and I consider the other issues (including #578) to have been incorrectly or misleadingly duplicated to M-05, and I have to deal with them as separate/new issues/errors.
>I have submitted reports on 2-4 as three new issues, as well as a report on the error from the mitigation of #578. I will make use of this submission as the report of an unmitigated 1., the details on which and recommended mitigation steps follow at the end of this report, after an overview of the review of these issues and the error.
>
>The mitigation review of the part of the scope which is "M-05" thus consists of a total of 5 reports:
>
>1. This report.
>2. "User influence on DNA in `claimFighters()` (unmitigated issue 2. grouped under M-05)"
>3. "User influence on DNA via `fighters.length` in `mintFromMergingPool()` (unmitigated issue 3. grouped under M-05)"
>4. "The user can select the best out of all rerolls (unmitigated issue 4. grouped under M-05)"
>
>       Error. "[M-05/#578] Mitigation error: user influence on DNA via `to`"
>
>## Mitigation re- and overview
>
>1. Nothing has been done to mitigate this.
>
>2. has been mitigated in `reRoll()` by [replacing `msg.sender` with `tokenId`](https://github.com/ArenaX-Labs/2024-02-ai-arena-mitigation/blob/d81beee0df9c5465fe3ae954ce41300a9dd60b7f/src/FighterFarm.sol#L424). But [it has NOT been mitigated in `claimFighters()`](https://github.com/ArenaX-Labs/2024-02-ai-arena-mitigation/blob/d81beee0df9c5465fe3ae954ce41300a9dd60b7f/src/FighterFarm.sol#L241).
>
>3. [Nothing has been done about this in `mintFromMergingPool()`](https://github.com/ArenaX-Labs/2024-02-ai-arena-mitigation/blob/d81beee0df9c5465fe3ae954ce41300a9dd60b7f/src/FighterFarm.sol#L366).
>In `claimFighters()` `fighters.length` has been replaced by `nftsClaimed[msg.sender][0], nftsClaimed[msg.sender][1]`. This is only slightly better. `fighters.length` will be updated by all other users so the attacker would get a steady stream of new values. `nftsClaimed[msg.sender][]` is changed only by the user. However, by transferring the token to another address this value will change, so the user can still influence the DNA.
>
>4. Nothing has been done to mitigate this.
>
>In summary, none of the issues 1-4 have been fully mitigated.
>
>#578 lead to replacing `msg.sender` by `to`. This introduced the error discussed below, also reported separately.
>
>### Mitigation error - introduction of issue 2. in `mintFromMergingPool()`
>
>In the mitigation of #578, having changed the DNA in `mintFromMergingPool()` from `uint256(keccak256(abi.encode(msg.sender, fighters.length)))` to [`uint256(keccak256(abi.encode(to, fighters.length)))`](https://github.com/ArenaX-Labs/2024-02-ai-arena-mitigation/blob/d81beee0df9c5465fe3ae954ce41300a9dd60b7f/src/FighterFarm.sol#L366) has introduced issue 2. for this function as well. `to` is set to `msg.sender` from `MergingPool.claimRewards()`. This is checked to be a winner, which is [set by the admin in `pickWinners()` as the owner of the token](https://github.com/ArenaX-Labs/2024-02-ai-arena-mitigation/blob/d81beee0df9c5465fe3ae954ce41300a9dd60b7f/src/MergingPool.sol#L128). The user can transfer his token to an appropriate address, such that if he is picked as the winner this address generates the desired DNA. A complication for the user is that `fighters.length` might change, so he needs to take the few possible such values into account, and try to limit them by acting quickly and hoping not many fighters will be minted in between. In any case he can significantly influence his DNA.
>
>### Overview of mitigation steps for 1-4 and the error
>
>1. Do not set the DNA in the same transaction as the user receives the fighter.
>
>2. Do not use `msg.sender` to set the DNA.
>
>3. Do not use `fighters.length` to set the DNA.
>
>4. Do not use currently known values to set the DNA.
>
>       Error: Do not use `to` (here equivalent to `msg.sender` in 2.) to set the DNA.
>
>If a third party decentralized VRF is not an option, you can act as your own centralized randomness provider, otherwise equivalent to the typical VRF oracles. The only consideration then is whether users will trust that your server is unbiased and uninfluenceable.
>
>Otherwise it seems the best solution is to set the DNA as `blockhash(block.number - 1)` in a call from the admin, and then in a second call by the user assign the fighter to the user, whether this is by direct minting to the user, by transferring the fighter from an escrow (e.g. FighterFarm itself), or as a finalizing of the reroll.
>
>## M-05: Can mint NFT with the desired attributes by reverting transaction
>
>To clarify the original report:
>
>This issue makes no claim about the direct influence of the DNA calculated in `_createNewFighter()`. The DNA is set in the same transaction as the the fighter is minted. The issue is that the user can let the DNA be calculated and the fighter minted to him, and then, based on what he is to receive, revert the transaction. This is enabled by the `onERC721Received()` hook.
>
>`_createNewFighter()` calls [`_safeMint(to, newId)`](https://github.com/ArenaX-Labs/2024-02-ai-arena-mitigation/blob/d81beee0df9c5465fe3ae954ce41300a9dd60b7f/src/FighterFarm.sol#L574). This calls [`ERC721Utils.checkOnERC721Received(_msgSender(), address(0), to, newId, "")`](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/11dc5e3809ebe07d5405fe524385cbe4f890a08b/contracts/token/ERC721/ERC721.sol#L314), which on a contract executes
>
>```solidity
>try IERC721Receiver(to).onERC721Received(msg.sender, address(0), newId, "") returns (bytes4 retval) {
>    if (retval != IERC721Receiver.onERC721Received.selector) {
>        // Token rejected
>        revert IERC721Errors.ERC721InvalidReceiver(to);
>    }
>```
>
>The user (his contract) may thus implement his `onERC721Received()` to reject unwanted fighters, e.g.
>
>```solidity
>function onERC721Received(
>    address operator,
>    address from,
>    uint256 tokenId,
>    bytes calldata data
>) public returns (bytes4){
>    
>    (,,uint256 weight,,,,) = fighterFarm.getAllFighterInfo(tokenId);
>    require(weight == 95, "I don't want this attribute");
>
>    return bytes4(keccak256(bytes("onERC721Received(address,address,uint256,bytes)")));
>}
>```
>
>## Recommended mitigation steps
>
>There is no way to avoid this issue if the randomness is set in the same transaction as the minting/`onERC721Received()` call. The DNA must be committed first, and then, in a separate transaction, should the user be able to get his fighter. One way is to mint the fighter to the FighterFarm contract, acting as an escrow, and implementing a function for the user to claim it by having it transferred to him.

## Summary

The `_createNewFighter()` function calculates all random Fighter NFT attributes (element, weight, physical traits) and immediately mints the NFT using `_safeMint()` within the same transaction. This allows malicious contract recipients to inspect the freshly computed traits in their `onERC721Received` hook and revert the transaction if the attributes are undesirable, enabling users to "cherry-pick" rare or optimal NFT traits through repeated attempts.

## Real-World Context

- **AI Arena Gaming Economy**: Fighter NFTs have varying rarity levels that directly impact gameplay performance and market value. Rare attributes like specific weights, elements, or physical traits are intended to be randomly distributed to maintain game balance and collectible scarcity.
- **Unfair Advantage**: Users exploiting this vulnerability can guarantee themselves rare Dendroids or optimal stat combinations while honest users receive truly random (often inferior) attributes.
- **Economic Impact**: The exploit undermines the intended rarity distribution, devaluing legitimately obtained rare NFTs and creating an uneven playing field in both gameplay and secondary markets.

## Key Details

- **Pattern**: Same-transaction randomness computation and NFT delivery via `_safeMint()`
- **Root Cause**: The `onERC721Received` hook executes before the mint transaction completes, allowing recipients to inspect and conditionally reject NFTs
- **Risk**: Complete breakdown of intended randomness, allowing systematic generation of rare attributes

## Vulnerable Code Reference

```solidity
function _createNewFighter(
    address to, 
    uint256 dna, 
    string memory modelHash,
    string memory modelType, 
    uint8 fighterType,
    uint8 iconsType,
    uint256[2] memory customAttributes
) private {  
    require(balanceOf(to) < MAX_FIGHTERS_ALLOWED);
    uint256 element; 
    uint256 weight;
    uint256 newDna;
    if (customAttributes[0] == 100) {
        // ❌ Traits computed in same transaction
        (element, weight, newDna) = _createFighterBase(dna, fighterType);
    }
    // ... more attribute computation ...

    // ❌ Physical traits generated immediately
    FighterOps.FighterPhysicalAttributes memory attrs = _aiArenaHelperInstance.createPhysicalAttributes(
        newDna,
        generation[fighterType],
        iconsType,
        dendroidBool
    );

    fighters.push(/* fighter with computed traits */);

    // ❌ Mint in same transaction - allows onERC721Received inspection
    _safeMint(to, newId);
    FighterOps.fighterCreatedEmitter(newId, weight, element, generation[fighterType]);
}
```

## Impact

- **Broken Randomness**: Intended unpredictability is completely compromised
- **Economic Manipulation**: Users can guarantee rare NFTs, devaluing the collection
- **Game Balance Disruption**: Optimal fighters become readily available to exploiters
- **Trust Erosion**: Legitimate players lose confidence in fair distribution

## Mitigation Steps

### 1. Two-Phase Commit-Reveal Pattern

```solidity
// Phase 1: Commit to mint (escrow)
function requestFighter(...) external returns (uint256 requestId) {
    // Mint placeholder NFT to contract (escrow)
    _mint(address(this), requestId);
    // Request external randomness (Chainlink VRF)
    return _requestRandomness();
}

// Phase 2: Reveal attributes and claim
function claimFighter(uint256 requestId) external {
    require(randomnessReady[requestId]);
    // Assign traits using VRF randomness
    _assignTraits(requestId, vrfRandomness[requestId]);
    // Transfer from escrow to user
    _safeTransfer(address(this), msg.sender, requestId);
}
```

### 2. Chainlink VRF Integration

```solidity
function _createNewFighter(...) private {
    uint256 newId = fighters.length;
    // Mint with placeholder traits
    fighters.push(/* empty/placeholder fighter */);
    _mint(to, newId);

    // Request VRF randomness for trait assignment
    uint256 requestId = VRFCoordinatorV2Interface(vrfCoordinator).requestRandomWords(
        keyHash,
        subscriptionId,
        requestConfirmations,
        callbackGasLimit,
        1
    );
    pendingRequests[requestId] = newId;
}

function fulfillRandomWords(uint256 requestId, uint256[] memory randomWords) internal override {
    uint256 tokenId = pendingRequests[requestId];
    // Assign traits using VRF randomness
    _assignFighterTraits(tokenId, randomWords[0]);
}
```

### 3. Blockhash-Based Randomness (Simpler Alternative)

```solidity
mapping(uint256 => uint256) private pendingReveals;

function requestFighter(...) external returns (uint256) {
    uint256 requestId = _nextRequestId++;
    pendingReveals[requestId] = block.number;
    _mint(address(this), requestId); // Mint to escrow
    return requestId;
}

function revealAndClaim(uint256 requestId) external {
    require(block.number > pendingReveals[requestId] + 1);
    uint256 seed = uint256(blockhash(pendingReveals[requestId] + 1));
    _assignTraits(requestId, seed);
    _safeTransfer(address(this), msg.sender, requestId);
}
```

## Pattern Recognition Notes

- Look for NFT mint functions that assign random traits (DNA, attributes, stats) and call `_safeMint()` in the same user-triggered transaction.
- Identify contracts implementing `onERC721Received()` — especially if the function actively checks on-chain attributes (e.g., querying weight, type, or stats) and conditionally reverts based on those values.
- Flag random generation that relies on user-controlled or publicly known inputs, such as `msg.sender`, `to`, `tokenId`, `block.number`, or `fighters.length`.
- Watch for deterministic trait assignment logic where the same input always yields the same trait output (e.g., use of `keccak256(DNA) → attributes`) — enabling repeatable results and replay attacks.
- Monitor minting functions that combine randomness and delivery, especially via `_safeMint` + immediate access to assigned traits before the transaction completes.
- **Red flags**:
  - Random traits derived from predictable inputs like `msg.sender`, `tokenId`, or `block.timestamp`.
  - Inclusion of trait-assignment code right before `_safeMint()` or `_safeTransfer()` to a user-controlled address.
  - No use of VRF or commit-reveal mechanism in randomized NFT drops.
  - Contracts with `onERC721Received()` that inspect NFT content before accepting.

- **Common locations**:
  - NFT-based games, loot box systems, breeding/fusion mechanics.
  - Generative art or collectibles with programmatically assigned attributes.
  - Randomized stat rerolls (e.g., RPG equipment, character stats).
  - "Mystery box" or claim-based reward systems in gaming/metaverse applications.

- **Testing approach**:
  - Deploy a custom receiver contract that inspects traits in `onERC721Received()` and reverts if undesired.
  - Simulate repetitive minting to check if brute-forcing can yield optimal traits over multiple attempts.
  - Precompute DNA or trait generation from known input seeds (e.g., `keccak256(to, id)`) and validate repeatability.
  - Measure if `onERC721Received()` reverts abort the entire transaction and undo state changes, enabling retrials.
