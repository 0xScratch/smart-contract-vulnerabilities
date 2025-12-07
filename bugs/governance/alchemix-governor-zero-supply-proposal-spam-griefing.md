# Zero-Supply Proposal Spam in AlchemixGovernor (Griefing Attack)

* **Severity**: Medium
* **Source**: [Solodit (Immunefi)](https://solodit.cyfrin.io/issues/bypassing-the-governances-proposal-threshold-to-spam-malicious-proposal-as-a-griefing-attack-immunefi-alchemix-git)
* **Affected Contract**: [`AlchemixGovernor.sol`](https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/AlchemixGovernor.sol)
* **Vulnerability Type**: Governance Misconfiguration / Denial of Service (Spam) / Launch-Phase Weakness

## Summary

`AlchemixGovernor` calculates the **proposal threshold** as a **percentage of the total veALCX supply**. Immediately after deploying the `VotingEscrow` contract (before any user creates a lock), the **total supply is zero**:

```text
proposalThreshold = 0% of 0 = 0
```

This means **any address with 0 voting power can create proposals**, enabling an attacker to **spam unlimited governance proposals** at zero cost. Proposals won't pass (as quorum remains unobtainable), but the **Alchemix team must spend gas and operational time** canceling each malicious proposal.

This is a **pure griefing** vector — no profit, but forced work and noise.

## A Better Explanation (With Simplified Example)

### Intended Behavior

1. **veALCX holders** obtain voting power by locking ALCX in the `VotingEscrow` contract.

2. A user must hold at least `proposalThreshold()` votes to **create a proposal**.

3. `proposalThreshold()` is computed as:

   ```solidity
   totalSupplySnapshot * proposalNumerator / 10,000
   ```

   (default: 4% of total supply)

4. This prevents low-stake actors from spamming proposals.

### What Actually Happens (Bug)

* At protocol deployment, **no one has locked tokens yet**, so:

  ```text
  totalSupply == 0
  proposalThreshold() = (0 * 400) / 10,000 = 0
  ```

* The Governor checks:

  ```solidity
  require(getVotes(msg.sender) > proposalThreshold);
  ```

* Since both are **zero**, the check passes.

**Result**:
Any address with **zero** voting power can successfully call:

```solidity
governor.propose(targets, values, calldatas, description, L2_DOMAIN);
```

This allows:

* Unlimited creation of proposals
* No quorum = no passing, but still clutters governance
* Forces the team to manually cancel each one

Because canceling proposals is permissioned and gas-costly, this becomes an **operational DoS through spam**.

## Why This Matters

* Governance proposals require human review and on-chain cancelation.
* Malicious users can flood:

  * UIs
  * Indexers
  * Governance interfaces
  * Proposal queues
* Even though proposals **cannot pass**, spam reduces trust/signal quality and wastes admin gas.

This is especially impactful at **protocol launch time**, when supply = 0 and the system is most vulnerable.

## Concrete Walkthrough (Alice & Mallory)

* **Setup**:
  `VotingEscrow` deployed, but **no locks exist** → `totalSupply = 0`.

* **Mallory attack**:
  Mallory has **0 veALCX**, but because threshold == 0, she can:

  ```solidity
  governor.propose(...);
  ```

  Repeating this call from multiple fresh addresses → **proposal spam**.

* **Team burden**:
  The Alchemix team must call:

  ```solidity
  governor.cancel(proposalId)
  ```

  for **each spammed proposal**, wasting gas and attention.

* **No user harm in funds**, but governance is polluted.

> **Analogy**: A voting system that requires "4% of all citizens" to propose a motion, but the system launches before anyone has a citizenship card → 0% of 0 people is 0, so literally **anyone on the street** can flood the ballot box with nonsense motions until administrators manually remove them.

## Vulnerable Code Reference

### 1) Proposal threshold computed against total supply (which can be zero)

```solidity
function proposalThreshold() public view override(L2Governor) returns (uint256) {
    return (token.getPastTotalSupply(block.timestamp) * proposalNumerator)
        / PROPOSAL_DENOMINATOR;
}
```

If `getPastTotalSupply(...) = 0`, this always returns **0**.

### 2) Zero threshold bypasses proposer vote check

Inherited from OZ-style Governors:

```solidity
require(getVotes(msg.sender, snapshot) > proposalThreshold());
```

If both are zero → condition passes.

## Recommended Mitigation

### 1. **Set a minimum floor for proposalThreshold > 0**

```solidity
uint256 base = token.getPastTotalSupply(block.timestamp);
uint256 threshold = (base * proposalNumerator) / PROPOSAL_DENOMINATOR;

if (threshold == 0) {
    // floor to prevent zero-supply infinite proposals
    return 1;
}
return threshold;
```

This ensures **no one can propose** when supply = 0.

### 2. **Add a "governanceActivation" switch**

Require governance to be explicitly enabled **after initial veALCX is minted**.

```solidity
require(governanceActive, "governance not yet active");
```

### 3. **Temporarily set a fixed absolute threshold at launch**

Use an absolute minimum like 1 ALCX (or similar) until the system has supply.

### 4. **Automated proposal rate-limiting / cooldown**

Not necessary if a floor is added, but useful for robustness.

## Pattern Recognition Notes

* **Zero-Value Percent Pitfall**:
  Any "percent of total supply" logic becomes exploitable if total supply can be **zero** at system boot.

* **Governance Bootstrapping Hazards**:
  Governance systems requiring minimum stake to propose must enforce:

  * Supply > 0
  * Threshold > 0
  * Activation after initialization

* **Spam / Griefing in Permissionless Flows**:
  Any user-triggerable operation that:

  * costs attacker almost nothing
  * forces admin/operator to spend gas
    = **griefing vector**.

* **Snapshot-Based Voting**:
  Snapshot logic must gracefully handle zero-supply edge cases.

* **Operational DoS**:
  Not all DoS attacks target protocol logic. Some target **human bandwidth** (proposal spam, event spam, log flooding).

## Quick Recall (TL;DR)

* **Bug**: Proposal threshold = percentage of supply. With supply = 0, threshold becomes **0**, so anyone with zero votes can propose.
* **Impact**: Attacker spams infinite proposals → team must cancel them manually → governance noise, time waste, gas cost.
* **Fix**: Add a **minimum non-zero proposal threshold**, or gate proposal creation until veALCX supply is live.
