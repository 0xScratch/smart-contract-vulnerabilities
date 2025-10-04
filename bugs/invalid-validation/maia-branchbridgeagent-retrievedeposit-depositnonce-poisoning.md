# Cross‑chain DepositNonce Poisoning — `retrieveDeposit()` allows arbitrary nonces to be marked executed

* **Severity**: High
* **Source**: [Code4rena](https://github.com/code-423n4/2023-05-maia-findings/issues/688) / [One Bug Per Day](https://www.onebugperday.com/v1/134)
* **Affected Contract**: [BranchBridgeAgent.sol](https://github.com/code-423n4/2023-05-maia/blob/main/src/ulysses-omnichain/BranchBridgeAgent.sol)
* **Vulnerability Type**: Invalid Validation / Cross‑chain State Poisoning / Funds Lock (Denial of Service leading to funds loss)

## Summary

`BranchBridgeAgent.retrieveDeposit(uint32 _depositNonce)` allows *anyone* to craft and send a cross‑chain message for an arbitrary `_depositNonce` without verifying that the deposit exists or that the caller owns it. When the message arrives at the root agent, `RootBridgeAgent.anyExecute` toggles `executionHistory[fromChainId][_depositNonce] = true` for that nonce. If a legitimate user later receives the same nonce for a real deposit, the root will ignore it (already marked executed) while the branch has locked the tokens — leaving the user's funds stuck on the branch with no settlement on root.

## A Better Explanation (With Simplified Example)

### Intended Behavior

1. **User deposit (branch)**: When a user deposits tokens (e.g., via `callOutAndBridge`), the branch assigns a unique `depositNonce`, records the deposit metadata, and sends a cross‑chain execution request to the root so the root can process/mint/credit on the remote side.
2. **Root processing**: Root processes the request for that `depositNonce` and marks it executed only after validating/handling the real deposit message, ensuring branch and root state stay in sync.

### What Actually Happens (Bug)

* `retrieveDeposit()` constructs a packed message (flag `0x08`) and sends it through AnyCall **without checking** that `_depositNonce` exists in `getDeposit` or that `msg.sender` is the deposit owner.
* `RootBridgeAgent.anyExecute()` processes the `0x08` (retrieveDeposit) flag and, if not previously marked, sets `executionHistory[fromChainId][nonce] = true` unconditionally for the supplied nonce. This marks the nonce as executed on the root.
* Later, a real deposit assigned the same nonce (by the branch) will be **ignored** on root because `executionHistory` already records it as done — while the branch has already collected/locked the user's tokens. No fallback is triggered to return funds to the user. Result: **user funds locked on branch / not reflected on root**.

### Why This Matters

* **Permanent user funds loss / effective theft**: The branch holds the tokens; root will not release or mint corresponding tokens because it thinks the nonce was already executed.
* **Cross‑chain desynchronization**: Root and branch no longer agree on which nonces are pending. Cross‑chain systems rely heavily on nonce integrity — breaking that invites permanent state divergence.
* **Low attacker cost**: An attacker need not hold any tokens or privileges — simply call `retrieveDeposit()` with a future nonce.

## Concrete Walkthrough (Mallory & Alice)

* **Setup**: Current global `depositNonce` on branch = `50`. No deposit `60` exists yet.
* **Mallory (attacker)**: Calls `retrieveDeposit(60)` on the branch agent. The branch sends an AnyCall message corresponding to flag `0x08` and nonce `60`.
* **Root**: `RootBridgeAgent.anyExecute()` receives the message and executes the `0x08` branch: it sets `executionHistory[fromChainId][60] = true` (marks nonce 60 as executed) and returns.
* **Alice (victim)**: Later makes a legitimate deposit which the branch assigns `depositNonce = 60`. Branch collects Alice's tokens and sends the normal deposit message to root.
* **Root**: Sees `executionHistory[fromChainId][60] == true` and short‑circuits or refuses to process (treats it as already executed). No mint/release happens.
* **Outcome**: Alice's tokens are locked on the branch and she cannot claim them — the operation has been poisoned by Mallory.

## Vulnerable Code Reference

### 1) BranchBridgeAgent.retrieveDeposit (vulnerable snippet)

```solidity
function retrieveDeposit(
    uint32 _depositNonce
) external payable lock requiresFallbackGas {
    //Encode Data for cross-chain call.
    bytes memory packedData = abi.encodePacked(
        bytes1(0x08),
        _depositNonce,
        msg.value.toUint128(),
        uint128(0)
    );

    //Update State and Perform Call
    _sendRetrieveOrRetry(packedData);
}
```

*Problem*: No `require` that: the deposit exists, or that `msg.sender == getDeposit[_depositNonce].owner`, or that `_depositNonce` is within a safe range.

### 2) RootBridgeAgent.anyExecute (relevant behavior)

```solidity
// DEPOSIT FLAG: 8 (retrieveDeposit)
else if (flag == 0x08) {
    //Get nonce
    uint32 nonce = uint32(bytes4(data[1:5]));

    //Check if tx has already been executed
    if (!executionHistory[fromChainId][nonce]) {
        //Toggle Nonce as executed
        executionHistory[fromChainId][nonce] = true;

        //Retry failed fallback
        (success, result) = (false, "");
    } else {
        _forceRevert();
        //Return true to avoid triggering anyFallback in case of `_forceRevert()` failure
        return (true, "already executed tx");
    }
}
```

*Problem*: Root sets `executionHistory[fromChainId][nonce] = true` for the arbitrary nonce sent by the branch agent. There is no proof or validation that the branch actually intended to process a real deposit for that nonce.

## Recommended Mitigation

### 1) Primary: Validate deposit ownership & existence in `retrieveDeposit()` (branch)

Reject calls that attempt to retrieve a nonce that does not exist or that the caller does not own:

```solidity
require(getDeposit[_depositNonce].owner != address(0, "Invalid deposit nonce");
require(msg.sender == getDeposit[_depositNonce].owner, "Not deposit owner");
```

This ensures only the deposit owner (or a privileged actor) can request the retrieve action for an existing deposit.

### 2) Defensive: Ensure branch only emits retrieve/retry messages for pending, recorded nonces

* Enforce a check that the `_depositNonce` belongs to the branch's local pending deposit set (e.g., `depositNonce <= currentGlobalNonce && getDeposit[nonce].status == PENDING`).
* Reject arbitrary future nonces.

### 3) Root‑side hardening

* Root should avoid blindly toggling `executionHistory` to `true` without verifying the origin/intent: for example, include proof data in the message (an authenticated proof or a branch signature / Merkle proof) that this nonce corresponds to an actual recorded deposit. If root cannot validate, it should refuse the marking.
* Consider moving to a model where root only marks execution after successful processing of the deposit, or at least marks as executed with evidence.

### 4) Observability & Recovery

* Emit events on `retrieveDeposit()` attempts including `_depositNonce` and `msg.sender` so operators can detect suspicious pre‑marking activity.
* Add an admin/emergency mechanism to clear or reprocess poisoned nonces if the contract supports upgrades or governance intervention.

### 5) Tests & Fuzzing

* Unit tests that call `retrieveDeposit()` with non‑existent/future nonces must revert.
* Property tests: ensure that for any `n`, a message cannot mark `executionHistory[fromChainId][n]` unless `getDeposit[n]` existed and was owned by the caller at the time of invocation.

## Pattern Recognition Notes

* **Sequence‑number poisoning**: Systems that use monotonically issued nonces or serial counters are vulnerable if an unprivileged party can mark future/unused numbers as consumed.
* **Authentication missing on cross‑chain intent**: Cross‑chain messages that change critical bookkeeping must include proofs or be gated by local invariant checks. Blind toggling of state across chains is dangerous.
* **Sentinel conflation**: Using the absence of a flag or `executionHistory == false` as the only proof that something is pending is fragile — the system must validate intent and ownership explicitly.
* **Fail‑open vs. fail‑safe**: Setting `executionHistory = true` preemptively is effectively failing open for the root's state machine and can be weaponized; prefer conservatively refusing unknown or unverifiable actions.

### Quick Recall (TL;DR)

* **Bug**: `retrieveDeposit()` lets anyone send a cross‑chain message for any nonce (including future nonces) without verifying existence or ownership. Root then marks that nonce as executed.
* **Impact**: When a later legitimate deposit receives that nonce, the root ignores it and the branch holds funds — user funds are stuck / effectively lost.
* **Fix**: Require `msg.sender == getDeposit[_depositNonce].owner` and `getDeposit[_depositNonce].owner != address(0)` in `retrieveDeposit()`; add root‑side validation or proofing; add tests and observability.
