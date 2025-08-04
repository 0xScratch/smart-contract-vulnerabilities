# Incorrect Vote Deduction in Livepeer Governance System

* **Severity**: High
* **Source**: [Code4rena](https://github.com/code-423n4/2023-08-livepeer-findings/issues/96) / [One Bug Per Day](https://www.onebugperday.com/v1/480)
* **Affected Contract**: [GovernorCountingOverridable.sol](https://github.com/code-423n4/2023-08-livepeer/blob/a3d801fa4690119b6f96aeb5508e58d752bda5bc/contracts/treasury/GovernorCountingOverridable.sol#L174-L212)
* **Vulnerability Type**: Invalid Validation / Logic Bug

## Original Bug Description

>## Lines of code
>
>[https://github.com/code-423n4/2023-08-livepeer/blob/a3d801fa4690119b6f96aeb5508e58d752bda5bc/contracts/treasury/GovernorCountingOverridable.sol#L174-L212](https://github.com/code-423n4/2023-08-livepeer/blob/a3d801fa4690119b6f96aeb5508e58d752bda5bc/contracts/treasury/GovernorCountingOverridable.sol#L174-L212)
>
>## Vulnerability details
>
>## Impact
>
>A delegate can subtract their own voting weight from the voting choice of another delegate, even if that user isn't a transcoder. Since they are not a transcoder, they don't have their votes initially increased by the amount delegated to them, voting weight is still subtracted from the tally of their vote choice.
>
>Maliciously, this could be used to effectively double one's voting power, by delegating their votes to a delegator who is about to vote for the choice which they don't want. It can also occur accidentally, for example when somebody delegates to a transcoder who later switches role to delegate.
>
>## Proof of Concept
>
>When a user is not a transcoder, their votes are determined by the amount they have delegated to the delegatedAddress, and does not increase when a user delegates to them:
>
>```solidity
>        if (bond.bondedAmount == 0) {
>            amount = 0;
>        } else if (isTranscoder) {
>            amount = bond.delegatedAmount;
>        } else {
>            amount = delegatorCumulativeStakeAt(bond, _round);
>        }
>```
>
>Lets that this delegator (Alice) has 100 votes and votes `For`, Then another delegator(Bob) has delegated 1000 votes to Alice As stated above, Alice doesn't get the voting power of Bob's 1000 votes, so the `For` count increases by 100.
>
>Bob now votes, and `_handleVotesOverrides` is called. In this function, the first conditional, `if isTranscoder` will return false as Bob is not self-delegating.
>
>Then, there is a check that the address Bob has delegated to has voted. Note that there is a missing check of whether the delegate address is a transcoder. Therefore the logic inside `if (delegateVoter.hasVoted)` is executed:
>
>```solidity
>if (delegateVoter.hasVoted) {
>// this is a delegator overriding its delegated transcoder vote,
>// we need to update the current totals to move the weight of
>// the delegator vote to the right outcome.
>VoteType delegateSupport = delegateVoter.support;
>
>        if (delegateSupport == VoteType.Against) {
>            _tally.againstVotes -= _weight;
>        } else if (delegateSupport == VoteType.For) {
>            _tally.forVotes -= _weight;
>        } else {
>            assert(delegateSupport == VoteType.Abstain);
>            _tally.abstainVotes -= _weight;
>        }
>    }
>```
>
>The logic reduces the tally of whatever choice Alice voted for by Bob's weight. Alice initially voted `For` with 100 votes, and then the For votes is reduced by Bob's `1000 votes`. Lets say that Bob votes `Against`. This will result in an aggregate 900 vote reduction in the `For` tally and +1000 votes for `Against` after Alice and Bob has finished voting.
>
>If Alice was a transcoder, Bob will be simply reversing the votes they had delegated to Alice. However since Alice was a delegate, they never gained the voting power that was delegated to her.
>
>Bob has effectively gained the ability to vote against somebody else's votes (without first actually increasing their voting power since they are not a transcoder) and can vote themselves, which allows them to manipulate governance.
>
>## Tools Used
>
>Manual Review
>
>## Recommended Mitigation Steps
>
>There should be a check that a delegate is a transcoder before subtracting the tally. Here is some pseudocode:
>
>```solidity
>if (delegateVoter.hasVoted && ---delegate is transcoder ---)
>```
>
>This is an edit of the conditional of the function `_handleOverrides`. This ensures that the subtraction of vote tally is only performed when the delegate is a voter AND the delegate is a transcoder. This should fix the accounting/subtraction issue of vote tally for non-transcoder delegates.
>
>## Assessed type
>
>Invalid Validation

## Summary

The Livepeer governance system allows users to delegate their voting power to transcoders. However, when a user (delegator) overrides a vote that was previously delegated, the system blindly deducts vote weight from the delegate — even if that delegate is no longer an active transcoder or has never voted. This results in incorrect tallies and potential vote misaccounting, undermining the accuracy of governance outcomes.

## A Better Explanation (With Simplified Example)

To understand this bug in Livepeer, let's walk through it like a story — starting with how **delegation and voting** work in the protocol, and where things go wrong.

### 🧠 What's the Voting Model in Livepeer?

In Livepeer governance:

* Users can either **vote directly** on proposals, or
* **Delegate their voting power** to a **transcoder**, who votes on their behalf.

Each user has an associated `bond` which tracks:

* How much stake they've delegated
* To which address (a transcoder)

If the user **doesn't vote manually**, their vote is counted via the **transcoder's vote**, weighted by their delegated amount.

If the user **does vote manually**, the protocol:

1. **Overrides** the delegator's vote over their transcoder.
2. **Deducts** their weight from the transcoder's total (to prevent double-counting).
3. **Adds** their vote individually.

### 📦 So What's the Problem?

This all works great — *if the delegate is a transcoder*.

But there's a catch: **anyone** can receive delegation (not just transcoders). And if that "delegate" isn't a transcoder, they **don't gain any vote weight from those delegations**.

So what happens when a delegator **overrides** such a vote?

> 💥 The protocol still tries to *deduct* the delegator's weight from the **non-transcoder delegate** — even though that person's voting power wasn't affected in the first place.

That's a logic flaw: it incorrectly reduces the delegate's vote weight even though their weight was never boosted by that delegation.

### 🧪 Let's Break This With a Simple Example

#### Setup

* **Alice** has 100 LPT. She is *not* a transcoder.
* **Bob** has 200 LPT and **delegates** to Alice.
* **Alice votes "Yes"** manually.
* **Bob does not vote manually** — his vote follows Alice's (as per delegation logic).

#### Expected Behavior

* Alice's vote should count for **100 LPT**.
* Bob's vote should count for **200 LPT** (via Alice).
* Total "Yes" vote: **300 LPT**

Now, Bob changes his mind.

#### Bob Overrides His Vote

* Bob decides to vote **"No"** manually.
* The protocol says: "Ah, override detected! We must remove Bob's vote from Alice's total, and count it separately."

**Here's the problem:**

* Alice **never had Bob's vote weight** in the first place — she's not a transcoder.
* But the system **still deducts 200 LPT** from Alice's vote weight.

#### What Actually Happens

* Alice's vote is now incorrectly reduced to **100 - 200 = -100 LPT**
* Bob's vote is counted as 200 LPT for "No"
* Final vote tally:

  * Yes: **-100 LPT**
  * No: **200 LPT**

❌ This is completely wrong.

### 🔍 Why This Happens

This code snippet is where the issue starts:

```solidity
if (bond.bondedAmount == 0) {
    amount = 0;
} else if (isTranscoder) {
    amount = bond.delegatedAmount;
} else {
    amount = delegatorCumulativeStakeAt(bond, _round);
}
```

If the delegate is **not a transcoder**, `delegatedAmount` is *not added* to their vote weight.

But later on, in `_handleVoteOverrides`, the code *still deducts* `_weight` from the `delegateVoter` regardless of whether they ever received that `_weight` in the first place.

This creates an imbalance.

### 🎂 Real-Life Analogy: The Birthday Cake Problem

* You throw a party and order a cake for everyone.
* Each guest gives money to **Alice** to handle the order.
* Alice isn't the one baking or serving the cake (that's what transcoders do).
* Still, some guests (like Bob) change their mind and decide to get their own slice.
* The system says: "Okay, take Bob's slice back from Alice!"

But wait — Alice **never had** Bob's slice to begin with.

Still, the system reduces her share of the cake, even though she was only eating her own. Now, she gets less cake for no reason.

## Vulnerable Code Reference

```solidity
// Inside _handleVoteOverrides()
ProposalVoterState storage delegateVoter = _tally.voters[delegate];
delegateVoter.deductions += _weight;

if (delegateVoter.hasVoted) {
    VoteType delegateSupport = delegateVoter.support;
    _subtractSupport(tally, delegateSupport, _weight);
}
```

### Problem

* `_tally.voters[delegate]` is mutated to include a deduction **before checking** if the delegate has even voted.
* Deductions are applied regardless of whether:

  * The delegate has cast a vote.
  * The delegate is a valid transcoder anymore.

## Recommended Mitigation

1. **Add validation checks** before applying deductions:

   * Ensure the delegate is still a registered transcoder.
   * Ensure the delegate has actually cast a vote for this proposal.

2. **Apply deductions conditionally**:

   * Only subtract vote weight from delegate's support outcome if `delegateVoter.hasVoted` is `true`.
   * Otherwise, skip deduction logic entirely.

3. **Emit events** when deductions are applied to improve traceability during audits or debugging.

## Pattern Recognition Notes

This vulnerability stems from improper assumptions during vote overrides in delegation-based systems — especially when vote deductions are applied without validating the delegate's eligibility.

Here are core patterns to watch for:

1. **Delegate Eligibility Checks Are Missing**
    * Ensure that vote deductions only apply if the delegate (e.g., transcoder) was eligible to receive vote weight during the snapshot. Don't assume every delegate is a valid vote-carrier.

2. **Invalid Deduction on Ineligible Delegates**
    * If a delegate didn't contribute to the original vote tally (e.g., not a transcoder), applying a deduction on override leads to undercounting — or even negative tallies.

3. **Role Transitions Before Snapshot**
    * A user might become ineligible (e.g., step down as a transcoder) after receiving delegated weight. Systems should check role status at snapshot time, not current state.

4. **Mismatch in Snapshot Timing**
    * Vote weight and delegate role should both be determined using the same snapshot round. Mismatches can introduce logical inconsistencies.

5. **Missing Edge-Case Tests**

    Always test scenarios where:
    * Delegator overrides vote from an invalid or inactive delegate.
    * Delegate voted, then lost eligibility.
    * Multiple delegators delegate to an inactive address.

6. **Separate Delegation vs. Voting Logic**
    * Avoid combining vote casting and delegation logic into one flow. Treat them distinctly to prevent unintentional interactions.
