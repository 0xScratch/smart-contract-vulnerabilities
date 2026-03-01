# Kuiper — Reentrancy-Driven Basket Drain via `settleAuction`

* **Severity**: High
* **Source**: [Code4rena](https://github.com/code-423n4/2021-09-defiprotocol-findings/issues/223)
* **Affected Contract**: [Auction.sol](https://github.com/code-423n4/2021-09-defiProtocol/blob/main/contracts/contracts/Auction.sol)
* **Vulnerability Type**: Reentrancy / State Desynchronization / Economic Drain

## Summary

`Auction.settleAuction()` performs external token transfers **before finalizing critical state** (notably `ibRatio` updates) and lacks any reentrancy protection. Because the function accepts **arbitrary ERC20 input tokens**, an attacker can supply a malicious token whose `transferFrom` re-enters `settleAuction()` during execution.

Each nested call updates the Basket's `ibRatio` to a progressively lower value (by design of the Dutch auction). The safety check that is supposed to ensure sufficient reserves depends directly on this ratio. As the ratio decreases during reentrancy, the required remaining basket balance becomes smaller, allowing the attacker to withdraw increasing amounts of legitimate basket assets (e.g., ETH, USDC) within a single transaction.

By recursively exploiting this mechanism, the attacker can **drain the basket's funds**.

## A Better Explanation (With Simplified Example)

### Intended Behavior

The rebalance auction is meant to work like this:

1. Basket starts an auction.
2. A bonder commits collateral.
3. Bonder calls `settleAuction()` once to perform the rebalance.
4. Function computes a time-decayed `newRatio`.
5. Contract verifies the basket still holds enough assets.
6. Basket updates weights and ratio.
7. Auction ends.

**Key assumption:** `settleAuction()` executes atomically exactly once.

### What Actually Happens (Bug)

Because the function:

* accepts arbitrary ERC20 tokens,
* performs external calls mid-execution,
* and lacks a reentrancy guard,

an attacker can interrupt the function and run it again **before the outer call finishes**.

Each inner execution lowers the basket's `ibRatio`, which weakens the reserve safety check used by the outer execution. This allows progressively larger withdrawals from the basket.

### Why the Malicious Token Matters

The attacker's token is **not the asset being stolen**.

Instead, it is a booby-trapped ERC20 whose `transferFrom()` function calls back into `Auction.settleAuction()`.

Think of it as:

> 🔓 The crowbar that forces the door open repeatedly.

The actual stolen funds are the **legitimate assets held by the Basket**.

### Concrete Walkthrough (Alice & Mallory)

#### Initial State

Basket holds:

* 100 ETH
* 200,000 USDC
* `ibRatio = 1.0e18`

Auction has approval to move basket assets.

#### Step 1 — Mallory Bonds

Mallory becomes the auction bonder via:

```solidity
bondForRebalance()
```

She now controls settlement.

#### Step 2 — Mallory Calls `settleAuction`

She supplies:

* `inputTokens = [evilToken]`
* `outputTokens = [ETH]`

The malicious token's `transferFrom` triggers reentrancy.

#### Step 3 — Inner Reentrant Call

Inner execution computes:

```solidity
newRatio ≈ 0.9e18
```

Safety requirement becomes:

```solidity
tokensNeeded = 90 ETH
```

Basket has 100 ETH → passes.

Mallory withdraws **10 ETH**.

Inner call finishes and updates:

```solidity
ibRatio = 0.9e18
```

#### Step 4 — Outer Call Resumes

Now the outer call recomputes using the **already lowered** ratio:

```solidity
newRatio ≈ 0.7e18
tokensNeeded = 70 ETH
```

Basket currently has 90 ETH.

Mallory can now withdraw **20 more ETH**.

#### Step 5 — Recursive Drain

By nesting deeper calls, the ratio keeps dropping:

| Iteration | ibRatio | tokensNeeded | Basket Drainable |
| --------- | ------- | ------------ | ---------------- |
| Start     | 1.0     | 100          | 0                |
| Inner     | 0.9     | 90           | 10               |
| Next      | 0.7     | 70           | +20              |
| Next      | 0.3     | 30           | +40              |

Eventually the basket can be nearly emptied.

### Why This Matters

* The safety invariant depends on a mutable value (`ibRatio`).
* That value is updated during reentrant execution.
* The function assumes single-shot execution but is externally interruptible.
* Basket pre-approval gives Auction full token pull power.

Result: **economic drain of basket reserves**.

## Vulnerable Code Reference

### 1) External call before critical state finalization

```solidity
for (uint256 i = 0; i < inputTokens.length; i++) {
    IERC20(inputTokens[i]).safeTransferFrom(
        msg.sender,
        address(basket),
        inputWeights[i]
    );
}
```

This allows arbitrary token callbacks.

### 2) Basket assets transferred out during the same execution

```solidity
for (uint256 i = 0; i < outputTokens.length; i++) {
    IERC20(outputTokens[i]).safeTransferFrom(
        address(basket),
        msg.sender,
        outputWeights[i]
    );
}
```

Auction is trusted to pull real assets from the basket.

### 3) Ratio updated only at the end (too late)

```solidity
basket.setNewWeights();
basket.updateIBRatio(newRatio);
```

During reentrancy, inner calls update this value before the outer call finishes.

### 4) Safety check depends on mutable ratio

```solidity
uint256 tokensNeeded =
    basketAsERC20.totalSupply()
    * pendingWeights[i]
    * newRatio
    / BASE
    / BASE;

require(
    IERC20(pendingTokens[i]).balanceOf(address(basket)) >= tokensNeeded
);
```

As `newRatio` decreases, the required reserves shrink.

## Recommended Mitigation

### 1. Add reentrancy protection (primary fix)

```solidity
import "@openzeppelin/contracts/security/ReentrancyGuard.sol";

contract Auction is ReentrancyGuard {
    function settleAuction(...) external nonReentrant {
        ...
    }
}
```

This alone blocks the recursive attack.

### 2. Follow Checks-Effects-Interactions

Update critical state **before** external token transfers where possible.

### 3. Restrict accepted tokens

If design permits, validate `inputTokens` against the expected pending token set to avoid arbitrary callback surfaces.

### 4. Defense-in-depth

* Consider pull-based output transfers.
* Add invariant tests ensuring basket value cannot decrease beyond bounds in a single settlement.
* Consider rate-limiting or single-execution guards per auction.

## Pattern Recognition Notes

* **Reentrancy via Arbitrary ERC20**: Any time user-supplied tokens are transferred, assume callback risk.
* **State Updated Too Late**: Critical accounting variables changed after external calls are prime reentrancy targets.
* **Time-Decay + Reentrancy = Dangerous**: Dutch auctions and decay formulas amplify recursive attacks.
* **Trusted Allowance Abuse**: When a vault pre-approves a helper contract, any logic bug becomes a direct drain vector.
* **Atomicity Assumptions**: If correctness depends on "this function runs once," enforce it explicitly.

## Quick Recall (TL;DR)

* **Bug**: `settleAuction()` is reentrant and uses a ratio that decreases each nested call.
* **Impact**: Safety check weakens mid-execution → attacker can repeatedly withdraw basket assets.
* **Root Cause**: External calls before final state updates + no reentrancy guard.
* **Fix**: Add `nonReentrant` and follow CEI; optionally restrict input tokens.

