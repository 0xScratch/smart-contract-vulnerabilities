# Reorg-Based Vault Address Hijacking via CREATE Deployment in Amphora

* **Severity**: Medium
* **Source**: [Code4rena](https://github.com/code-423n4/2023-07-amphora-findings/issues/233)
* **Affected Contracts**:
  * [VaultDeployer.sol](https://github.com/code-423n4/2023-07-amphora/blob/main/core/solidity/contracts/core/VaultDeployer.sol#L28)
  * [Vault.sol](https://github.com/code-423n4/2023-07-amphora/blob/main/core/solidity/contracts/core/Vault.sol)
* **Vulnerability Type**: Reorg / Address-Stealing / Deployment-Order Dependency / Business Logic Flaw

## Summary

Amphora deploys user-specific Vault contracts using **`CREATE`**, where the deployed address depends on the **VaultDeployer's nonce**. Users are expected to **deploy a Vault and immediately deposit funds into it**.

In the presence of a **chain reorg**, the deployment order of transactions can change. An attacker can exploit this by **pre-deploying a malicious Vault at the address the user originally expected**, causing the user's deposit to be sent to an attacker-controlled Vault. The attacker can then drain the deposited funds.

This issue arises from combining:

* nonce-based contract deployment (`CREATE`)
* user-initiated contract creation
* immediate reliance on the deployed address
* lack of reorg resilience

## A Better Explanation (With Simplified Example)

### Intended Behavior

1. **Vault Deployment**
   A user calls `VaultDeployer.deployVault(id, minter)`
   → a new Vault is deployed at an address derived from the deployer's nonce.

2. **Deposit**
   The user deposits assets into the newly created Vault address.

3. **Ownership Assumption**
   The protocol assumes the deployed address uniquely belongs to the user.

### What Actually Happens (Bug)

Because `CREATE` derives addresses from `(deployer, nonce)`:

* Vault addresses are **predictable**
* Vault addresses **depend on deployment order**
* Reorgs can **rewind and reorder deployments**

If a reorg occurs between deployment and deposit, an attacker can **front-run the redeployment** and claim the expected Vault address.

### Concrete Walkthrough (Alice & Mallory)

#### Setup

* Alice submits:

  1. `deployVault(AliceVault)`
  2. `deposit(1000 USDC → VaultAddress)`

#### Reorg Happens

* Both transactions are removed from the canonical chain.

#### Mallory Attacks During Reorg Replay

1. Mallory predicts the Vault address Alice's deployment would create.
2. Mallory **front-runs** and deploys a **malicious Vault** at that address.
3. Alice's deployment now executes later and creates a Vault at a **different address**.
4. Alice's deposit transaction still targets the **original address**.
5. Funds are deposited into **Mallory's Vault**, not Alice's.

#### Outcome

* Alice's deposit succeeds on-chain
* Mallory controls the Vault
* Mallory drains Alice's funds

### Why This Matters

* Funds can be **stolen without reverting**
* User actions appear successful
* Attack is invisible to the user
* Risk increases significantly on L2s / alt-L1s with deep reorgs
* Breaks the assumption that "deploy → deposit" is safe

> **Analogy**:
> You're told your hotel room number, but during a power outage someone else claims that room first. You walk in and leave your luggage — but it's no longer your room.

## Vulnerable Code Reference

### 1) Vault deployment uses `CREATE` (nonce-based)

```solidity
function deployVault(uint96 _id, address _minter) external returns (IVault _vault) {
    _vault = IVault(new Vault(_id, _minter, msg.sender, CVX, CRV));
}
```

* Vault address depends on `VaultDeployer` nonce
* Address changes if deployment order changes

### 2) User deposits rely on assumed Vault address

```solidity
// Vault.deposit(...)
require(msg.sender == minter, "not minter");
```

* Deposit does not verify Vault authenticity
* Assumes the Vault at that address belongs to the user

### 3) No reorg-safe binding between user and Vault address

* No deterministic address derivation
* No CREATE2
* No salt binding Vault address to user identity

## Recommended Mitigation

### 1. Use CREATE2 for Vault Deployment (Primary Fix)

Derive the Vault address using a salt bound to the user:

```solidity
salt = keccak256(abi.encode(id, minter, msg.sender));
```

This ensures:

* Vault address is deterministic
* Vault address does not depend on nonce
* Attacker cannot pre-deploy the same address

### 2. Bind Vault Identity Strongly

Optionally:

* Store a mapping of `(user → vault address)`
* Verify deposits only target the expected Vault

### 3. Avoid Critical Logic Immediately After Deployment

* Consider delaying deposits until sufficient confirmations
* Or combine deployment + initialization atomically

## Pattern Recognition Notes

* **Nonce-Based Address Dependency**
  Using `CREATE` for user-specific contracts introduces address instability under reorgs.

* **Deploy-Then-Deposit Pattern**
  Any flow where users deposit funds immediately after contract creation is reorg-sensitive.

* **Address Assumption Bugs**
  Assuming "this address must be mine" without cryptographic binding is unsafe.

* **Reorg-Amplified Attacks**
  Low-likelihood issues on Ethereum L1 become realistic on L2s with deep reorgs.

* **CREATE2 as a Security Primitive**
  CREATE2 is not an optimization — it is a **security boundary** for user-bound contracts.

## Quick Recall (TL;DR)

* **Bug**: Vaults deployed with `CREATE` → address depends on nonce
* **Trigger**: Chain reorg reorders deployment
* **Attack**: Attacker front-runs and steals expected Vault address
* **Impact**: User deposits into attacker-controlled Vault → funds stolen
* **Fix**: Use `CREATE2` with user-bound salt
