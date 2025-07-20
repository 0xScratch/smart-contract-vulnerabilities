# Fully Slashed Transcoder Vote Override Denial of Service Vulnerability

- **Severity**: Medium
- **Source**: [Code4rena](https://github.com/code-423n4/2023-08-livepeer-findings/issues/194) / [One Bug Per Day](https://www.onebugperday.com/v1/477)
- **Affected Contract**: [GovernorCountingOverridable.sol](https://github.com/code-423n4/2023-08-livepeer/blob/a3d801fa4690119b6f96aeb5508e58d752bda5bc/contracts/treasury/GovernorCountingOverridable.sol#L181-L189) / [BondingManager.sol](https://github.com/code-423n4/2023-08-livepeer/blob/a3d801fa4690119b6f96aeb5508e58d752bda5bc/contracts/bonding/BondingManager.sol#L402-L418)
- **Vulnerability Type**: Denial of Service / Governance Disruption

## Original Bug Description

## Vulnerability details

> ### Impact
>
> If a transcoder gets slashed fully he can still vote with 0 amount of `weight` making any other delegated user that wants to change his vote to subtract their `weight` amount from other delegators/transcoders.
>
> ### Proof of Concept
>
> In `BondingManager.sol` any transcoder can gets slashed by a specific percentage, and that specific transcoder gets resigned and that specific percentage gets deducted from his `bondedAmount`
>
> [https://github.com/code-423n4/2023-08-livepeer/blob/a3d801fa4690119b6f96aeb5508e58d752bda5bc/contracts/bonding/BondingManager.sol\#L394-L411](https://github.com/code-423n4/2023-08-livepeer/blob/a3d801fa4690119b6f96aeb5508e58d752bda5bc/contracts/bonding/BondingManager.sol#L394-L411)
>
> If any `bondedAmount` will remain then the penalty will also gets subtracted from the `delegatedAmount`, if not, nothing happens
>
> [https://github.com/code-423n4/2023-08-livepeer/blob/a3d801fa4690119b6f96aeb5508e58d752bda5bc/contracts/bonding/BondingManager.sol\#L412-L417](https://github.com/code-423n4/2023-08-livepeer/blob/a3d801fa4690119b6f96aeb5508e58d752bda5bc/contracts/bonding/BondingManager.sol#L412-L417)
>
> After that the `penalty` gets burned and the fees gets paid to the finder, if that is the case
>
> [https://github.com/code-423n4/2023-08-livepeer/blob/a3d801fa4690119b6f96aeb5508e58d752bda5bc/contracts/bonding/BondingManager.sol\#L420-L440](https://github.com/code-423n4/2023-08-livepeer/blob/a3d801fa4690119b6f96aeb5508e58d752bda5bc/contracts/bonding/BondingManager.sol#L420-L440)
>
> The problem relies in the fact that a fully slashed transcoder, even if it gets resigned, he is still seen as an active transcoder in the case of voting. Let's assume this case:
>
> - a transcoder gets fully slashed and gets resigned from the transcoder pools, getting his `bondedAmount` to 0, but he still has `delegatedAmount` to his address since nothing happens this variable [https://github.com/code-423n4/2023-08-livepeer/blob/a3d801fa4690119b6f96aeb5508e58d752bda5bc/contracts/bonding/BondingManager.sol\#L402-L418](https://github.com/code-423n4/2023-08-livepeer/blob/a3d801fa4690119b6f96aeb5508e58d752bda5bc/contracts/bonding/BondingManager.sol#L402-L418)
> - transcoder vote a proposal and when his weighting gets calculated here
>[https://github.com/OpenZeppelin/openzeppelin-contracts-upgradeable/blob/f34a3a7e5a1d698d87d517fda698d48286310bee/contracts/governance/GovernorUpgradeable.sol\#L581](https://github.com/OpenZeppelin/openzeppelin-contracts-upgradeable/blob/f34a3a7e5a1d698d87d517fda698d48286310bee/contracts/governance/GovernorUpgradeable.sol#L581), it will use the `_getVotes` from `GovernorVotesUpgradeable.sol` [https://github.com/OpenZeppelin/openzeppelin-contracts-upgradeable/blob/f34a3a7e5a1d698d87d517fda698d48286310bee/contracts/governance/extensions/GovernorVotesUpgradeable.sol\#L55-L61](https://github.com/OpenZeppelin/openzeppelin-contracts-upgradeable/blob/f34a3a7e5a1d698d87d517fda698d48286310bee/contracts/governance/extensions/GovernorVotesUpgradeable.sol#L55-L61)
> - `_getVotes` calls `getPastVotes` on `BondingVotes.sol` [https://github.com/code-423n4/2023-08-livepeer/blob/a3d801fa4690119b6f96aeb5508e58d752bda5bc/contracts/bonding/BondingVotes.sol\#L167-L170](https://github.com/code-423n4/2023-08-livepeer/blob/a3d801fa4690119b6f96aeb5508e58d752bda5bc/contracts/bonding/BondingVotes.sol#L167-L170) which returns the amount of weight specific to this transcoder and as you can see, because the transcoder has a `bondedAmount` equal to 0, the first if statement will be true and 0 will be returned [https://github.com/code-423n4/2023-08-livepeer/blob/a3d801fa4690119b6f96aeb5508e58d752bda5bc/contracts/bonding/BondingVotes.sol\#L372-L373](https://github.com/code-423n4/2023-08-livepeer/blob/a3d801fa4690119b6f96aeb5508e58d752bda5bc/contracts/bonding/BondingVotes.sol#L372-L373)
> - 0 weight will be passed into `_countVote` which will then be passed into `_handleVoteOverrides` [https://github.com/code-423n4/2023-08-livepeer/blob/a3d801fa4690119b6f96aeb5508e58d752bda5bc/contracts/treasury/GovernorCountingOverridable.sol\#L151](https://github.com/code-423n4/2023-08-livepeer/blob/a3d801fa4690119b6f96aeb5508e58d752bda5bc/contracts/treasury/GovernorCountingOverridable.sol#L151)
> - then it will check if the caller is a transcoder, which will be true in our case, because nowhere in the `slashTranscoder` function, or any other function the transcoder `delegateAddress` gets changed, so this if statement will be true [https://github.com/code-423n4/2023-08-livepeer/blob/a3d801fa4690119b6f96aeb5508e58d752bda5bc/contracts/treasury/GovernorCountingOverridable.sol\#L182-L184](https://github.com/code-423n4/2023-08-livepeer/blob/a3d801fa4690119b6f96aeb5508e58d752bda5bc/contracts/treasury/GovernorCountingOverridable.sol#L182-L184), which will try to deduct the weight from any previous delegators [https://github.com/code-423n4/2023-08-livepeer/blob/a3d801fa4690119b6f96aeb5508e58d752bda5bc/contracts/treasury/GovernorCountingOverridable.sol\#L184-L189](https://github.com/code-423n4/2023-08-livepeer/blob/a3d801fa4690119b6f96aeb5508e58d752bda5bc/contracts/treasury/GovernorCountingOverridable.sol#L184-L189)
> - if any delegator already overridden any amount this subtraction would revert, but if that is not the case, 0 weight will be returned, which is then used to vote `for`, `against` , `abstain`, but since 0 weight is passed no changed will be made.
> - now the big problem arise, if any delegator that delegated their votes to this specific transcoder want to change their vote, when his weight gets calculated, `delegatorCumulativeStakeAt` gets called [https://github.com/code-423n4/2023-08-livepeer/blob/a3d801fa4690119b6f96aeb5508e58d752bda5bc/contracts/bonding/BondingVotes.sol\#L459-L487](https://github.com/code-423n4/2023-08-livepeer/blob/a3d801fa4690119b6f96aeb5508e58d752bda5bc/contracts/bonding/BondingVotes.sol#L459-L487) which will return most of the time his `bondedAmount`, amount which is greater than 0, since he didn't unbound.
> - because of that when `_handleVoteOverrides` gets called in `_countVote`, to override the vote, this if statement will be true, since the transcoder voted already, but with 0 weight [https://github.com/code-423n4/2023-08-livepeer/blob/a3d801fa4690119b6f96aeb5508e58d752bda5bc/contracts/treasury/GovernorCountingOverridable.sol\#L195](https://github.com/code-423n4/2023-08-livepeer/blob/a3d801fa4690119b6f96aeb5508e58d752bda5bc/contracts/treasury/GovernorCountingOverridable.sol#L195) and the delegator weight gets subtracted from the support that the transcoder used in his vote
> - the system in this case expect the transcoder to vote with the whole `delegatedAmount`, which will make the subtraction viable, since the weight of the delegator should be included in the full `delegatedAmount` of that specific transcoder, but since the transcoder voted with 0 weight, the subtraction would be done from other delegators/transcoders votes.
> - also this can be abused by a transcoder by voting a category which he knows will not get a lot of votes, if let's say a transcoder used his 0 weight to vote for `abstain` and every other voter will vote on `for` or `against`, every time one of his delegators want to change the vote the subtraction can revert, which will force those delegators out of the vote, until they will change their transcoder
>
>### Tools Used
>
>Manual review
>
>### Recommended Mitigation Steps
>
> If a transcoder gets fully slashed and resigned, delete his address from `delegateAddress` so he will not appear as a transcoder in the mechanism of counting the votes. If he still wants to participate in the system he can act as a delegator to another transcoder. Another solution would be to not let 0 weight votes happen anyways, since they don't modify the vote state at all.

## Summary

A fully slashed transcoder with zero bonded amount can still vote with zero weight, causing arithmetic underflow errors when their delegators attempt to override the vote. This creates a **denial-of-service attack** vector that prevents legitimate delegators from participating in governance, effectively disenfranchising stakeholders and disrupting the protocol's democratic decision-making process.

## Key Details

- **Pattern**: State inconsistency between slashing logic and voting system creates exploitable edge case
- **Root Cause**: `delegateAddress` field remains unchanged after full slashing, maintaining transcoder status for voting
- **Risk**: Legitimate stakeholders lose voting rights due to flawed contract logic, not permissions or stake requirements

## Vulnerable Code Reference

The vulnerability arises in `BondingManager.sol` when a transcoder is fully slashed in `slashTranscoder` function, but their `delegateAddress` remains unchanged. As you can see in the code snippet below, the `delegateAddress` is not reset:

```solidity
function slashTranscoder(...) external {
    ...
    if (del.bondedAmount > 0) {
        uint256 penalty = MathUtils.percOf(delegators[_transcoder].bondedAmount, _slashAmount);

        // If active transcoder, resign it
        if (transcoderPool.contains(_transcoder)) {
            resignTranscoder(_transcoder);
        }

        // Decrease bonded stake
        del.bondedAmount = del.bondedAmount.sub(penalty);

        // If still bonded decrease delegate's delegated amount
        if (delegatorStatus(_transcoder) == DelegatorStatus.Bonded) {
            delegators[del.delegateAddress].delegatedAmount = delegators[del.delegateAddress].delegatedAmount.sub(
                penalty
            );
        }
    ...
}
```

The vulnerability exists in the vote override handling mechanism within `GovernorCountingOverridable.sol`:

```solidity
function _handleVoteOverrides(...) internal returns (uint256) {
    address delegate = votes().delegatedAt(_account, timepoint);
    bool isTranscoder = _account == delegate;  // ❌ Still true for slashed transcoder
    
    if (isTranscoder) {
        return _weight - _voter.deductions;  // Returns 0 - 0 = 0 for slashed transcoder
    }
    
    // For delegator overrides
    if (delegateVoter.hasVoted) {  // TRUE - slashed transcoder voted with 0 weight
        if (delegateSupport == VoteType.For) {
            _tally.forVotes -= _weight;  // ❌ Attempts to subtract from wrong vote category
        }
    }
}
```

## Attack Scenario Breakdown

The only thing you need to keep in mind is that the `delegateAddress` of a slashed transcoder remains unchanged, due to which the transcoder is not removed from the voting system, allowing them to cast a vote with zero weight. This leads to arithmetic errors when delegators attempt to override the vote.

Let's illustrate the attack with a simplified scenario:

**Setup Phase**:

- Alice (transcoder): 100 LPT self-bonded + 1000 LPT from delegator Bob
- Charlie (voter): 500 LPT independent voting weight
- Active governance proposal under consideration

**Note**: Here LPT = Livepeer Token, the native token of the Livepeer protocol.

**Exploitation Sequence**:

1. **Alice Gets Fully Slashed**
    - `bondedAmount`: 100 → 0 (correctly reduced)
    - Removed from transcoder pool (correctly handled)
    - `delegateAddress` still points to Alice ❌ (bug: should be reset)
2. **Charlie Votes "For"** with 500 weight
    - Vote tallies: `forVotes = 500, againstVotes = 0`
3. **Alice Casts Phantom Vote**
    - Alice votes "For" but calculated weight = 0
    - System records vote but tallies remain unchanged
    - Vote tallies still: `forVotes = 500, againstVotes = 0`
4. **Bob Attempts Override** (voting "Against" to override Alice)
    - Bob's delegated weight: 1000 LPT
    - System attempts: `forVotes -= 1000` → `500 - 1000 = -500` ❌
    - **Transaction fails or corrupts vote accounting**

**Impact**: Bob is completely blocked from voting, despite having legitimate stake and voting rights.

## Attack Impact

- **Governance Disruption**: Slashed transcoders can systematically block their delegators from voting
- **Vote Manipulation**: Arithmetic errors can corrupt vote tallies by subtracting from incorrect categories
- **Democratic Failure**: Legitimate stakeholders lose their fundamental governance rights
- **Strategic Exploitation**: Malicious actors can vote on losing options to maximize disruption of delegator participation

## Real-World Exploitation Potential

The vulnerability becomes particularly dangerous when combined with strategic behavior:

- **Targeted Disenfranchisement**: Slashed transcoders can deny voting access to delegators who disagree with them
- **Vote Suppression**: By voting on unpopular options, attackers maximize the number of delegators who attempt overrides and face transaction failures
- **Systemic Risk**: Multiple exploitations could significantly skew governance outcomes by excluding legitimate voices

## Mitigation Steps

**Primary Solution: Reset Delegation State**:

```solidity
function slashTranscoder(...) external {
    // ... existing slashing logic ...
    
    if (del.bondedAmount == 0) {
        del.delegateAddress = address(0);  // Reset to unbonded state
    }
}
```

**Secondary Protection: Prevent Zero-Weight Votes**:

```solidity
function _countVote(...) internal {
    require(_weight > 0, "Cannot vote with zero weight");
    // ... rest of function
}
```

**Defense in Depth Approach**:

- Implement both solutions for comprehensive protection
- Add general zero-weight vote prevention to catch edge cases
- Include state validation checks in governance functions

## Pattern Recognition Notes

- **State Inconsistency Vulnerabilities**: Look for contracts where slashing/penalty logic updates some fields but not others, creating hybrid states
- **Governance Edge Cases**: Examine voting systems for scenarios where participants retain voting privileges despite losing economic stake
- **Arithmetic Assumptions**: Flag subtraction operations that assume previous additions without validating sufficient balance
- **Delegation State Management**: Verify that all state transitions properly update delegation relationships and voting eligibility

**Red Flags**:

- Functions that modify stake or participant status without updating all related state variables
- Voting systems that check delegation status but not economic stake for vote weight calculations
- Arithmetic operations on vote tallies without sufficient bounds checking
- Missing state resets in penalty or removal procedures

**Testing Approach**:

- Simulate full slashing scenarios and attempt subsequent voting operations
- Test delegator override functionality after various transcoder state changes
- Verify that all governance participants maintain consistent state across contract interactions
- Use property-based testing to explore edge cases in delegation and voting mechanics
