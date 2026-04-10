# Fractional — Missing ERC2981 Interface Declaration Breaks Royalty Detection

* **Severity**: Medium
* **Source**: [Code4rena](https://github.com/code-423n4/2022-07-fractional-findings/issues/544)
* **Affected Contract**: [FERC1155.sol](https://github.com/code-423n4/2022-07-fractional/blob/main/src/FERC1155.sol)
* **Vulnerability Type**: Standards Compliance / Business Logic Flaw / Integration Failure

## Summary

The `FERC1155` contract in Fractional implements **ERC-2981 royalty logic**, but **fails to declare ERC-2981 support via ERC-165** using `supportsInterface()`.

Because most NFT marketplaces rely on **ERC-165 interface detection** to determine whether an NFT supports royalties, they will **not detect royalty support** even though the royalty logic exists.

As a result, **royalties are silently ignored**, and creators **lose royalty revenue** when NFTs are traded.

This is a **standards compliance bug** that leads to **real economic impact**.

## A Better Explanation (With Simplified Example)

### Intended Behavior

1. Fractional NFTs support **ERC-2981 royalties**
2. Marketplaces detect royalty support using:

```solidity
supportsInterface(ERC2981_INTERFACE_ID)
```

3. Marketplace calls:

```solidity
royaltyInfo(tokenId, salePrice)
```

4. Creator receives royalty

### What Actually Happens (Bug)

Fractional **implemented royalty logic** but **did not implement ERC-165 interface detection** for ERC-2981.

So marketplaces do:

```text
Does this NFT support ERC-2981?
```

Contract responds:

```text
false
```

Because `supportsInterface()` doesn't return true for ERC-2981.

Marketplace then **never calls `royaltyInfo()`**, even though it exists.

Result:

* Royalties exist in contract
* Marketplaces ignore them
* Creator receives **0 royalties**

### Why This Matters

* Royalties silently break
* Creators lose revenue
* Marketplace integrations fail
* Hard to detect unless audited

This is especially dangerous because **nothing appears broken**.

Everything works — except creators **don't get paid**.

### Concrete Walkthrough (Alice & Marketplace)

#### Setup

* Alice creates NFT using Fractional
* Sets royalty = 10%

#### Marketplace Trade

Bob buys NFT for **100 ETH**

Marketplace checks:

```solidity
supportsInterface(ERC2981)
```

Returns:

```solidity
false
```

Marketplace decides:

```text
No royalty support
```

Bob buys NFT

Expected:

* Alice gets 10 ETH

Actual:

* Alice gets 0 ETH

### Analogy

This is like:

A shop supports **discount coupons**, but forgets to **put the "Coupons Accepted" sign**.

Customers never use coupons.

Coupons exist — but nobody knows.

## Vulnerable Code Reference

**Missing ERC2981 interface detection**

The contract fails to implement:

```solidity
function supportsInterface(bytes4 interfaceId)
    public view override returns (bool)
{
    return
        super.supportsInterface(interfaceId) ||
        interfaceId == 0x2a55205a; // ERC2981
}
```

Because this is missing:

* ERC2981 exists
* But not discoverable

## Recommended Mitigation

Add ERC2981 interface support:

```solidity
/*//////////////////////////////////////////////////////////////
                        ERC165 LOGIC
//////////////////////////////////////////////////////////////*/

function supportsInterface(bytes4 interfaceId) 
    public view override returns (bool) 
{
    return
        super.supportsInterface(interfaceId) ||
        interfaceId == 0x2a55205a; // ERC165 Interface ID for ERC2981
}
```

## Pattern Recognition Notes

### 1. Missing Interface Declaration

Whenever you see:

* ERC721
* ERC1155
* ERC2981

Always check:

* supportsInterface implemented?
* interface ID registered?

### 2. Silent Integration Failures

This vulnerability is dangerous because:

* No revert
* No failure
* No visible bug

Only integration breaks.

### 3. Standards Compliance Bugs

Common audit pattern:

* Partial standard implementation
* Missing interface detection
* Integration failures

### 4. ERC165 Checklist (Auditor Mental Model)

Whenever auditing NFT contracts:

Check:

* `supportsInterface()` exists
* correct interface IDs added
* inheritance working properly

## Quick Recall (TL;DR)

* **Bug**: ERC2981 implemented but not declared via ERC165
* **Impact**: Marketplaces fail to detect royalties
* **Result**: Creators receive no royalties
* **Fix**: Add ERC2981 interface ID in `supportsInterface()`
