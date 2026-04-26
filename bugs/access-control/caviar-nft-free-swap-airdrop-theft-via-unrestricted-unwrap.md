# Unrestricted NFT Unwrap Enables Free Swaps & Airdrop Theft

* **Severity**: Medium
* **Source**: [Code4rena](https://github.com/code-423n4/2022-12-caviar-findings/issues/367)
* **Affected Contract**: [Pair.sol](https://github.com/code-423n4/2022-12-caviar/blob/0212f9dc3b6a418803dbfacda0e340e059b8aae2/src/Pair.sol)
* **Vulnerability Type**: Access Control / Economic Exploit / NFT Ownership Abuse

## Summary

The `Pair` contract allows users to **wrap NFTs into fungible fractional tokens** and later **unwrap NFTs by burning those tokens**. However, the protocol **does not enforce any linkage between deposited NFT IDs and withdrawn NFT IDs**.

This enables:

* **Free NFT swaps** (no fee, no slippage)
* **Temporary ownership of arbitrary NFTs**
* Exploitation of **external NFT privileges** (airdrops, staking, rewards)

An attacker can repeatedly unwrap NFTs held by the contract, extract benefits (e.g., claim airdrops), and rewrap them—resulting in **theft of rewards belonging to all liquidity providers**.

## A Better Explanation (With Simplified Example)

### Intended Behavior

1. **Wrap**:

   * User deposits NFT(s)
   * Receives `1e18` tokens per NFT

2. **Unwrap**:

   * User burns tokens
   * Receives NFT(s)

👉 Implicit assumption:

> "All NFTs in the pool are interchangeable"

### What Actually Happens (Bug)

* `wrap()` only checks if NFT IDs are **whitelisted (via Merkle proof)**
* `unwrap()` allows withdrawing **ANY NFT ID in the contract**

❗ There is **no check**:

```solidity
// Missing conceptually:
require(depositorOf[tokenId] == msg.sender);
```

### Why This Matters

This creates two major problems:

## 🚨 Issue 1: Free NFT Swaps

### Example

* Alice deposits NFT #1 → gets 1 token
* She unwraps NFT #999 → burns 1 token

✅ Swap completed
❌ No fee paid

👉 This breaks:

* AMM economics
* Liquidity provider incentives

## 💣 Issue 2: Airdrop Theft (Critical)

### Setup

* Contract holds **100 NFTs from users**
* NFT collection distributes rewards via:

```solidity
nft.getAirDrop(tokenId);
```

Only **current owner** can claim.

### Attack Walkthrough (Mallory)

#### Step 1: Entry

* Mallory owns 1 NFT (#A)

#### Step 2: Wrap

* Deposits #A → gets 1 token

#### Step 3: Loop through pool NFTs

For each NFT `i` in contract:

1. **Unwrap NFT `i`**

   * Burn 1 token
   * Mallory becomes temporary owner

2. **Claim airdrop**

   * Calls: `nft.getAirDrop(i)`
   * Receives rewards

3. **Wrap NFT `i` back**

   * Deposits again
   * Gets token back

🔁 Repeat for all NFTs

#### Step 4: Exit

* Unwrap original NFT #A

### Final Result

* Mallory still has same token value
* BUT also stole:
  👉 **All airdrops from all NFTs in the pool**

### Why This Works

* Ownership is **temporarily transferable**
* Protocol allows:

  > "Withdraw any NFT → return later → no restriction"

### Additional Exploitable Cases

This issue extends beyond airdrops:

* **Staking rewards theft** (e.g., BAYC / MAYC)
* Governance manipulation
* Access-controlled perks
* Any "owner-only" NFT functionality

> **Analogy**:
> A vault holds 100 valuable items. You deposit 1 item and get a pass. The system lets you temporarily take **any item**, use its benefits, and return it—without restriction.

## Vulnerable Code Reference

### 1) `wrap()` - No ownership tracking

```solidity
fractionalTokenAmount = tokenIds.length * ONE;
_mint(msg.sender, fractionalTokenAmount);

// Transfers NFT into contract
ERC721(nft).safeTransferFrom(msg.sender, address(this), tokenIds[i]);
```

* No mapping:

  * `tokenId → depositor`
* No uniqueness enforcement

### 2) `unwrap()` - Arbitrary NFT withdrawal

```solidity
fractionalTokenAmount = tokenIds.length * ONE;
_burn(msg.sender, fractionalTokenAmount);

// Transfers ANY NFT from contract
ERC721(nft).safeTransferFrom(address(this), msg.sender, tokenIds[i]);
```

❗ Critical flaw:

* Caller chooses **any tokenId**
* No validation of origin or ownership

## Recommended Mitigation

### 1. Enforce NFT ownership linkage (strong fix)

```solidity
mapping(uint256 => address) depositorOf;

require(depositorOf[tokenId] == msg.sender);
```

### 2. Restrict unwrap flexibility

* Only allow:

  * Same NFT IDs as deposited
    OR
  * Controlled swap logic with fees

### 3. Add protocol-level safeguards

* Lock NFTs during:

  * staking
  * reward periods
* Track "active utility" NFTs

### 4. Handle airdrops proactively

* Claim rewards at protocol level
* Distribute proportionally to LPs

### 5. Introduce friction

* Unwrap delay
* Exit fees
* Cooldown periods

## Pattern Recognition Notes

### 🔁 Fungibility Assumption Violation

* Treating NFTs as interchangeable ignores:

  * unique rights
  * external dependencies

### 🎭 Temporary Ownership Exploit

* Gaining short-term ownership to:

  * extract value
  * return asset

### 💰 External Value Leakage

* Protocol ignores:

  * off-chain / external rewards
* Leads to value extraction outside system accounting

### 🧨 Missing Asset Binding

* No link:

  * deposit → withdrawal
* Enables arbitrary asset selection

### ⚠️ Composability Risk

* NFTFi protocols must consider:

  * staking
  * airdrops
  * governance
  * reward systems

## Quick Recall (TL;DR)

* **Bug**: Unwrap allows withdrawing **any NFT**, not just deposited ones
* **Impact**:

  * Free NFT swaps
  * Airdrop theft
  * Reward extraction
* **Root Cause**: No mapping between depositor and NFT ID
* **Fix**: Enforce ownership linkage + restrict arbitrary withdrawals
