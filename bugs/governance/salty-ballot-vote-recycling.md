# Vote Inflation via SALT Recycling in Proposals.sol

* **Severity**: Medium
* **Source**: [Code4rena](https://github.com/code-423n4/2024-01-salty-findings/issues/844) / [One Bug Per Day](https://www.onebugperday.com/v1/902)
* **Affected Contract**: [Proposals.sol](https://github.com/code-423n4/2024-01-salty/blob/main/src/dao/Proposals.sol)
* **Vulnerability Type**: Governance Manipulation / Vote Inflation / Snapshot Missing

## Summary

The governance contract (`Proposals.sol`) calculates voting power dynamically from the **current staked SALT balance** (`staking.userShareForPool`).  
Since the system does **not snapshot stake at proposal creation** and ballots remain open indefinitely until quorum is reached, the **same SALT can be unstaked, transferred, restaked, and voted again** on the same proposal using a different account.  

This creates **vote inflation**, where a single chunk of SALT can be recycled across accounts, multiplying its influence. An attacker can exploit this to pass malicious proposals such as draining DAO funds.

## A Better Explanation (With Simplified Example)

### Intended Behavior

1. **Create Proposal**: A SALT staker creates a proposal (e.g., whitelist a token).  
2. **Voting**: Each staker's voting power equals their staked SALT balance.  
3. **Finalization**: Once quorum is reached and minimum time passes, the proposal can be finalized.

### What Actually Happens (Bug)

* Voting power is checked **live** when `castVote` is called:  

  ```solidity
  uint256 userVotingPower = staking.userShareForPool(msg.sender, PoolUtils.STAKED_SALT);
  ```

* There is **no snapshot** at the time of proposal creation.
* Proposals **never expire** if quorum is not reached.
* This allows a staker to:

  1. Stake SALT
  2. Vote once
  3. Unstake SALT
  4. Transfer SALT to another account
  5. Restake and vote again

➡️ **The same SALT is counted multiple times** toward quorum and approval.

### Why This Matters

* Quorum is supposed to reflect **real community participation**.
* Recycling votes undermines quorum by **artificially inflating participation**.
* A single attacker can **force malicious proposals through** with far less SALT than quorum intends.
* This directly threatens the DAO's funds and governance integrity.

### Concrete Walkthrough (Alice & Bob)

1. **Alice stakes 100,000 SALT** and creates a proposal.
2. Alice votes **YES** (100,000 votes recorded).
3. Alice unstakes and recovers the same 100,000 SALT.
4. She transfers the SALT to Bob's address.
5. Bob stakes the 100,000 SALT and votes **YES**.
6. Now the proposal shows **200,000 YES votes**, but only 100,000 SALT ever existed.
7. Repeating this process can push the proposal past quorum and approval thresholds.

> **Analogy**: It's like voting in an election, leaving, changing clothes, and coming back to vote again. Since the system doesn't check IDs, the same person can cast multiple ballots.

## Vulnerable Code Reference

### 1) `castVote` uses live balance instead of a snapshot**

[Proposals.sol#L259-L293](https://github.com/code-423n4/2024-01-salty/blob/main/src/dao/Proposals.sol#L259-L293)

```solidity
uint256 userVotingPower = staking.userShareForPool(msg.sender, PoolUtils.STAKED_SALT);
// ...
_votesCastForBallot[ballotID][vote] += userVotingPower;
```

### 2) `canFinalizeBallot` allows indefinite voting until quorum**

[Proposals.sol#L385-L400](https://github.com/code-423n4/2024-01-salty/blob/main/src/dao/Proposals.sol#L385-L400)

```solidity
if ( totalVotesCastForBallot(ballotID) < requiredQuorumForBallotType(ballot.ballotType) )
    return false;
// no maximum end time → ballot remains open forever
```

## Recommended Mitigation

1. **Snapshot Voting Power**

   * Adopt a system like OpenZeppelin's `ERC20Votes`, which snapshots token balances at proposal creation.
   * This ensures voting power cannot be reused after unstaking or transfers.

2. **Introduce `ballotMaximumEndTime`**

   * Cap how long proposals remain open.
   * If quorum isn't met by deadline, proposal expires.

3. **Stake Locking**

   * Optionally lock SALT until proposal finalization to prevent mid-ballot recycling.

## Pattern Recognition Notes

* **Missing Snapshotting**: Governance must bind voting power to a fixed state (block number or timestamp). Using live balances invites replay and recycling issues.
* **Infinite Voting Window**: Allowing proposals to stay open indefinitely increases the chance of exploitation. Always enforce a max end time.
* **Vote Inflation Risk**: Similar issues have occurred in other DAOs where voting power wasn't decoupled from token transfers.
* **Governance Security Principle**: Once a vote is cast, its weight should remain immutable regardless of later token transfers.

### Quick Recall (TL;DR)

* **Bug**: Voting power uses live balances and proposals never expire.
* **Impact**: SALT can be recycled across accounts to inflate votes → attacker can pass malicious proposals.
* **Fix**: Use snapshots for voting power + enforce max ballot duration.
