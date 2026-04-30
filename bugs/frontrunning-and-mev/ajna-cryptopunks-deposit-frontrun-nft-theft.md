# CryptoPunks Deposit Frontrunning Leading to NFT Theft

* **Severity**: High
* **Source**: [Sherlock](https://github.com/sherlock-audit/2023-01-ajna-judging/issues/140)
* **Affected Contract**: [ERC721Pool.sol](https://github.com/sherlock-audit/2023-01-ajna/blob/main/contracts/src/ERC721Pool.sol#L577)
* **Vulnerability Type**: Front-Running / Missing Ownership Validation / Unauthorized Asset Transfer

## Summary

Depositing **CryptoPunks NFTs into Ajna pools can be **front-run**, allowing a malicious actor to deposit **someone else's NFT** and receive the corresponding collateral credit.

This happens because CryptoPunks does **not follow the ERC-721 standard**, forcing Ajna to implement deposits via a **purchase mechanism (`buyPunk`)** instead of a direct transfer. However, the `addCollateral` function **does not verify that the caller owns the NFT**, allowing anyone to trigger the purchase and claim the credit.

As a result, an attacker can **steal NFTs during the deposit process**.

## A Better Explanation (With Simplified Example)

### Intended Behavior

1. User owns a CryptoPunk NFT.
2. User calls:

   * `offerPunkForSaleToAddress(tokenId, price, poolAddress)`
     → NFT is listed for sale **only to the pool**.
3. User calls:

   * `addCollateral(tokenId)`
4. Pool internally calls:

   * `buyPunk(tokenId)`
5. NFT is transferred to the pool.
6. **Collateral credit is assigned to the user**.

### What Actually Happens (Bug)

* `addCollateral` is **permissionless** (anyone can call it).
* There is **no check** ensuring:

  ```solidity
  punkIndexToAddress(tokenId) == msg.sender
  ```
* The pool executes `buyPunk()` regardless of who called it.
* The **collateral is credited to the caller**, not the NFT owner.

### Why This Matters

* The deposit process becomes **race-condition dependent**.
* Any observer can **front-run deposits** from the mempool.
* Victims lose NFTs **without interacting directly with the attacker**.
* This is a **silent theft vector**, not just a griefing issue.

### Concrete Walkthrough (Alice & Mallory)

* **Setup**: Alice owns Punk #123.

#### Step 1 — Alice prepares deposit

* Calls:

  ```solidity
  offerPunkForSaleToAddress(123, price, pool)
  ```
* NFT is now sellable **only to the pool**

#### Step 2 — Mallory observes mempool

* Sees Alice is about to call `addCollateral`

#### Step 3 — Mallory front-runs

* Calls:

  ```solidity
  addCollateral(123)
  ```
* Uses higher gas → executes first

#### Step 4 — Pool executes purchase

* Pool calls:

  ```solidity
  buyPunk(123)
  ```
* Inside **CryptoPunksMarket:

  * `msg.sender = pool` ✅ allowed buyer
* NFT transferred:

  * Alice → Pool

#### Step 5 — Credit misassigned

* Ajna credits collateral to:

  * **Mallory (caller of addCollateral)** ❌

#### Step 6 — Theft completed

* Mallory can:

  * withdraw NFT from pool
* Alice:

  * loses NFT permanently

> **Analogy**: You authorize a vault to pick up your asset, but anyone can press the "deposit" button and claim the vault receipt.

## Vulnerable Code Reference

**1) Missing ownership validation before purchase**

```solidity
// ERC721Pool.sol
ICryptoPunks(collateralAddress).buyPunk(tokenId);
```

❌ Missing:

```solidity
require(
  ICryptoPunks(collateralAddress).punkIndexToAddress(tokenId) == msg.sender
);
```

**2) Permissionless deposit function**

```solidity
function addCollateral(uint256 tokenId) external {
    // no ownership check
    _transferOrBuyNFT(tokenId);
    _creditCollateral(msg.sender);
}
```

👉 Core issue:

* NFT transfer logic is **decoupled from caller identity**
* Credit is assigned to `msg.sender`

**3) CryptoPunks purchase logic allows this flow**

```solidity
if (offer.onlySellTo != 0x0 && offer.onlySellTo != msg.sender) throw;
```

✔ Restriction applies to:

* **buyer (pool)**

❌ Not:

* the caller who initiated the process

## Recommended Mitigation

### 1. Enforce ownership at deposit (primary fix)

```solidity
require(
  ICryptoPunks(collateralAddress).punkIndexToAddress(tokenId) == msg.sender,
  "Not owner"
);
```

### 2. Bind credit to actual owner (defensive)

Instead of:

```solidity
_creditCollateral(msg.sender);
```

Consider:

* fetching owner before purchase and crediting that address

### 3. Avoid indirect "buy-based" transfers when possible

* Prefer:

  * **user-initiated transfers**
* If unavoidable:

  * enforce strict ownership checks

### 4. Add mempool/front-run aware tests

* Simulate:

  * attacker calling deposit before victim
* Ensure:

  * ownership mismatch reverts

## Pattern Recognition Notes

* **Permissionless Entry + Value Assignment**

  * Any function that assigns value to `msg.sender` must validate eligibility.

* **Non-Standard Token Integration Risk**

  * Custom flows (like CryptoPunks "buy") often bypass standard safety guarantees.

* **Front-Running via Mempool Visibility**

  * Multi-step flows (approve → act) are especially vulnerable.

* **Caller vs Asset Owner Mismatch**

  * If these differ and aren't validated → theft risk.

* **Pull-Based Asset Acquisition**

  * When contracts *pull assets themselves*, ownership checks become critical.

## Quick Recall (TL;DR)

* **Bug**: No ownership check in `addCollateral` for CryptoPunks deposits
* **Exploit**: Attacker front-runs deposit and calls function first
* **Result**: Pool buys NFT → credit goes to attacker
* **Impact**: **Full NFT theft**
* **Fix**: Require `punkIndexToAddress(tokenId) == msg.sender`
