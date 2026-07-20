# Missing Token Existence Check Causes ERC-721 Metadata Specification Violation

* **Severity:** Medium
* **Source:** [Code4rena](https://github.com/code-423n4/2023-10-opendollar-findings/issues/243)
* **Affected Contract:** [Vault721.sol](https://github.com/open-dollar/od-contracts/blob/f4f0246bb26277249c1d5afe6201d4d9096e52e6/src/contracts/proxies/Vault721.sol#L140)
* **Vulnerability Type:** Standards Compliance / Missing Validation / ERC-721 Metadata

## Summary

The `Vault721` contract implements the ERC-721 standard to represent OpenDollar vaults as NFTs. However, its `tokenURI()` implementation does not verify that the requested token actually exists before returning metadata.

According to the ERC-721 Metadata specification, `tokenURI()` **must revert** when queried with a non-existent token ID.

Instead, the contract directly delegates metadata generation to `NFTRenderer`:

```solidity
function tokenURI(uint256 _safeId) public view override returns (string memory uri) {
    uri = nftRenderer.render(_safeId);
}
```

This makes standards compliance depend on the renderer implementation instead of the ERC-721 contract itself.

## Simplified Explanation

Imagine a library where every book has a unique ID.

If someone asks for information about **Book #500**, but the library only contains books **#1-#100**, the librarian should immediately reply:

> "That book does not exist."

Instead, OpenDollar's `Vault721` forwards the request to another employee (the `NFTRenderer`) without first checking whether the book actually exists.

Today, that employee might reject the request.

Tomorrow, they might not.

The problem is that **the librarian was responsible for performing that check in the first place.**

Likewise, the ERC-721 contract—not an external renderer—is responsible for ensuring nonexistent token IDs revert.

## Intended Behavior

When someone calls:

```solidity
tokenURI(tokenId)
```

the contract should:

1. Verify the NFT exists.
2. Revert if the NFT was never minted.
3. Otherwise generate and return its metadata.

This is the behavior required by the ERC-721 Metadata specification.

## Actual Behavior

`Vault721` skips the existence check completely.

Instead it immediately calls:

```solidity
nftRenderer.render(_safeId);
```

Whether the call succeeds or reverts now depends entirely on the renderer implementation.

As a result, the ERC-721 contract itself no longer guarantees compliance with the standard.

## Attack Walkthrough

Although this issue is not an exploit that steals funds, the following scenario demonstrates the incorrect behavior.

### Alice owns Vault NFT #10

The NFT exists.

A marketplace requests:

```solidity
tokenURI(10)
```

The renderer generates metadata successfully.

Everything works as expected.

---

### Someone requests a nonexistent NFT

Suppose NFT #9999 has never been minted.

A marketplace or indexer calls:

```solidity
tokenURI(9999)
```

Instead of first checking whether NFT #9999 exists, `Vault721` immediately forwards the request:

```solidity
nftRenderer.render(9999)
```

Now one of two things happens:

* the renderer happens to revert
* the renderer returns metadata

Either outcome depends entirely on the renderer implementation.

The ERC-721 contract itself never enforced the specification.

## Root Cause

The root cause is that `Vault721` delegates metadata generation without validating token existence first.

```solidity
function tokenURI(uint256 _safeId) public view override returns (string memory uri) {
    uri = nftRenderer.render(_safeId);
}
```

The contract assumes that the renderer will perform the necessary validation.

However, ERC-721 requires the NFT contract itself to reject nonexistent token IDs.

The responsibility for standards compliance should never be delegated to an external helper contract.

## Vulnerable Code Reference

```solidity
function tokenURI(uint256 _safeId)
    public
    view
    override
    returns (string memory uri)
{
    uri = nftRenderer.render(_safeId);
}
```

Missing validation:

```solidity
_requireMinted(_safeId);
```

## Why the Impact Occurs

Many wallets, NFT marketplaces, explorers, and indexers assume ERC-721 contracts behave exactly as defined by the standard.

When a contract violates those guarantees:

* integrations may behave unexpectedly,
* future renderer upgrades may accidentally introduce incorrect behavior,
* standards compliance is no longer guaranteed by the NFT contract itself.

Although this issue does not directly lead to fund loss, it weakens interoperability and makes correctness dependent on another contract.

## Recommended Mitigation

Validate token existence inside `Vault721` before delegating metadata generation.

```solidity
function tokenURI(uint256 _safeId)
    public
    view
    override
    returns (string memory uri)
{
    _requireMinted(_safeId);
    uri = nftRenderer.render(_safeId);
}
```

This guarantees:

* nonexistent token IDs always revert,
* compliance no longer depends on `NFTRenderer`,
* the ERC-721 implementation fully satisfies the specification.

## Pattern Recognition Notes

This finding represents a common auditing pattern:

### 1. Standards Must Be Enforced Locally

If a contract claims to implement a standard (ERC-20, ERC-721, ERC-4626, etc.), it should enforce that standard itself rather than relying on another contract.

---

### 2. Never Delegate Validation

Delegating functionality is acceptable.

Delegating required validation usually is not.

If another contract is upgraded or modified, the invariant may silently disappear.

---

### 3. Helper Contracts Should Not Define Core Behavior

Rendering metadata is an appropriate responsibility for `NFTRenderer`.

Determining whether an NFT exists is not.

That responsibility belongs to the ERC-721 implementation.

## Quick Recall (TL;DR)

* **Bug:** `tokenURI()` does not verify that the requested NFT exists.
* **Root Cause:** Missing `_requireMinted()` check before calling the renderer.
* **Impact:** ERC-721 Metadata specification is violated and standards compliance depends on an external contract.
* **Why It Matters:** Wallets, marketplaces, and other integrations expect ERC-721 contracts to enforce the standard themselves.
* **Fix:** Call `_requireMinted(tokenId)` before generating metadata.
