# Seaport Merkle Tree Intermediate Hash Bypass in Criteria Resolution

* **Severity**: Medium
* **Source**: [Code4rena](https://github.com/code-423n4/2022-05-opensea-seaport-findings/issues/168)
* **Affected Contract**: [CriteriaResolution.sol](https://github.com/code-423n4/2022-05-opensea-seaport/blob/4140473b1f85d0df602548ad260b1739ddd734a5/contracts/lib/CriteriaResolution.sol)
* **Vulnerability Type**: Business Logic Error / Merkle Proof Verification / Input Validation

## Summary

The Seaport protocol enables "criteria" offers for ERC-721 NFTs, allowing offerers to accept multiple token IDs via a Merkle tree root stored on-chain as `identifierOrCriteria`. Fulfillers submit a token ID and Merkle proof to verify membership. However, the `_verifyProof` function treats the submitted token ID directly as the leaf hash without hashing it first. This allows attackers to submit an intermediate tree hash (e.g., the root itself) as a fake token ID, paired with a trivial proof, bypassing verification. The offerer receives an unintended, low-value NFT, leading to economic losses—especially in trait-based offers where specified IDs represent high-value assets.

## A Better Explanation (With Simplified Example)

### Intended Behavior

1. **Offer Creation**: Offerer (Alice) selects allowed token IDs (e.g., 1 and 2), builds a Merkle tree off-chain with leaves as raw `bytes32(tokenId)`, computes the root, and sets it as `identifierOrCriteria` in the offer.
2. **Fulfillment**: Fulfillers submit a valid token ID (e.g., 1) and proof (intermediate hashes to rebuild the path to root). The contract's `_verifyProof` starts with the token ID as the initial `computedHash`, pairs it with proof elements via sorted keccak256 hashing, and checks if it equals the root.
3. **Trade Completion**: If verified, the specified NFT transfers to the offerer, ensuring only intended assets are accepted.

### What Actually Happens (Bug)

* The Merkle tree leaves are raw `uint256(tokenId)` cast to `bytes32`, but intermediates are `keccak256(64 bytes)` (paired hashes).
* `_verifyProof` initializes `computedHash := leaf` (raw token ID) without hashing, assuming it's a valid leaf.
* Attackers can mint/acquire an NFT with `tokenId` equal to an intermediate hash (e.g., the root, per ERC-721's arbitrary ID allowance) and submit it with an empty/minimal proof.
* Verification passes because starting from an intermediate as "leaf" can trivially reach the root, but the transferred NFT is unrelated to the offerer's criteria.
* No leaf format enforcement (e.g., hashing raw IDs) allows domain collision between leaves (32-byte raw) and intermediates (32-byte hash of 64 bytes).

### Why This Matters

* Undermines criteria's purpose: Efficiently accepting *specific* high-value NFTs (e.g., by traits) while rejecting others.
* Economic exploit: Offerers overpay for junk NFTs; attackers profit by trading arbitrary hash-ID tokens.
* Feasible due to ERC-721 flexibility—IDs aren't sequential counters (EIP-721 warns against assuming patterns; implementations like OpenZeppelin allow arbitrary minting).
* Affects real-world use: Trait-gated offers (e.g., "any rare BAYC") become unsafe without fixes.

### Concrete Walkthrough (Alice & Mallory)

* **Setup**: Alice creates a criteria offer for token IDs 1 or 2 from Collection X (valuable traits). Merkle leaves: `bytes32(1)` and `bytes32(2)`. Root: `keccak256(sorted(leaf1 || leaf2)) = 0xe90b7bceb6e7df5418fb78d8ee546e97c83a08bbccc01a0644d599ccd2a7c2e0`.
* **Mallory attack**: Mallory mints an NFT in Collection X with `tokenId = 0xe90b...` (the root hash—possible via arbitrary mint).
  * Submits fulfillment: `identifier = 0xe90b...`, empty proof (since it's already the root).
  * `_verifyProof` sets `computedHash = 0xe90b...` (leaf), checks `eq(computedHash, root)` → true (no proof needed).
* **Alice's loss**: Trade completes; Alice receives the worthless root-ID NFT (no traits) but pays full price (e.g., 1 ETH). Mallory profits without providing a criteria-matched asset.

> **Analogy**: A VIP list summarized by a master key (root). Guards check your "invite code" (token ID) against the key using a puzzle (proof). But the system accepts the master key itself as a valid invite—letting imposters in with a duplicate key, while real VIPs get sidelined.

## Vulnerable Code Reference

### 1) `_verifyProof` treats raw token ID as leaf without hashing (line ~157 onward)

```solidity
function _verifyProof(
    uint256 leaf,
    uint256 root,
    bytes32[] memory proof
) internal pure returns (bool) {
    bool isValid;
    assembly {
        let computedHash := leaf  // Bug: Raw leaf, no keccak256(leaf) — allows intermediate hashes as fake leaves
        let proofSize := proof.length
        let proofPos := 0
        // If no proof, check if leaf equals root
        if eq(proofSize, 0) {
            isValid := eq(computedHash, root)
        } {
            // Loop over proof elements, hashing pairs (sorted for consistency)
            for { let end := proofSize } lt(proofPos, end) { proofPos := add(proofPos, 1) } {
                let proofElement := mload(add(proof, proofPos))
                if lt(computedHash, proofElement) {
                    computedHash := keccak256(add(computedHash, 0x20), sub(0x20, 0x20))  // Simplified; actual hashes pair
                } {
                    // Hash proofElement || computedHash (or vice versa)
                    mstore(0x00, proofElement)
                    mstore(0x20, computedHash)
                    computedHash := keccak256(0x00, 0x40)
                }
            }
            isValid := eq(computedHash, root)
        }
    }
    return isValid;
}
```

### 2) Criteria resolution in fulfillment calls vulnerable proof without leaf validation

```solidity
// In resolveCriteria (simplified from fulfillment logic)
for (uint256 i = 0; i < criteriaResolvers.length; ) {
    // ...
    uint256 identifier = criteriaResolvers[i].identifier;
    bytes32[] memory proof = criteriaResolvers[i].criteriaProof;
    // Calls _verifyProof(identifier, criteria, proof) — identifier treated as raw leaf
    if (!_verifyProof(identifier, criteria, proof)) {
        revert OfferInvalidCriteria();  // But fake intermediates pass!
    }
    // Proceeds to transfer unintended NFT
    unchecked { ++i; }
}
```

### 3) No enforcement in offer item setup

```solidity
// When setting ItemType.ERC721_WITH_CRITERIA
// identifierOrCriteria = merkleRoot;  // Root stored, but no leaf hashing required off-chain/on-chain
// Fulfillers can supply any uint256 as identifier, including intermediates
```

## Recommended Mitigation

1. **Hash leaves explicitly in verification** (primary fix—prevents collision)

    ```solidity
    function _verifyProof(
        uint256 leaf,
        uint256 root,
        bytes32[] memory proof
    ) internal pure returns (bool) {
        bool isValid;
        assembly {
    -       let computedHash := leaf
    +       computedHash := keccak256(add(leaf, 0x20), 0x20)  // Hash raw leaf: keccak256(bytes32(leaf))
            // Rest unchanged...
        }
        return isValid;
    }
    ```

   * Off-chain: Generate trees with pre-hashed leaves (`keccak256(tokenId)`). Ensures leaves = hash(32 bytes), intermediates = hash(64 bytes)—no overlap.

2. **Add leaf prefix/type byte**: Prefix leaves with `0x00` during tree build and verification to distinguish from intermediates.

    ```solidity
    // Off-chain leaf: keccak256(abi.encodePacked(bytes1(0x00), bytes32(tokenId)))
    // In contract: computedHash := keccak256(abi.encodePacked(bytes1(0x00), leaf))
    ```

3. **Validate token ID bounds/range**: Optionally restrict to expected ID spaces (e.g., < 2^32), but avoid per EIP-721—better as secondary check.

4. **Testing & invariants**: Add fuzz tests for intermediate hashes as identifiers; ensure proofs fail for non-leaf values post-fix. Audit off-chain tree generators.

## Pattern Recognition Notes

* **Domain Separation in Merkle Trees**: Raw data vs. hashes must be distinguished (e.g., via initial hashing) to prevent "fake leaf" attacks—common in proofs without type prefixes.
* **Assumption of Input Format**: Treating user-supplied values (token IDs) as pre-hashed invites collisions; always normalize/validate at verification boundaries.
* **ERC-721 ID Flexibility as Vector**: Standards allowing arbitrary IDs amplify risks in ID-based proofs—design assuming worst-case (random IDs).
* **Off-Chain/On-Chain Sync**: Merkle roots hide details, so on-chain must robustly enforce leaf semantics; mismatches lead to undetectable exploits.
* **Economic Incentives in Offers**: Features for flexibility (criteria) need strong validation to avoid "close enough" attacks eroding value.

### Quick Recall (TL;DR)

* **Bug**: `_verifyProof` uses raw `tokenId` as leaf hash, allowing intermediate tree hashes as fake valid IDs.
* **Impact**: Attackers trade worthless hash-ID NFTs for full-price criteria offers → offerer losses, protocol trust erosion.
* **Fix**: Hash raw leaves in verification (`keccak256(leaf)`); update off-chain tree gen to pre-hash IDs; test for collisions.
