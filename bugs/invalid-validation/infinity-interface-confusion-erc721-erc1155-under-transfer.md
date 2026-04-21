# Interface Confusion Leads to Under-Transfer of NFTs in Hybrid ERC721/ERC1155 Collections

* **Severity**: High
* **Source**: [Code4rena](https://github.com/code-423n4/2022-06-infinity-findings/issues/43)
* **Affected Contract**: [InfinityExchange.sol](https://github.com/code-423n4/2022-06-infinity/blob/765376fa238bbccd8b1e2e12897c91098c7e5ac6/contracts/core/InfinityExchange.sol)
* **Vulnerability Type**: Business Logic Flaw / Invalid Interface Assumption / Asset Under-Transfer

## Summary

`InfinityExchange::_transferNFTs` determines whether to transfer NFTs using ERC721 or ERC1155 logic based on `supportsInterface`. However, it **assumes these interfaces are mutually exclusive**, which is not always true.

Some real-world NFT collections (e.g., The Sandbox ASSET tokens) support **both ERC721 and ERC1155 interfaces**. Because the contract checks ERC721 first, it incorrectly routes execution to ERC721 transfer logic—even when the asset behaves like ERC1155.

As a result, when users attempt to purchase multiple quantities of such NFTs, the contract may **transfer fewer tokens than expected**, leading to **silent loss of funds**.

## A Better Explanation (With Simplified Example)

### Intended Behavior

1. User places an order to buy NFTs.
2. Contract determines token standard:

   * If ERC721 → transfer each token individually
   * If ERC1155 → transfer with quantity
3. Correct number of NFTs are transferred.

### What Actually Happens (Bug)

* `_transferNFTs` checks:

  ```solidity
  if (supports ERC721) → use ERC721 logic
  else if (supports ERC1155)
  ```
* Hybrid collections return **true for both** interfaces.
* Because ERC721 is checked first → contract always selects ERC721 path.
* However, the token internally behaves like ERC1155.

### Why This Breaks Things

ERC721 transfer:

```solidity
safeTransferFrom(from, to, tokenId);
```

👉 Assumes **1 NFT per call**

But in hybrid tokens:
👉 This may map to **ERC1155 transfer of quantity = 1**

### Concrete Walkthrough (Alice & Mallory)

* **Scenario**: Alice buys **2 units of the same NFT (same tokenId)** from a hybrid collection.

#### Step-by-step:

1. Contract checks interface:

   * ERC721 → ✅ true → selects ERC721 path
2. Calls `safeTransferFrom` twice (loop of length 2)
3. Internally:

   * Each call behaves like ERC1155 transfer of **1 unit**
   * But due to implementation quirks, only **one effective unit is transferred**

### Final Outcome

| Expected | Actual  |
| -------- | ------- |
| 2 NFTs   | ❌ 1 NFT |

👉 User **overpays** but receives fewer assets
👉 No revert, no warning → **silent failure**

### Why This Matters

* Affects **real-world collections** (not theoretical)
* Causes **direct financial loss**
* Hard to detect during normal execution
* Violates core exchange invariant:

  > "User receives what they paid for"

### Analogy

> You order 2 items from a store, but the system misidentifies the packaging type and ships only 1—without any error. Payment is still for 2.

## Vulnerable Code Reference

**1) Interface-based routing assumes exclusivity**

```solidity
function _transferNFTs(
  address from,
  address to,
  OrderTypes.OrderItem calldata item
) internal {
  if (IERC165(item.collection).supportsInterface(0x80ac58cd)) {
    _transferERC721s(from, to, item);
  } else if (IERC165(item.collection).supportsInterface(0xd9b67a26)) {
    _transferERC1155s(from, to, item);
  }
}
```

**2) ERC721 transfer logic ignores quantity semantics**

```solidity
function _transferERC721s(
  address from,
  address to,
  OrderTypes.OrderItem calldata item
) internal {
  uint256 numTokens = item.tokens.length;
  for (uint256 i = 0; i < numTokens; ) {
    IERC721(item.collection).safeTransferFrom(from, to, item.tokens[i].tokenId);
    unchecked {
      ++i;
    }
  }
}
```

## Recommended Mitigation

### 1. Prefer ERC1155 check over ERC721

```solidity
if (supports ERC1155) {
  _transferERC1155s(...)
} else if (supports ERC721) {
  _transferERC721s(...)
}
```

👉 Ensures quantity-aware transfers when both interfaces exist

### 2. Explicitly validate expected behavior

* Verify whether collection is **strict ERC721** or **hybrid**
* Consider maintaining:

  * allowlist of known standards
  * or explicit order metadata specifying token type

### 3. Sanity checks after transfer

* Validate:

  * expected balance changes
  * or number of tokens received

### 4. Avoid relying solely on `supportsInterface`

👉 Interface support ≠ behavioral guarantee

## Pattern Recognition Notes

* **Interface Ambiguity**: Contracts relying on ERC165 must handle cases where multiple interfaces return true.
* **Incorrect Priority Ordering**: Choosing the wrong branch due to ordering can silently break logic.
* **Assumption of Standard Purity**: Real-world tokens often deviate from strict standards.
* **Silent Value Loss**: Bugs that don't revert but violate economic expectations are especially dangerous.
* **Multi-Standard Token Risk**: Hybrid ERC721/ERC1155 tokens require special handling.

## Quick Recall (TL;DR)

* **Bug**: ERC721 check is prioritized over ERC1155 in hybrid tokens
* **Impact**: Fewer NFTs transferred than paid for
* **Root Cause**: Assuming interface exclusivity
* **Fix**: Check ERC1155 first + don't rely solely on `supportsInterface`
