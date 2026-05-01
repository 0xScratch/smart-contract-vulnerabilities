# Approval Hijack via Missing Ownership Validation in WPunkGateway

* **Severity**: High
* **Source**: [Code4rena](https://github.com/code-423n4/2022-11-paraspace-findings/issues/137)
* **Affected Contract**: [WPunkGateway.sol](https://github.com/code-423n4/2022-11-paraspace/blob/c6820a279c64a299a783955749fdc977de8f0449/paraspace-core/contracts/ui/WPunkGateway.sol)
* **Vulnerability Type**: Access Control / Front-Running / Approval Hijack

## Summary

The `WPunkGateway` contract allows users to deposit CryptoPunks by first granting approval through the non-standard CryptoPunks mechanism (`offerPunkForSaleToAddress`). However, the contract **does not verify that the caller is the actual owner of the Punk** being deposited.

Because of this, once a user approves the gateway to transfer their Punk, **any attacker can front-run the deposit transaction**, call `supplyPunk`, and set themselves as the `onBehalfOf` recipient. This results in the attacker receiving the corresponding collateral tokens (nWPunk), effectively gaining ownership of the deposited Punk.

This leads to **complete theft of user assets**.

## A Better Explanation (With Simplified Example)

### Intended Behavior

1. User owns a CryptoPunk.
2. User approves the gateway using:

   ```
   offerPunkForSaleToAddress(WPunkGateway, 0)
   ```
3. User calls `supplyPunk(...)`.
4. Gateway:

   * Transfers Punk
   * Wraps it into WPunk
   * Supplies it to ParaSpace
   * Mints nWPunk to the user

### What Actually Happens (Bug)

* Approval (`offerPunkForSaleToAddress`) is **publicly visible on-chain**
* The gateway **does not check ownership**
* Anyone can:

  * Call `supplyPunk`
  * Set `onBehalfOf` arbitrarily

👉 The contract only checks:

> "Can I transfer this Punk?"

NOT:

> "Does the caller own this Punk?"

### Why This Matters

* Approval is **decoupled from execution**
* Approval is **not exclusive to the owner's transaction**
* This creates a **front-running window**

👉 Result:

> Whoever calls `supplyPunk` first gets the credit

### Concrete Walkthrough (Alice & Mallory)

#### Step 1: Alice prepares deposit

Alice owns Punk #1 and calls:

```solidity
offerPunkForSaleToAddress(WPunkGateway, 0)
```

👉 This allows the gateway to transfer her Punk.

#### Step 2: Mallory monitors mempool

Mallory detects:

* Approval transaction
* Incoming deposit intent

#### Step 3: Mallory front-runs

Mallory calls:

```solidity
supplyPunk([1], onBehalfOf = Mallory)
```

#### Step 4: Contract executes blindly

Inside `supplyPunk`:

* Gateway buys Punk (allowed due to approval)
* Wraps it into WPunk
* Supplies to pool
* Mints nWPunk to **Mallory**

#### Final Outcome

| Actor   | Result                                   |
| ------- | ---------------------------------------- |
| Alice   | ❌ Lost Punk                              |
| Mallory | ✅ Gains nWPunk (can later withdraw Punk) |

> **Analogy**:
> You leave your car keys at a valet counter (approval), intending to park your car yourself. Someone else grabs the keys first and parks the car under their name — now they own the parking ticket.

## Vulnerable Code Reference

### 1) Missing ownership validation in `supplyPunk`

```solidity
for (uint256 i = 0; i < punkIndexes.length; i++) {
    Punk.buyPunk(punkIndexes[i].tokenId);
    Punk.transferPunk(proxy, punkIndexes[i].tokenId);
    WPunk.mint(punkIndexes[i].tokenId);
}
```

❌ Missing:

```solidity
require(Punk.punkIndexToAddress(tokenId) == msg.sender);
```

### 2) Arbitrary `onBehalfOf` parameter

```solidity
function supplyPunk(
    ...,
    address onBehalfOf,
    ...
)
```

❌ Anyone can redirect ownership of minted tokens.

### 3) Same issue in credit functions

```solidity
acceptBidWithCredit(...)
batchAcceptBidWithCredit(...)
```

* Same flow:

  * Transfer Punk
  * Wrap
  * Assign benefit to caller

## Recommended Mitigation

### 1. Enforce ownership check (primary fix)

```solidity
address owner = Punk.punkIndexToAddress(tokenId);
require(owner == msg.sender, "Not punk owner");
```

### 2. Bind execution to caller

Optional stricter design:

```solidity
require(onBehalfOf == msg.sender);
```

### 3. Reduce approval attack surface

* Avoid relying on:

  ```text
  offerPunkForSaleToAddress
  ```
* Or enforce immediate execution pattern (approval + action atomically)

### 4. Add invariant tests

* Ensure:

  * Only owner can deposit
  * Front-running cannot redirect ownership

## Pattern Recognition Notes

* **Approval Hijack Pattern**
  When a protocol relies on external approval mechanisms, but **does not bind execution to the approver**, attackers can steal assets.

* **Missing Ownership Validation**
  Never assume that:

  > "If transfer succeeds → caller is legitimate"

* **Decoupled Approval + Execution**
  Any system where:

  * Step 1: Approve
  * Step 2: Execute later
    is vulnerable to front-running unless tightly validated.

* **User-Controlled Recipient Parameters**
  Allowing arbitrary `onBehalfOf` without validation is dangerous in asset transfer flows.

* **Non-Standard Token Risks**
  Legacy systems like CryptoPunks introduce edge cases:

  * No `transferFrom`
  * No standard approval
  * Custom mechanics → higher risk surface

## Quick Recall (TL;DR)

* **Bug**: No ownership check when depositing Punk
* **Exploit**: Attacker front-runs after approval
* **Impact**: Attacker receives nWPunk → steals Punk
* **Fix**: Validate `msg.sender` is Punk owner
