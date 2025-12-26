# Loss of Bond Amounts on Re-org Attacks in Fault Dispute Game

* **Severity**: Medium
* **Source**: [Sherlock Audits](https://github.com/sherlock-audit/2024-02-optimism-2024-judging/issues/201/#issuecomment-2075436915)
* **Affected Contract**: [`FaultDisputeGame.sol`](https://github.com/sherlock-audit/2024-02-optimism-2024/blob/main/optimism/packages/contracts-bedrock/src/dispute/FaultDisputeGame.sol)
* **Vulnerability Type**: Blockchain Re-org Exploit / State Inconsistency / Bond Theft

## Summary

The `move()` function in Optimism's Fault Dispute Game relies solely on a `parentIndex` to identify the target claim for challenges, without verifying the underlying claim data. This allows Ethereum block re-orgs to alter the claim at that index between transaction submission and execution, enabling attackers to replace invalid claims with valid ones. Honest participants' moves then apply to the wrong claim, resulting in invalid actions and loss of their posted bonds.

## A Better Explanation (With Simplified Example)

### Intended Behavior

1. **Claim Submission**: A participant makes a claim about the L2 state (e.g., disputing an output root) and posts a bond.
2. **Challenge Moves**: Others respond via `attack()`, `defend()`, or `move()`, specifying a `parentIndex` (the ID of the claim they're targeting) and a committed `_claim` (a hash of their counter-argument).
   * The contract looks up the claim at `claims[parentIndex]` and applies the move if valid.
   * Bonds are escrowed, and winners claim losers' bonds after resolution.
3. **Game Progression**: Moves build a game tree, with bonds escalating in deeper levels. The first successful challenger often gets a reward, incentivizing quick responses.

### What Actually Happens (Bug)

* The `move()` function does **not** require or verify the expected claim hash or position for the `parentIndex`.
* A re-org can reshuffle recent claims, changing what `parentIndex` points to.
* Participants' pending TXs (with commitments tied to the original claim) execute on the altered state, making moves invalid (e.g., challenging a valid claim as invalid).
* No safeguards like waiting for finality are documented, and racing for rewards discourages delays.

### Why This Matters

* Honest participants lose bonds without fault, eroding trust in the dispute system.
* Re-orgs are frequent on Ethereum (multiple daily), making this practical—not theoretical.
* Deep-game bonds can be substantial (e.g., thousands of ETH equivalents), amplifying losses.
* Undermines Optimism's fault proof security, as fewer honest challengers may participate due to risk.

### Concrete Walkthrough (Alice & Mallory)

* **Setup**: A dispute game is active with claims stored in `claims[]`.
* **Mallory Attack**: Mallory submits an invalid claim TX, which gets mined (now at some `parentIndex`).
* **Alice (Honest Defender) Responds**: Alice sees the invalid claim, computes a disproving `_claim` commitment, and submits a `move()` TX targeting that `parentIndex`. Multiple Alices might race for the reward.
* **Re-org Occurs**: Before Alice's TX mines, a natural re-org orphans the block with Mallory's invalid claim, dropping it back to the mempool.
* **Mallory Replaces**: Mallory quickly broadcasts a replacement TX (same nonce, higher gas) with a valid claim, which mines on the new chain at the same `parentIndex`.
* **Alice's TX Executes**: Now on the re-orged chain, Alice's move targets the valid claim but her commitment was for the invalid one—it's mismatched and invalid.
* **Outcome**: Alice loses her bond. Mallory counters the invalid move and claims it. If multiple defenders rushed in, Mallory scoops all their bonds.

> **Analogy**: Imagine mailing a chess move to "Player #5" in a tournament bracket. En route, the bracket reshuffles (re-org), and #5 becomes a different opponent. Your move arrives but doesn't make sense against the new player, so you forfeit your stake. The attacker, knowing the shuffle, swaps their weak piece out for a strong one just in time.

## Vulnerable Code Reference

1. **Missing claim verification in `move()`**

    ```solidity
        /// @notice Generic move function for attacks and defenses.
        /// @param _challengeIndex The index of the claim being moved against.
        /// @param _claim The claim at the next Gindex in the trace.
        /// @param _isAttack Whether or not the move is an attack.
        function move(uint256 _challengeIndex, Claim _claim, bool _isAttack) public payable {
            // ... (bond handling and other logic)
            // Fetch the parent claim
            ClaimData storage parent = claims[_challengeIndex];
            // No verification that parent.claim matches an expected hash or position
            // Proceeds with move application, potentially on wrong claim after re-org
            // ...
        }
    ```

2. **Attack and defend wrappers rely on the same unverified index**

    ```solidity
    /// @notice Attacks a claim with a move.
    /// @param _disputed The disputed claim
    /// @param _attackClaim The attacking claim
    function attack(Claim _disputed, Claim _attackClaim) external payable {
        move(_disputed.position.indexAtDepth(), _attackClaim, true);
    }
    /// @notice Defends a claim with a move.
    /// @param _disputed The disputed claim
    /// @param _defendClaim The defending claim
    function defend(Claim _disputed, Claim _defendClaim) external payable {
        move(_disputed.position.indexAtDepth(), _defendClaim, false);
    }
    ```

3. **Claim storage allows re-org shifts without move safeguards**:

    ```solidity
    // Claims are stored in an array, indexed by position
    ClaimData[] public claims;
    // Claims can be added/altered via re-orgs affecting recent blocks
    // Moves reference by index only, no hash/position check
    ```

## Recommended Mitigation

1. **Require explicit claim parameters in moves** (primary fix)

    ```solidity
    // Updated move signature
    function move(uint256 _challengeIndex, Claim _claim, bool _isAttack, Hash _expectedParentClaim, Position _expectedPosition) external payable {
        ClaimData storage parent = claims[_challengeIndex];
        if (parent.claim != _expectedParentClaim || parent.position != _expectedPosition) {
            revert InvalidParent();
        }
        // Proceed with move
    }
    ```

2. **Document re-org risks**: Add warnings in code/docs for users to wait for sufficient confirmations in high-stakes disputes.
3. **Enhance off-chain tools**: Client libraries could compute and include expected claim data automatically.
4. **Add tests for re-org scenarios**: Include fuzzing/property tests simulating re-orgs and claim replacements.

## Pattern Recognition Notes

* **Index Fragility in Volatile State**: Relying on array indices for references fails in re-org-prone environments; always pair with immutable identifiers like hashes.
* **Timing Attacks via Mempool/Replacement**: Assumes TX execution context matches submission; vulnerable to RBF (Replace-By-Fee) during forks.
* **Incentive Misalignment**: Reward for first mover encourages rushing without safety checks, amplifying exploits.
* **Boundary Validation at Execution**: Verify state assumptions (e.g., parent claim) at runtime to handle blockchain volatility.
* **Recoverability**: Design moves to revert safely on mismatch rather than penalize, or provide appeal mechanisms for re-org victims.

## Quick Recall (TL;DR)

* **Bug**: `move()` uses unverified `parentIndex`; re-orgs can swap claims, misapplying honest moves.
* **Impact**: Honest participants lose bonds to attackers via claim replacement during re-orgs.
* **Fix**: Require expected claim hash and position in `move()`; check against current state; document re-org waits.
