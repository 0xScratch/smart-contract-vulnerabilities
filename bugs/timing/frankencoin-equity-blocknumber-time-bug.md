# Inaccurate Holding Duration on Optimism Due to `block.number` Usage in `Equity.sol`

* **Severity**: Medium
* **Source**: [Code4rena](https://github.com/code-423n4/2023-04-frankencoin-findings/issues/914) / [One Bug Per Day](https://www.onebugperday.com/v1/35)
* **Affected Contract**: [Equity.sol](https://github.com/code-423n4/2023-04-frankencoin/blob/main/contracts/Equity.sol)
* **Vulnerability Type**: Time Measurement / Cross-Chain Compatibility

## Summary

The Frankencoin `Equity` contract uses `block.number` to approximate elapsed time in its `anchorTime()` function. On Ethereum mainnet, this provides a rough correlation to real-time since blocks are produced at \~12s intervals.

However, when deployed on **Optimism (or other L2s)**, block numbers are not tied to wall-clock time. Optimism produces one block per transaction with irregular spacing, so elapsed block numbers do not equal elapsed seconds or days. This causes the computation of the **minimum 90-day holding duration** for equity tokens to be inaccurate, potentially allowing early redemption or delaying redemption longer than intended.

## A Better Explanation

### Intended Behavior

* Users must hold equity tokens for a minimum of **90 days** before redeeming.
* The system enforces this by recording an **anchor time** when tokens are acquired and comparing it with the current anchor time at redemption.
* `anchorTime()` is expected to provide a reliable proxy for elapsed time.

### What Actually Happens (Bug)

* `anchorTime()` returns `block.number << BLOCK_TIME_RESOLUTION_BITS`.
* On Ethereum L1, this roughly scales with elapsed time (blocks \~12s each).
* On Optimism, `block.number` increments per transaction and does **not map to real time**.
* As a result, the "90-day minimum" check becomes unreliable:

  * Users may redeem **too early** if block numbers accumulate quickly.
  * Users may be blocked **too long** if block production slows.

### Why This Matters

* **Governance guarantees are weakened**: Holding duration for votes or redemptions may not align with intended 90 days.
* **Cross-chain deployments are fragile**: Contracts assuming mainnet block behavior break when deployed to rollups with different block semantics.
* **Predictability is lost**: Token holders cannot reliably know when they become eligible for redemption.

## Vulnerable Code Reference

### 1) Inaccurate time calculation in `anchorTime()`

```solidity
function anchorTime() internal view returns (uint64){
    return uint64(block.number << BLOCK_TIME_RESOLUTION_BITS);
}
```

### 2) Used in enforcing 90-day holding period (comments at lines 54-59)

```solidity
// Equity tokens must be held for at least 90 days before redeem()
```

## Recommended Mitigation

1. **Use `block.timestamp` instead of `block.number`** for all time-based conditions.

    ```solidity
    function anchorTime() internal view returns (uint64){
        return uint64(block.timestamp);
    }
    ```

2. **Explicitly document assumptions**: If contracts are designed for Ethereum mainnet only, state that clearly. Otherwise, design for cross-chain environments from the start.

3. **Add invariant tests**: Unit tests should verify that a 90-day duration corresponds to 90 days of wall-clock time on different target networks.

## Pattern Recognition Notes

* **Block.number misuse**: Using block height as a proxy for time is fragile; block.timestamp is the canonical time source in EVM.
* **Cross-Chain Deployment Pitfall**: Protocols assuming L1 behavior may silently break when deployed to L2s with different consensus and block production models.
* **Time-based Governance Vulnerability**: Inaccurate time tracking undermines lockups, vesting, voting periods, and redemption windows.
* **Resolution Strategy**: Always validate which chain a protocol will target and pick the correct mechanism (`block.timestamp` for time, `block.number` only for ordering).

### Quick Recall (TL;DR)

* **Bug**: `anchorTime()` uses `block.number`, which is unreliable on Optimism.
* **Impact**: 90-day holding period becomes inaccurate → early or delayed redemptions.
* **Fix**: Replace with `block.timestamp`; add tests to ensure true time-based enforcement.
