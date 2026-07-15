# Service NFT Unstake DoS via Reward Token Blocklist or Pause

* **Severity**: Medium
* **Source**: [Code4rena](https://github.com/code-423n4/2024-05-olas-findings/issues/31)
* **Affected Contract**: [StakingBase.sol](https://github.com/code-423n4/2024-05-olas/blob/3ce502ec8b475885b90668e617f3983cea3ae29f/registries/contracts/staking/StakingBase.sol#L868)
* **Vulnerability Type**: Denial of Service (DoS) / External Dependency Failure / Coupled Operations

## Summary

`StakingBase` allows service owners to stake a **Service NFT**, earning reward tokens over time. When the owner later calls `unstake()`, the contract performs **two actions in a single transaction**:

1. Returns the Service NFT to the owner.
2. Transfers all accumulated rewards to the service's stored multisig address.

The problem is that the reward transfer depends on an **external ERC20 token** that may implement features such as **blocklisting** or **pausing**. If the reward transfer fails because the multisig has been blocklisted or the token is paused, the **entire transaction reverts**.

Since Ethereum transactions are atomic, the NFT transfer is rolled back as well, leaving the user's Service NFT permanently (or temporarily) locked inside the staking contract.

## A Better Explanation (With Simplified Example)

### Intended Behavior

1. **Stake**

   * Alice stakes her Service NFT.
   * The staking contract takes custody of the NFT.
   * The contract stores the service's multisig address inside `ServiceInfo`.
   * Rewards begin accumulating.

2. **Unstake**

   * Alice calls `unstake()`.
   * Contract transfers the NFT back to Alice.
   * Contract sends accumulated rewards to the stored multisig.
   * Unstake completes successfully.

Everything works as expected.

---

### What Actually Happens (Bug)

The staking contract permanently stores the service's multisig address when staking occurs.

Later, suppose one of the following happens:

* the reward token blocklists that multisig address, or
* the reward token pauses all transfers.

Now when `unstake()` attempts to send rewards, the ERC20 transfer reverts.

Because the reward payment and NFT return occur in the **same transaction**, Ethereum rolls back **everything**, including the successful NFT transfer.

As a result, the owner cannot retrieve their Service NFT even though the NFT itself has no issue.

---

### Why This Matters

* A user's Service NFT can become permanently locked if the multisig is blocklisted.
* A paused reward token temporarily prevents all unstaking.
* Users also cannot claim accumulated rewards since rewards always go to the stored multisig.
* The failure is caused by an external token feature, but it bricks an unrelated protocol operation.

## Concrete Walkthrough (Alice & Token Admin)

### Setup

Alice owns:

```
Service NFT #42
```

She stakes it.

The contract stores:

```
owner    = Alice
multisig = 0xABC...
reward   = 0
```

Months later:

```
reward = 500 OLAS
```

---

### The Problem

Later, the reward token blocklists:

```
0xABC...
```

Alice now calls:

```solidity
unstake()
```

The function executes:

```
Transfer Service NFT back

↓

Send 500 reward tokens to 0xABC...

↓

Reward token rejects transfer

↓

Transaction reverts
```

Because the transaction reverted:

```
NFT transfer is undone

↓

Service NFT remains inside StakingBase
```

Alice can never retrieve her Service NFT until the blocklist issue is resolved (or forever if it is permanent).

---

> **Analogy:** Imagine checking out of a hotel. Before returning your luggage, the receptionist first tries to refund money to a debit card that has been frozen. Since the refund fails, the hotel refuses to give back your luggage—even though the luggage has nothing to do with the payment problem.

## Vulnerable Code Reference

### 1) Multisig address is stored during staking

During staking, the contract records the service's multisig address inside `ServiceInfo`.

```
ServiceInfo {
    owner,
    multisig,
    ...
}
```

This address is later used for every reward withdrawal and cannot be updated.

---

### 2) `unstake()` couples NFT return with reward payment

```solidity
// Transfer the service back to the owner
IService(serviceRegistry).safeTransferFrom(
    address(this),
    msg.sender,
    serviceId
);

// Transfer accumulated rewards
if (reward > 0) {
    _withdraw(multisig, reward);
}
```

If `_withdraw()` fails, the entire transaction reverts.

---

### 3) Reward transfer depends on external token behavior

`_withdraw()` ultimately performs an ERC20 transfer.

If the reward token:

* blocklists the multisig,
* pauses transfers,
* or otherwise rejects transfers,

the reward transfer fails and `unstake()` cannot complete.

## Recommended Mitigation

### 1. Decouple unstaking from reward withdrawal (Preferred)

Returning the Service NFT should not depend on successfully sending rewards.

Instead:

```
unstake()

↓

Return NFT

↓

Record pending rewards
```

Then let users claim rewards separately:

```
claimRewards()
```

This ensures reward transfer failures never prevent NFT retrieval.

---

### 2. Allow updating the stored multisig

If the Service Registry allows changing the service multisig, the staking contract should synchronize with it so rewards are not permanently tied to an unusable address.

---

### 3. Gracefully handle reward transfer failures

Instead of reverting the entire unstake, consider:

* recording unpaid rewards,
* emitting an event,
* allowing later retry.

Critical user assets should never depend on optional reward transfers.

---

### 4. Test against blocklisted and paused tokens

Include unit tests covering:

* blocklisted reward recipient,
* paused reward token,
* reward transfer failures,

while verifying that unstaking still succeeds.

## Pattern Recognition Notes

* **Coupled Critical and Non-Critical Operations**: Never bundle essential asset recovery (unstake) with optional side effects (reward payment). Failure of the latter should not block the former.
* **External Dependency Risk**: Any reliance on external token behavior (blocklists, pause mechanisms, transfer hooks) can unexpectedly break protocol logic.
* **Atomic Transaction Awareness**: Even if a critical action executes first, a later revert rolls back the entire transaction. Ordering alone does not isolate failures.
* **Immutable Recipient Risk**: Storing a recipient address permanently can become dangerous if that address later becomes unusable. Consider allowing updates or fallback mechanisms.
* **Failure Isolation**: Separate user asset retrieval from reward distribution so each can fail independently without affecting the other.

## Quick Recall (TL;DR)

* **Bug:** `unstake()` returns the Service NFT and pays rewards in the same transaction.
* **Trigger:** The reward token blocklists the stored multisig or pauses transfers.
* **Impact:** Reward transfer reverts → entire transaction reverts → NFT cannot be unstaked.
* **Root Cause:** Critical asset retrieval is tightly coupled with a non-critical external token transfer.
* **Fix:** Decouple NFT retrieval from reward distribution and allow rewards to be claimed separately.
