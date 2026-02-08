# Incorrect DISPUTED_L2_BLOCK_NUMBER Causes Cross-Game Context Collisions & Invalid VM Outcomes

* **Severity**: High
* **Source**: [Code4rena](https://github.com/code-423n4/2024-07-optimism-findings/issues/36)
* **Affected Contract**: [FaultDisputeGame.sol](https://github.com/code-423n4/2024-07-optimism/blob/70556044e5e080930f686c4e5acde420104bb2c4/packages/contracts-bedrock/src/dispute/FaultDisputeGame.sol#L453)
* **Vulnerability Type**: Incorrect State Context / Fault Proof Misdirection / Cross-Game Inconsistency

## Summary

The Fault Dispute Game incorrectly computes the **DISPUTED_L2_BLOCK_NUMBER** passed to the Fault Proof VM. Instead of being bounded by the **claimed L2 block number** of the dispute game, the contract unconditionally computes:

```solidity
startingOutputRoot.l2BlockNumber + traceIndex + 1
```

This value can exceed the disputed claim's actual block height.

**Impact:**
This causes the VM to execute **beyond the claim's intended scope**, sometimes all the way to the *safe head*. As a result:

* A dispute about block `k` can end up validating the correctness of block `k+1` instead.
* Two separate dispute games (for block `k` and for block `k+1`) can produce **identical VM inputs**, breaking determinism.
* A dishonest prover can win against a correct claim by "borrowing" the correctness of later safe blocks.
* The intended "proof about *this* block number" becomes a proof about a **different block**.

The issue becomes exploitable when combined with move-ordering rules in execution-trace bisection (status-byte requirement + no-defense-at-root constraints). With these aligned, a dishonest actor can force the game to evaluate transitions beyond the claim (e.g., `k → k+1`) and defeat an otherwise valid claim.

## A Better Explanation (With Simplified Example)

### Intended Behavior

The VM should only evaluate execution transitions **up to the block being disputed**, not past it.

For a game disputing block `k`:

* The execution trace bisection should eventually evaluate:
  **Transition: block k−1 → block k**
* It should never evaluate:
  **Transition: block k → block k+1**
  …because block `k+1` is not part of the claim.

The VM must stay within the **logical bounds of the proposal**.

### What Actually Happens (Bug)

The code computes:

```solidity
uint256 l2Number =
    startingOutputRoot.l2BlockNumber
    + disputedPos.traceIndex(SPLIT_DEPTH)
    + 1;
```

This **ignores the claimed block height**, so:

* If the bottom-half trace index is large enough,
* Or if the starting block is low enough,

then:

```solidity
DISPUTED_L2_BLOCK_NUMBER > claimed L2 block
```

This tricks the VM into validating blocks **past** the claim.

## Why This Matters (And Creates Real Contradictions)

The VM is deterministic.
If two games feed it the **same (startRoot, disputedRoot, l1Head, blockNumber)** tuple, it must return the same result.

This bug makes it possible for:

* **Game A** (claim for block `k`) and
* **Game B** (claim for block `k+1`)

to feed the **exact same VM context**, even though they are disputes about different blocks.

This breaks the fundamental security invariant: **each Fault Dispute Game must be self-contained and contextually isolated**.

Worse, if block `k+1` is *already safe*, then a dishonest actor can intentionally:

1. Steer both games to evaluate the same transition (`k → k+1`).
2. Force themselves to make the first move into the execution subgame (so they can post an *attack root* with status byte INVALID/PANIC).
3. Drive the game to `step()` such that the VM reports that transition as VALID.
4. Use that VALID outcome to counter the honest claim at block `k`.

Thus, a completely valid output for block `k` can be attacked and defeated using the correctness of block `k+1`.

## Concrete Walkthrough (Block-k / Block-k+1 Example)

Assume:

* `startingOutputBlock = 0`
* Honest proposer Alice claims block **2** (`B2`)
* Bob claims block **3** (`B3`)
* Block **3** is already safe at the L1 head
* The split-depth span allows reaching trace index `2 → 3` in execution bisection

### What happens

1. Both Alice's and Bob's games can steer to the execution leaf that evaluates
   **B2 →₃ B3**

2. Due to the bug, both games compute:
   `DISPUTED_L2_BLOCK_NUMBER = starting + traceIndex + 1 = 3`

3. Thus the VM sees identical input for both games.

4. Bob can arrange move ordering such that:

   * He posts the **root attack** of the execution subgame (status byte = INVALID, as required).
   * Alice is forced to defend deeper, leading to a final `step()` call.

5. But the VM now evaluates **block 2 → block 3**, and since block 3 is safe, the op-program returns **VALID**:

   * In Bob's game: good (as expected)
   * In Alice's game: *bad* — the leaf is out of scope (should have only evaluated 1→2)

6. This VALID result **counters Alice's correct claim**, even though Bob's attack was semantically irrelevant to block 2.

### Result

A correct claim for block `k` can be defeated by pointing to the correctness of block `k+1`.

This is the exact incorrect game resolution the judge highlighted.

## Vulnerable Code Reference

### 1) DISPUTED_L2_BLOCK_NUMBER ignores claimed block height

```solidity
// FaultDisputeGame.addLocalData
uint256 l2Number = startingOutputRoot.l2BlockNumber
    + disputedPos.traceIndex(SPLIT_DEPTH)
    + 1;

oracle.loadLocalData(
    _ident,
    uuid.raw(),
    bytes32(l2Number << 0xC0),
    8,
    _partOffset
);
```

There is **no bounding** by:

```solidity
l2BlockNumber()   // the claimed block number for the dispute game
```

### 2) As a result, VM inter-block execution never stops at the disputed block

It continues until it reaches the L2 safe head, which may validate blocks beyond the proposal.

## Recommended Mitigation

### 1. Cap the block number fed to the VM

```solidity
uint256 computed =
    startingOutputRoot.l2BlockNumber +
    disputedPos.traceIndex(SPLIT_DEPTH) +
    1;

uint256 l2Number = (computed < l2BlockNumber())
    ? computed
    : l2BlockNumber(); // cap to claimed block

oracle.loadLocalData(
    _ident,
    uuid.raw(),
    bytes32(l2Number << 0xC0),
    8,
    _partOffset
);
```

### 2. (Optional but recommended)

Include `l2BlockNumber()` in the **local context UUID** so different games cannot collide on VM context hashes.

```solidity
uuid = keccak256(abi.encode(
    starting, startingPos,
    disputed, disputedPos,
    l2BlockNumber()  // new
));
```

### 3. Add tests ensuring VM evaluation stops at claimed block

This should be a required invariant.

## Pattern Recognition Notes

* **Context Bounding Failures**
  In fraud/fault proofs, "what block are we proving?" is part of the logical context. Any mismatch creates a new execution universe with different semantics.

* **Cross-Game State Collisions**
  When two separate dispute games feed the VM identical input, determinism guarantees identical behavior, even when the games *should* differ.

* **Out-of-Scope Evaluation**
  Evaluating an inter-block transition beyond a claim's height turns a well-scoped dispute into a proof of a larger prefix, causing incorrect accept/reject outcomes.

* **Move-Order Sensitivity**
  Execution-bisection rules (no-defense-at-root, status-byte parity) can convert a "context bug" into a **winning strategy**, not just a correctness mismatch.

* **Invalid Root, Valid Leaf**
  A forced INVALID status byte at the execution subgame root does not prevent winning deeper if the final leaf `step()` evaluates a VALID transition. This inversion is only possible when evaluating the wrong block boundary.

### Quick Recall (TL;DR)

* **Bug:** `DISPUTED_L2_BLOCK_NUMBER` is not capped; VM may evaluate blocks beyond the claimed L2 block.
* **Impact:** A dishonest prover can defeat a correct claim by exploiting correctness of later safe blocks; cross-game inputs collide.
* **Fix:** Cap at `min(computed, claimedBlock)` and include claimed block in local-context hashing.

If you'd like, I can also generate a short "diagram version" that visually illustrates how both games collapse into the same VM context.
